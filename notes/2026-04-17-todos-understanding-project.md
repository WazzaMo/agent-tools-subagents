# Todos: information needed to understand this project fully

These items bridge `docs/plan-mcp-server-acp-subagents.md` and a complete picture of how this repository implements (or will implement) that plan.

# Protocol and ecosystem context

- [ ] Read the ACP architecture overview and protocol overview (initialization, sessions, prompt turns, streaming, bidirectional requests).

  - Which ACP version or capability flags will you pin at `initialize`?

  - Which client-to-agent methods must the MCP server implement on day one versus allowlisted later?

- [ ] Skim the documentation index at https://agentclientprotocol.com/llms.txt for version changes and RFDs that could affect design.

  - Which RFDs are in scope for the first milestone versus explicitly deferred?

- [ ] Read the MCP-over-ACP RFD enough to know when Option B applies versus the baseline Option A in the plan.

  - Under what product conditions would you revisit MCP-over-ACP instead of staying on stdio MCP plus subprocess ACP?

# Product and scope decisions

- [x] Confirm which architectural option the repo targets first (MCP server as ACP client versus MCP-over-ACP) and whether that matches the written plan.

  - MCP server as ACP client.

- [ ] Resolve or locate recorded answers to the plan’s open decisions: synchronous MCP tools only versus out-of-band progress; shared workspace roots versus sandboxed copies; who answers sub-agent permission prompts (policy, queue, deny).

  - Async and out-of-band progress is more productive. Share the workspace root; the PWD where the primary agent starts the MCP is the work sandbox.

  - A permission markdown file should be created called the `permission-log.md`

  - When sub-agent asks permission, check the permission log, if not present, the primary agent asks the user and adds it to the log and then answers the sub-agent over the MCP channel.

  - Where should `permission-log.md` live (workspace root, global config path, or elsewhere), and should it be gitignored?

  - What exact fields or markdown structure are recorded for approve versus deny, and for repeated prompts?

  - How does “primary agent asks the user” work on each target host (Cursor, Claude, Codex, Gemini) in practice?

  - Which single mechanism will deliver out-of-band progress to the host (for example MCP logging, a polling tool, resources, or something else)?

- [ ] Identify the intended MCP host(s) (for example Cursor, Claude Desktop) and transport (stdio versus HTTP) for the shipped server.

  - The MCP host will be the agent where the MCP is installed and running and could be any of cursor, claude, codex and gemini.

  - The transport should be stdio.

  - The ACP agent that is invoked over ACP by the MCP server should be the same client as the main host because we know it is available.

  - How is “the same client as the main host” discovered (absolute path, env var, host-specific launcher, or user configuration)?

  - What happens when the user runs the MCP server from a host where that client binary is not discoverable?

  - If lifecycle guidance relies on a Cursor Skill, what is the required workflow on hosts that do not load Skills?

# Implementation reality in the codebase

- [ ] Map the chosen language runtime and ACP client or SDK to actual dependencies and entrypoints in this repo.

  - Use TypeScript to implement the MCP server and ACP client.

  - Which ACP or JSON-RPC client libraries are chosen, and what are their versions?

  - What are the concrete entrypoints (for example `package.json` scripts, main module path, and how the MCP server is started)?

- [ ] List exposed MCP tools, their JSON schemas, and how each maps to ACP methods (`session/new`, `session/prompt`, `session/update`, cancellation, and related lifecycle).

  - Session lifecycle is needed: new, prompt (in), update (out), status check, alignment check, cancellation of task, terminate session, terminate sub-agent process.

  - What are the exact MCP tool names as seen by the host?

  - What does “alignment check” mean in this design (capabilities, paths, protocol version, or something else)?

  - For each tool, which ACP request or notification type implements it, and what is the success and error shape returned to MCP?

- [ ] Document subprocess lifecycle: spawn and teardown, concurrency limits, correlation between MCP calls and ACP sessions, and thread-safety assumptions for the JSON-RPC client.

  - Concurrency concern is that two agents may attempt to modify the same file concurrently; this can be mitigated by orchestration.

  - Assume concurrency should support a range between 2 sub-agents and 4 sub-agents.

  - Sub-agents must be given different roles to mitigate concurrency issues.

  - How are MCP tool call identifiers correlated to ACP session identifiers and subprocess handles?

    - An in-memory map will be required to maintain the association.
    - As new associations are formed, this can be added to a `mapping.log` file in the subagents directory. 

  - What is the JSON-RPC concurrency model (one reader per pipe, per-session queues, or library-provided guarantees)?

  - What is the policy when two sub-agents touch the same paths (serialize writes, advisory locks, task ownership, or accept conflicts)?

- [ ] Locate the sub-agent registry or equivalent: command lines, environment variables, capability flags, maximum concurrency, and how an agent profile is chosen from MCP tool inputs.

  - In the workspace root, the `subagents` directory should exist to define the roles and number of subagents.

  - The MCP will be provided with a Skill that guides the primary agent on managing the lifecycle of subagents and how to configure them.

  - What on-disk format does `subagents/` use (folders per agent, YAML, JSON, or mixed)?

  - How are command line, environment, capability flags, and max concurrency expressed per profile in that layout?

  - Is the Skill strictly optional documentation, or required for correct operation, and how is that stated for non-Cursor users?

# Operations, safety, and testing

- [ ] Read or author policy for filesystem and network scope, rate limits, secret handling, and logging redaction rules for JSON-RPC traffic.

  - What network egress is allowed for sub-agents by default?

  - What must never appear in logs or MCP error payloads (tokens, keys, raw prompts)?

  - How does `permission-log.md` interact with secret handling (can it leak sensitive tool names or arguments)?

- [ ] Align the plan’s testing matrix (unit, integration, concurrency, failure, security) with what exists in CI and local test commands.

  - Which layers are covered by automated tests today versus still only planned?

  - What is the minimal integration test that proves one subprocess, one session, and one deterministic stop reason?

- [ ] Confirm how streaming and permission UX are surfaced to MCP callers (aggregated responses, logging channel, SSE, or other).

  - How do long-running delegations avoid head-of-line blocking on the MCP tool response?

  - How does the host observe partial progress without breaking clients that only handle synchronous tool results?

# Meta

- [ ] Reconcile the plan with other top-level docs (README, AGENTS.md, ADRs) so one narrative describes current state versus future phases.

  - Which document is canonical for “current implementation status” versus “roadmap”?

  - What is the first ADR or README section that should capture decisions currently only in this note?
