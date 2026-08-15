A2A Protocol — Deep Dive
A technical walkthrough of the Agent-to-Agent (A2A) protocol: what it is, how a
server and client actually talk to each other on the wire, and the difference
between an Agent Card and an Agent Skill. Grounded in two real, verified
sources rather than guesswork:

The official reference implementation:
a2a-samples/samples/python/agents/langgraph
— real server (__main__.py, agent_executor.py) and real client
(test_client.py) code, fetched and read directly for this doc.
Our own agents-for-demo/subagent_a2a — a working Strands-based A2A
server, deployed and tested end-to-end against a real client this session.
Important version note up front: the public spec site (a2a-protocol.org)
currently documents a newer draft revision — fields like TASK_STATE_SUBMITTED,
ROLE_USER, a tenant field, and a POST /messages REST binding. That is
not what today's Python SDK (a2a-sdk, currently 0.3.x) actually implements.
Both the LangGraph sample and our own deployment use the older, stable wire
format: lowercase role: "user", lowercase kind: "text"/"message",
TaskState.working (not TASK_STATE_WORKING), no tenant field. This doc
describes what's actually running in a2a-sdk 0.3.x — the version you'll get
if you pip install a2a-sdk today — not the newer draft. Keep that in mind if
you cross-reference the spec site and see different field names.

1. What A2A actually is
A2A is a protocol — not a framework — for one AI agent to call another AI
agent as a peer, the same conceptual role MCP plays for an agent calling a
tool. The three things it standardizes:

Discovery: how a client learns what an agent can do and how to reach
it, without a human hand-writing integration code — it fetches a JSON
document (the Agent Card).
Invocation: a standard message format (JSON-RPC 2.0) for sending a
request and getting a response, including streaming and long-running
multi-turn tasks.
Task lifecycle: a state machine (submitted → working →
completed/input-required/failed/...) so a client can track a
long-running agent job, not just a single request/response.
It deliberately does not standardize what's inside an agent, what model it
uses, or how it reasons — only the boundary between agents.

2. Agent Card vs. Agent Skill — the actual difference
This is the most commonly confused pair of terms, so concretely:

Agent Card	Agent Skill
What it describes	The agent as a whole — its identity, where to reach it, what transport/auth it needs, whether it streams	One specific capability the agent offers
Cardinality	One per agent	Zero or more, listed inside the card's skills array
Analogy	A restaurant's storefront sign + business card	Individual menu items
The Agent Card, from the real sample (__main__.py):

agent_card = AgentCard(
    name='Currency Agent',
    description='Helps with exchange rates for currencies',
    url=f'http://{host}:{port}/',
    version='1.0.0',
    default_input_modes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
    default_output_modes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
    capabilities=capabilities,   # AgentCapabilities(streaming=True, push_notifications=True)
    skills=[skill],              # the AgentSkill list — see below
)
An Agent Skill, also from the real sample:

skill = AgentSkill(
    id='convert_currency',
    name='Currency Exchange Rates Tool',
    description='Helps with exchange values between various currencies',
    tags=['currency conversion', 'currency exchange'],
    examples=['What is exchange rate between USD and GBP?'],
)
So the card says "I am the Currency Agent, reachable at this URL, I support
streaming" and the skill inside it says "specifically, I can convert
currencies — here's an example prompt and some tags to categorize me by."
A more capable agent would have one card with many skills in its skills
array — e.g. our own gitlab_pipeline_agent's card has 74 skills, one per
GitLab MCP tool it wraps (list_pipelines, get_pipeline_job_output, etc.),
confirmed by actually calling GetAgentCard against our deployed runtime and
counting the array.

Where skills come from in practice depends entirely on the server
framework:

LangGraph sample: hand-written, one AgentSkill(...) object per
capability, listed explicitly.
Our Strands-based server (subagent_a2a/agent.py): not hand-written
at all. A2AServer auto-derives one skill per tool on the wrapped Agent
object — we never construct an AgentSkill ourselves; Strands converts
each of our 74 filtered MCP tools into one. A2AServer does accept an
explicit skills=[...] override if you want curated skill entries instead
of one-per-tool.
3. The A2A server side
3.1 The pieces, real code
From __main__.py in the LangGraph sample:

request_handler = DefaultRequestHandler(
    agent_executor=CurrencyAgentExecutor(),
    task_store=InMemoryTaskStore(),
    push_config_store=push_config_store,
    push_sender=push_sender,
)
server = A2AStarletteApplication(
    agent_card=agent_card, http_handler=request_handler
)
uvicorn.run(server.build(), host=host, port=port)
Three layers, bottom to top:

