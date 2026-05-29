# manada — evaluation scenarios

**GREEN check:** dispatch ONE subagent with ONLY this skill loaded. For each scenario it must (a) route to the correct reference file and (b) give the correct answer. Grade findability + routing.

## Scenarios

1. **Scope for one project** — *"I want an agent available only when I work in a specific project. Where does the file go, and what makes it project-only?"*
   - Expect: `<project>/.claude/agents/<name>.md`; discovered by walking up from cwd; not visible in other projects. → `scopes.md`

2. **Plugin agent + MCP/hooks/permissionMode** — *"My agent ships in a plugin but its `mcpServers` / `permissionMode` are ignored. Why, and how do I get them?"*
   - Expect: plugin agents drop `hooks`/`mcpServers`/`permissionMode` for security (third-party provenance) — NOT a capability ceiling; the same agent gets them in project/user scope or via the SDK. → `scopes.md`

3. **Persistent memory** — *"Can a subagent remember things across sessions?"*
   - Expect: yes — `memory: user|project|local`; `user` → `~/.claude/agent-memory/<name>/`; omit = ephemeral. → `personalization.md`

4. **Fan-out vs overhead** — *"I have 8 independent items to validate. One agent or many?"*
   - Expect: fan-out — N agents in parallel (independent → parallelism + context isolation); overhead for sequential work. → `dispatching.md`

5. **Subscription, no API key** — *"Run an agent on a timer using my Max plan, without an API key."*
   - Expect: SDK + `claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN`; unset `ANTHROPIC_API_KEY` (it wins and bills API). → `harness-vs-sdk.md`

6. **Agent not appearing** — *"I added a file to `agents/` but Claude doesn't see it."*
   - Expect: filesystem agents load at startup; restart the session (or reinstall/update the plugin). → `creating-deploying.md`

7. **Where does the agent definition live vs the slug dir** — *"Do agents live under `~/.claude/projects/<slug>/`?"*
   - Expect: no — that dir holds session/subagent transcripts (data); agent definitions are config under `.claude/agents/`, `~/.claude/agents/`, or a plugin. → `scopes.md`