AgentExecutor (you write this) — the actual "what does the agent do
when called" logic.
DefaultRequestHandler (library-provided) — parses incoming JSON-RPC,
manages the task store (tracks task state across turns), calls your
executor, and turns its output into proper A2A Task/Message/Artifact
responses.
A2AStarletteApplication (library-provided) — the actual ASGI/HTTP
app: serves the agent card at the well-known path, routes JSON-RPC POSTs
to the request handler, runs under uvicorn like any Starlette app.
3.2 Writing an AgentExecutor by hand (LangGraph sample)
class CurrencyAgentExecutor(AgentExecutor):
    def __init__(self):
        self.agent = CurrencyAgent()

    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        query = context.get_user_input()
        task = context.current_task
        if not task:
            task = new_task(context.message)
            await event_queue.enqueue_event(task)
        updater = TaskUpdater(event_queue, task.id, task.context_id)

        async for item in self.agent.stream(query, task.context_id):
            if not item['is_task_complete'] and not item['require_user_input']:
                await updater.update_status(TaskState.working, new_agent_text_message(item['content'], ...))
            elif item['require_user_input']:
                await updater.update_status(TaskState.input_required, ..., final=True)
                break
            else:
                await updater.add_artifact([Part(root=TextPart(text=item['content']))], name='conversion_result')
                await updater.complete()
                break

    async def cancel(self, context: RequestContext, event_queue: EventQueue) -> None:
        raise ServerError(error=UnsupportedOperationError())
Notice: this executor streams intermediate working status updates as the
underlying LangGraph agent reasons, then either asks for more input
(input_required) or finishes with an artifact + complete(). This is the
manual, framework-level way to implement A2A — you're directly managing task
state transitions.

3.3 The Strands shortcut — you don't write an executor
In subagent_a2a/agent.py, there is no AgentExecutor subclass anywhere.
Strands' A2AServer supplies its own internal executor
(StrandsA2AExecutor) that wraps a Strands Agent automatically — you give
it an agent (or an agent_factory), and it handles the execute()/cancel()
translation, task lifecycle, and streaming for you:

def _build_agent(context_id: str) -> Agent:
    return Agent(
        name="gitlab_pipeline_agent",
        description="Answers questions about GitLab CI/CD pipelines, jobs, and build status.",
        model=model,
        system_prompt=SYSTEM_PROMPT,
        tools=_tools,
        callback_handler=None,
    )

a2a_server = A2AServer(
    agent_factory=_build_agent,
    http_url=RUNTIME_URL,
    serve_at_root=True,
    enable_a2a_compliant_streaming=True,
)
app = FastAPI()
app.mount("/", a2a_server.to_fastapi_app())
Why agent_factory instead of a single agent=: Strands documents
passing a static agent= instance as deprecated — "a single agent serializes
all requests" and mixes different callers' conversation state together (we
hit this directly and fixed it). agent_factory is Callable[[context_id], Agent] — the server calls it once per new A2A context_id and caches the
result, giving each conversation its own isolated Agent instance. It's also
called once at server startup with a sentinel context id purely to read
.name/.description off the returned Agent for the agent card — that's
literally where our card's name/description come from (see the earlier
"where's the agent card" discussion — there is no separate AgentCard(...)
constructor call in our code at all).

3.4 Task state machine (the actual enum values, verified from real code)
submitted → working → completed
                     → input_required (needs another message from the client, same task_id)
                     → failed
                     → canceled
TaskState.working, TaskState.input_required, etc. — Python enum members,
lowercase on the wire ("working", "input-required"), confirmed both
from the LangGraph sample's literal code and from our own working responses.

4. The A2A client side
4.1 The "textbook" two-step: discover, then invoke
From the real test_client.py:

resolver = A2ACardResolver(httpx_client=httpx_client, base_url='http://localhost:10000')
public_card = await resolver.get_agent_card()   # GET {base_url}/.well-known/agent-card.json

client = A2AClient(httpx_client=httpx_client, agent_card=public_card)

request = SendMessageRequest(
    id=str(uuid4()),
    params=MessageSendParams(message={
        'role': 'user',
        'parts': [{'kind': 'text', 'text': 'how much is 10 USD in INR?'}],
        'message_id': uuid4().hex,
    }),
)
response = await client.send_message(request)
Two real, separate HTTP calls happen here:

GET {base_url}/.well-known/agent-card.json — discovery.
POST {base_url}/ with a JSON-RPC body {"jsonrpc":"2.0","id":...,"method":"message/send","params":{...}} — invocation.
4.2 Multi-turn conversations — task_id + context_id
When an agent responds input_required, the client continues the same
task by echoing back the task_id and context_id it got in the first
response, in the next message:

task_id = response.root.result.id
context_id = response.root.result.context_id

second_request = SendMessageRequest(
    id=str(uuid4()),
    params=MessageSendParams(message={
        'role': 'user',
        'parts': [{'kind': 'text', 'text': 'CAD'}],
        'message_id': uuid4().hex,
        'task_id': task_id,
        'context_id': context_id,
    }),
)
Without these two IDs, the server would treat it as a brand-new,
unrelated conversation.

4.3 Streaming
streaming_request = SendStreamingMessageRequest(id=str(uuid4()), params=MessageSendParams(**payload))
async for chunk in client.send_message_streaming(streaming_request):
    print(chunk.model_dump(mode='json', exclude_none=True))
Same method (message/stream instead of message/send), returns an async
generator of incremental TaskStatusUpdateEvent/TaskArtifactUpdateEvent
chunks instead of one final response.

4.4 Why we couldn't use this textbook client against our own deployment
A2ACardResolver/A2AClient (and Strands' equivalent wrapper,
strands_tools.a2a_client.A2AClientToolProvider) assume the target is a
plain, reachable HTTP server — they do a bare GET for discovery and a
bare POST for invocation, no request signing.

Our A2A subagent is hosted on AWS Bedrock AgentCore Runtime, which
requires every request — including the discovery GET — to carry an AWS
SigV4 signature. There is no unauthenticated .well-known/agent-card.json
route reachable on AgentCore's public endpoint at all. So the textbook client
above gets a straight rejection against our deployment; it has no way to
attach AWS credentials.

What we actually built instead (orchestrator/a2a_agent_client.py,
orchestrator/sigv4_auth.py) reimplements the same two-call shape — GET
for card discovery, POST for message/send — but through raw httpx with
a custom SigV4HTTPXAuth(httpx.Auth) that signs every outgoing request. The
protocol (A2A JSON-RPC) is identical; only the transport-level
authentication differs. AgentCore additionally exposes card discovery at a
different URL shape than the plain A2A spec path:

Plain A2A spec:  GET  {base_url}/.well-known/agent-card.json                     (no auth)
AgentCore:       GET  https://bedrock-agentcore.{region}.amazonaws.com/runtimes/{arn}/invocations/.well-known/agent-card.json   (SigV4-signed)
This is also exactly why HttpAgentClient/A2AAgentClient in our
orchestrator are written generically (URL + optional httpx.Auth, no
AWS-specific code inside them) — pointed at a plain A2A server elsewhere,
they'd work with auth=None and the standard .well-known path, no code
changes needed.

4.5 taskId vs contextId — what each one actually means, and a gap in our own code
These are two different IDs with two different scopes, easy to conflate:

contextId — a whole conversation/session. One context can span many
separate tasks over time (think: a chat thread).
taskId — one bounded unit of work with its own lifecycle
(submitted → working → completed/input-required/failed/canceled). A
single context can accumulate many tasks as the conversation continues.
They're generated server-side on the first message of a new
conversation (you don't invent them as the client) and come back in the
response envelope:

{ "result": { "id": "f94e5b74-...",        // <- taskId
              "contextId": "4d1843b4-...", // <- contextId
              "status": {"state": "completed"} } }
To continue the same conversation, the client reads both out of the
previous response and includes them in the next message (as shown in
test_client.py, §4.2). Omit them, and the server treats it as a brand-new
context with no memory of anything before it.

On the server side, contextId is also the cache key for
Strands' per-conversation isolation (§3.3): each new contextId gets its own
freshly-built Agent from agent_factory, cached for reuse if the same
contextId comes back. That cache is bounded — A2AServer(..., max_contexts: int = 1000) — and evicts least-recently-used entries past that limit
("context_id=<%s> | evicted least-recently-used A2A context" in Strands'
own executor log). So a context that goes quiet long enough, or gets crowded
out by 1000 newer ones, silently loses its cached Agent and conversation
history — a fresh one gets built next time that contextId resurfaces.

The honest gap: our own A2AAgentClient doesn't use either of these.
Checked directly — taskId/contextId appear nowhere in
orchestrator/a2a_agent_client.py. Every call to gitlab_pipeline_agent
sends a message with no taskId/contextId, gets back a response, and
discards both IDs. The practical effect:

Every single call from the orchestrator to the A2A subagent starts a
brand-new context, server-side — the subagent has zero memory of any
earlier question, even earlier in the same orchestrator conversation.
This matches what the HTTP subagent does too (a fresh Agent per
invocation, §3.3 in the main repo) — so today, neither subagent carries
memory between calls. That's consistent, just not something the A2A
protocol's multi-turn machinery is actually being used for here.
If you want real multi-turn continuity with the A2A subagent (e.g. so
"and what about last week?" as a follow-up actually has context), the fix is
in A2AAgentClient: accept and thread through task_id/context_id —
store the last response's IDs (per orchestrator session) and include them on
the next call(), same shape as test_client.py's multi-turn example. Not
implemented today; flagging it since you asked.

4.6 Other protocol pieces we haven't touched
Worth knowing exist, even though nothing in our current deployment uses them:

tasks/cancel — a client can cancel an in-flight task by ID. Our
AgentExecutor equivalent (Strands' internal one) supports this via the
same mechanism the LangGraph sample shows explicitly:
async def cancel(...). The LangGraph sample's own cancel() just raises
UnsupportedOperationError() — i.e. it opts out too. Not wired up in
either the sample or our code.
Push notifications — an alternative to polling/streaming: the client
registers a webhook URL, and the server calls it when task state changes
instead of the client holding a connection open. The LangGraph sample sets
up InMemoryPushNotificationConfigStore/BasePushNotificationSender but
the actual test client never registers one. Not used anywhere in our system.
final=True on status updates — seen in the LangGraph executor
(updater.update_status(TaskState.input_required, ..., final=True)) —
marks that this status update is the last one for this streaming turn, so
the client's async for chunk in stream loop knows to stop waiting for
more chunks on this task. Only relevant to hand-written executors doing
manual streaming; Strands' auto-generated executor handles this internally.
5. The wire format, concretely (verified request/response pairs)
Discovery response (GET .../.well-known/agent-card.json or AgentCore's
signed equivalent) — real, trimmed output from our deployed subagent:

{
  "capabilities": { "streaming": true },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"],
  "description": "Answers questions about GitLab CI/CD pipelines, jobs, and build status.",
  "name": "gitlab_pipeline_agent",
  "preferredTransport": "JSONRPC",
  "protocolVersion": "0.3.0",
  "skills": [
    { "id": "list_pipelines", "name": "list_pipelines", "description": "List pipelines with filtering options", "tags": [] },
    ...
  ],
  "url": "https://bedrock-agentcore.us-east-1.amazonaws.com/runtimes/.../invocations/"
}
Invocation request (POST, JSON-RPC message/send):

{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{ "kind": "text", "text": "what pipeline tools do you have?" }],
      "messageId": "c9a1...",
      "kind": "message"
    }
  }
}
Invocation response — real output, trimmed:

{
  "id": "1",
  "jsonrpc": "2.0",
  "result": {
    "artifacts": [
      {
        "artifactId": "a5502b97-...",
        "name": "agent_response",
        "parts": [
          { "kind": "text", "text": "I have tools to list pipelines, get pipeline details, ..." }
        ]
      }
    ],
    "contextId": "4d1843b4-...",
    "history": [ { "role": "user", "parts": [...], "messageId": "...", "taskId": "..." } ],
    "id": "f94e5b74-...",
    "kind": "task",
    "status": { "state": "completed", "timestamp": "2026-08-15T07:58:25Z" }
  }
}
Key things to notice, all lowercase/lowerCamelCase, no ROLE_/TASK_STATE_
prefixes: "role": "user", "kind": "text" / "kind": "message" /
"kind": "task", "state": "completed". Text streams back in multiple
parts chunks even in the non-streaming response — our orchestrator's
A2AAgentClient.call() explicitly joins them: "".join(texts).

6. Summary: how our three pieces map onto all of this
Concept	Where it lives in our repo
Agent Card	Auto-derived by A2AServer from the Agent returned by agent_factory — no explicit AgentCard(...) in our code
Agent Skills	Auto-derived, one per MCP tool, after filtering out the one tool Claude's API rejects (get_ci_catalog_resource, top-level anyOf schema)
A2A Server	subagent_a2a/agent.py — A2AServer(agent_factory=...), mounted into a FastAPI app, served by uvicorn on port 9000
AgentExecutor	Not hand-written — Strands' internal StrandsA2AExecutor, driven by agent_factory
A2A Client (generic)	orchestrator/a2a_agent_client.py — A2AAgentClient, works against any A2A server given a URL + optional auth
SigV4 signing	orchestrator/sigv4_auth.py — the one AWS-specific piece, needed only because our server happens to be AgentCore-hosted
Card discovery call	A2AAgentClient.get_agent_card() — TTL-cached, GET to AgentCore's signed card route
Invocation call	A2AAgentClient.call() — POST JSON-RPC message/send, parses result.artifacts[].parts[].text
taskId / contextId (multi-turn)	Not used — every call is a fresh, stateless one-shot; see §4.5 for what threading them through would take
tasks/cancel, push notifications	Not implemented — neither the LangGraph sample nor our system wires these up (see §4.6)
References
Real server/client source (fetched and read for this doc):
github.com/a2aproject/a2a-samples/tree/main/samples/python/agents/langgraph
Our own implementation: agents-for-demo/subagent_a2a/agent.py,
agents-for-demo/orchestrator/{agent.py,a2a_agent_client.py,sigv4_auth.py}
a2a-sdk PyPI package (the Python library both use) — version 0.3.26 at
time of writing, pinned by strands-agents[a2a]'s own dependency spec
(a2a-sdk<0.4.0,>=0.3.0)
