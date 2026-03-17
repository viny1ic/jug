# VS Code Extension Plan for Running Claude Code SKILL.md via Non‑Agentic Copilot‑Like Prompts

## Executive summary

You can build a VS Code extension that executes Claude Code `SKILL.md` files end-to-end (read context → propose edits → apply changes → run tests/build → report) **without using Copilot CLI** by implementing your own deterministic “agent loop” inside the extension and calling Copilot-backed models through VS Code’s **Language Model API** (`vscode.lm.selectChatModels` + `model.sendRequest`). This is explicitly supported: VS Code documents how extensions can select Copilot models, send requests, handle user-consent, and process streaming responses. citeturn8view0turn8view3turn3view0

Because many organizations disable “agent mode” (`chat.agent.enabled`) centrally, your design should **not depend on VS Code’s built-in agent loop**. Enterprise docs confirm that when agents are disabled, developers can still use **Ask/Edit** for chat and file edits, but autonomous task execution is unavailable. Your extension sidesteps that limitation by keeping the autonomy in your extension’s state machine while using the LM API as a non-agentic reasoning engine that emits strict JSON actions. citeturn2view0

The critical engineering work is: (1) robust skill loading + rendering (Claude substitutions + `!`command preprocessing), (2) a strict JSON action protocol, (3) safe edit application using `WorkspaceEdit` + `workspace.applyEdit`, (4) controlled command execution via VS Code Task/Terminal APIs with explicit user confirmations, and (5) enterprise‑friendly deployment (workspace trust gating, allowlisted extension distribution, telemetry opt-outs). citeturn7view0turn5view3turn14view1turn14view2turn15view2turn9view1turn13view0

## Goals and scope

### Goals

- Execute a Claude Code skill (`.claude/skills/<skill>/SKILL.md`) in a repeatable loop:
  - gather context (read files, optional preprocessing)
  - plan steps
  - propose code edits (as unified diff or structured edits)
  - apply edits to workspace
  - run a test/build command
  - produce a final summary report (and persist session artifacts). citeturn7view0turn15view2turn14view2turn14view1
- Use “Copilot-like prompt calls” in a **non-agentic** way: call the LM API and require **JSON-only** outputs; do not rely on agent mode or Copilot CLI. citeturn8view0turn2view0
- Be enterprise-safe by design: workspace trust gating, explicit approvals for risky operations, sensitive-file protection patterns, and opt-in telemetry. citeturn5view3turn9view3turn13view1

### Explicit non-goals for v1

- Full parity with Claude Code subagents (`context: fork`) and hooks semantics; you can approximate fork by isolating history in your own session store, but you won’t replicate Claude’s exact runtime. citeturn7view0turn2view0
- Implementing VS Code “agent tools” / tool-calling integration. Enterprise policy can disable extension-contributed chat tools; we’ll keep your runner independent and triggered via commands. citeturn2view0turn0search4

### Key assumptions (state explicitly in docs/README)

- Users have Copilot access in VS Code and will grant the consent dialog when the extension requests model access; VS Code notes Copilot models require consent and that `selectChatModels` should be called from a user-initiated action (command). citeturn8view0turn0search1
- Workspace is trusted for operations that execute code (tasks/terminal). If untrusted, extension runs in restricted mode and must disable trust-sensitive features; VS Code provides static manifest declarations and runtime APIs for this. citeturn5view3turn0search2

## Required VS Code APIs and enterprise constraints

### Language Model API access (core dependency)

- Select models via `vscode.lm.selectChatModels({ vendor: 'copilot', family: 'gpt-4o' })` and handle “no models available” (empty array). citeturn8view0turn3view2
- Call the model via `model.sendRequest(messages, {}, token)` and consume the streaming response (`for await (const fragment of chatResponse.text)`). citeturn3view0turn8view0
- Handle failures using `LanguageModelError` and anticipate user-consent and quota errors. citeturn8view0
- Rate limiting: VS Code cautions that extensions should be aware of rate limits and should not use the LM API for integration tests. citeturn1view0

### Workspace trust and restricted mode (security gate)

- Add `capabilities.untrustedWorkspaces` to `package.json` with `supported: 'limited'` (recommended) so the extension can run “read-only / list skills” in restricted mode but disables command execution and writes until trusted. citeturn5view3
- Gate at runtime using `vscode.workspace.isTrusted` and `onDidGrantWorkspaceTrust`. citeturn5view3

### File I/O and edits in workspace

- Reading files: use `workspace.fs`-based reads (or `TextDocument` APIs), and treat file access as workspace-scoped. The API reference explicitly distinguishes `workspace.fs` from `workspace.applyEdit`; prefer `applyEdit` for tracked, user-visible edits. citeturn14view0turn14view2
- Applying edits: convert model output to a `WorkspaceEdit` and call `workspace.applyEdit(edit)`. citeturn14view1turn14view2
- Use an `OutputChannel` for human-readable logs; VS Code provides `OutputChannel` as a read-only textual container for extension output. citeturn14view5

### Command execution

- Use VS Code Tasks for deterministic build/test execution: `tasks.executeTask(task)` returns a `TaskExecution`, and task lifecycle events exist (start/end and process start/end). citeturn15view2turn15view1
- Alternatively, use an integrated terminal (`window.createTerminal`) for interactive visibility, but Tasks are better for capturing lifecycle and exit codes cleanly in enterprise environments. citeturn14view3turn15view2

### Enterprise policies and Copilot availability

- Agent mode can be centrally disabled via policy (`ChatAgentMode`), corresponding to `chat.agent.enabled`; even then, developers can still use Ask/Edit. This supports your approach: implement your own loop instead of relying on built-in agent mode. citeturn2view0
- Enterprise can control tool approvals and terminal auto-approval for agent tools. Even though your extension is not “agent tools,” these settings reflect enterprise expectations: you should implement your own confirmation prompts for risky operations (file edits, terminal commands). citeturn2view0turn9view3
- Model availability is governed by plan, client, and organization restrictions; your extension must degrade gracefully if certain families/models are disabled. citeturn11view0turn10view0

## Architecture and file conventions

### File/folder conventions and manifest

Re-use Claude Code’s published skill conventions:

- `.claude/skills/<skill-name>/SKILL.md` is the canonical entry point; `name` defaults to directory name; substitutions, `allowed-tools`, and `!`command preprocessing are defined in the Claude skill doc. citeturn7view0
- Optional runner sidecar: `.claude/skills/<skill-name>/.copi-runner.json` (extension-specific settings like default test command, denylist patterns, max bytes). (Assumption: this is your extension’s convention, not a Claude standard.)

Recommended layout:

| Path | Purpose | Required |
|---|---|---|
| `.claude/skills/<skill>/SKILL.md` | Skill definition (frontmatter + instructions) | Yes citeturn7view0 |
| `.claude/skills/<skill>/.copi-runner.json` | Runner knobs (test command, safety rules) | No |
| `.copi-runner/sessions/<id>.json` | Session trace (plan/actions/diffs/results) | No |

### Components (modules) and responsibilities

**SkillLoader**
- Discover skills under `.claude/skills/**/SKILL.md` in the opened workspace (use `workspace.findFiles`).
- Parse YAML frontmatter + markdown body (recommend a small YAML parser dependency).
- Build a “skills index” with name/description for QuickPick.

**SkillRenderer**
- Implements Claude substitutions exactly:
  - `$ARGUMENTS`, `$ARGUMENTS[N]`, `$N`
  - `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`
  - “Append `ARGUMENTS:` if `$ARGUMENTS` not present.” citeturn7view0

**Preprocessor**
- Implements `!`command preprocessing:
  - Execute commands before sending prompt; replace placeholder with output.
  - Claude docs emphasize this is preprocessing (model sees only rendered output). citeturn7view0
- Security: require explicit confirmation (and disable in untrusted workspaces).

**Prompt modules**
- Planner → Executor → Reviewer → Final Reporter templates, each producing **JSON-only** responses.

**ToolRunner**
- Implements runner tools corresponding to your JSON protocol:
  - `read_file`: `workspace.fs.readFile`
  - `apply_patch`: parse unified diff → `WorkspaceEdit` → `workspace.applyEdit`
  - `run_cmd`: execute a VS Code Task; capture exit code via task process end events. citeturn14view0turn14view2turn15view2

**SessionStore**
- Persist:
  - prompts sent (redacted)
  - tool results
  - diffs/edits applied
  - command outputs
  - final report
- Storage options:
  - workspace-local `.copi-runner/sessions` for shareable traces
  - extension `globalStorageUri` / `storageUri` for private traces (API reference notes storage URIs exist and can be used with `workspace.fs`). citeturn14view0

**UI and Commands**
- Command palette commands:
  - `copiRunner.runSkill`
  - `copiRunner.listSkills`
  - `copiRunner.openLastSessionLog`
  - `copiRunner.cancelRun`
- Output channel: `Copi Runner` for step-by-step log.
- Optional: Chat participant integration (later). VS Code documents that for advanced scenarios you can implement tool calling yourself and suggests using prompt-tsx if desired. citeturn13view2turn1view0

## Prompt flow, JSON protocol, templates, and runtime flow

### Message protocol (JSON action schema)

All messages must be **one JSON object** (no markdown). Types:

- `plan`
- `tool_call`
- `apply_patch`
- `run_cmd`
- `review`
- `final`
- `fail` (recommended for explicit aborts)

Suggested minimal schema:

```json
{
  "type": "tool_call",
  "session_id": "string",
  "tool": "read_file | apply_patch | run_cmd",
  "args": { }
}
```

Additional fields by type:

- `plan`: `steps[]` with `id`, `goal`, `success_criteria[]`
- `apply_patch`: `diff` (unified diff), `files_touched[]`
- `run_cmd`: `command`, `cwd`, `timeout_sec`
- `review`: `verdict` (`pass|retry|fail`), `issues[]`
- `final`: `summary`, `diff_excerpt`, `command_result_excerpt`

### Mermaid runtime flowchart

```mermaid
flowchart TD
  A[User command: Run Skill] --> B[Workspace trust check + policy load]
  B --> C[Load SKILL.md + sidecar]
  C --> D[Render substitutions + !`cmd` preprocessing]
  D --> E[Select LM model via vscode.lm.selectChatModels]
  E --> F[Planner prompt -> plan JSON]
  F --> G[Executor prompt -> tool_call JSON]
  G --> H[ToolRunner executes tool; append tool result]
  H --> I{Next action?}
  I -->|apply_patch| J[Apply patch via WorkspaceEdit + workspace.applyEdit]
  I -->|run_cmd| K[Run task; capture exit code/output]
  I -->|review| L[Reviewer prompt -> verdict JSON]
  L -->|retry (<=N)| G
  L -->|pass| M[Final prompt -> final JSON]
  M --> N[Persist session + show summary + open diffs]
```

### Prompt templates (JSON-only)

These templates should be built as `LanguageModelChatMessage.User(...)` messages. VS Code docs show using “persona/rules” as the first message and the actionable task as the second, and that `LanguageModelChatMessage.User/Assistant` exist. citeturn8view2turn8view0turn3view0

**Planner**
```text
Return ONE valid JSON object only. No markdown. No extra text.

You are planning execution of a Claude Code SKILL.md using a VS Code extension runner.
You have NO direct tool access. The runner will execute file reads, edits, and commands as instructed.

Rendered skill instructions:
<<<
{RENDERED_SKILL}
>>>

Output schema:
{
  "type": "plan",
  "session_id": "{SESSION_ID}",
  "steps": [
    {
      "id": "step-1",
      "goal": "...",
      "success_criteria": ["..."],
      "allowed_runner_tools": ["read_file","apply_patch","run_cmd"]
    }
  ]
}
```

**Executor (choose next action)**
```text
Return ONE valid JSON object only.

Context:
- session_id: {SESSION_ID}
- step: {CURRENT_STEP}
- recent_tool_results: {TOOL_RESULTS_SUMMARY}
- current_workspace_diff: {DIFF_SUMMARY}

Choose exactly one action:
- tool_call(read_file)
- apply_patch(unified diff)
- run_cmd
- review
- fail

Output one of the schemas:
{...}
```

**Reviewer**
```text
Return ONE valid JSON object only.

Evaluate whether the changes satisfy the skill instructions and step success criteria.

Diff:
<<<DIFF
{UNIFIED_DIFF}
DIFF>>>

Command result:
<<<CMD
exit_code={EXIT}
stdout/stderr:
{OUTPUT}
CMD>>>

Schema:
{
  "type": "review",
  "session_id": "{SESSION_ID}",
  "verdict": "pass|retry|fail",
  "issues": [{"severity":"high|med|low","detail":"..."}],
  "next_action": "short"
}
```

**Final**
```text
Return ONE valid JSON object only.

Write a concise final report for the user:
- what changed
- what command ran and whether it passed
- any follow-ups

Schema:
{
  "type": "final",
  "session_id": "{SESSION_ID}",
  "summary": "string",
  "diff_excerpt": "string",
  "command_result_excerpt": "string"
}
```

## Error handling, retries, and security model

### Error handling and retries

- Model selection:
  - Handle empty model array and show a user-friendly message (“Copilot Chat extension not installed/enabled” is explicitly shown in the official sample). citeturn3view2turn8view0
- Model request failures:
  - Catch `LanguageModelError` and branch:
    - missing consent
    - quota exceeded / rate limited
    - “off topic” or policy constraints (VS Code guide mentions using `LanguageModelError` and checking `err.code`/`err.cause`). citeturn8view0
- JSON parsing failures:
  - Implement a “repair” prompt: feed the raw text back and demand JSON-only.
  - Bound it (e.g., 2 repair attempts), then fail with diagnostics.
- Task execution:
  - Use task lifecycle events (`onDidEndTaskProcess`) to capture exit code and stop the loop if nonzero unless the reviewer explicitly requests retry. citeturn15view2

### Security model (enterprise-ready)

**Workspace Trust (hard gate)**
- In `package.json`, set `capabilities.untrustedWorkspaces.supported: 'limited'` and disable write/run actions until trusted. VS Code provides both static declarations and the runtime `workspace.isTrusted` gate and UI recommendations (hide commands using `when` with `isWorkspaceTrusted`). citeturn5view3

**User confirmations**
- VS Code’s chat participant guide explicitly recommends asking for user consent before costly operations or edits that cannot be undone, as a UX/security guideline for AI features. citeturn13view2
- Implement confirmations for:
  - `!`command preprocessing execution
  - apply_patch crossing sensitive files
  - run_cmd execution.

**Allowlists and sandboxing**
- Tool allowlist derived from Claude `allowed-tools` field (even if downstream systems sometimes don’t enforce it perfectly, your runner can). Claude docs define `allowed-tools` as the tools Claude can use without asking permission when the skill is active; in your extension, treat it as the set of runner tools permitted by the skill. citeturn7view0
- Sensitive file protection:
  - Adopt the same concept as VS Code’s security guidance: protect sensitive files with glob patterns and require manual approval (the Copilot security doc gives an explicit example pattern for `.env`). citeturn9view3
- Patch path safety:
  - Reject diffs that reference paths outside workspace.
  - Only allow writes within `workspace.workspaceFolders`.

**Enterprise policy awareness**
- Agents can be disabled by policy; VS Code confirms Ask/Edit still work. Your extension should be resilient to this by not relying on agent mode at all. citeturn2view0
- Extension-contributed tools can be disabled by enterprise policy; avoid depending on “chat tool” contribution points for core behavior. citeturn2view0

## Testing strategy and Spec Kit integration

### Testing strategy

**Unit tests (fast, deterministic)**
- Skill parsing (frontmatter + body)
- Substitution renderer: `$ARGUMENTS`, `$0`, `${CLAUDE_SKILL_DIR}`, etc. citeturn7view0
- Diff parser → `WorkspaceEdit` mapping
- Policy engine (workspace trust gating; sensitive-file patterns)

**Integration tests (without calling real LMs)**
- VS Code recommends not using the LM API in integration tests due to rate limits, and mentions internal simulation approaches. Practically, you should mock your “LLM client” interface and replay canned JSON actions. citeturn1view0turn8view0

**Manual test matrix**
- Trusted vs untrusted workspace
- Copilot available vs unavailable (models list empty)
- Quota/rate limit simulation (force `LanguageModelError`)
- Patch failure and retry behavior.

### Spec Kit integration

Use Spec Kit as the development methodology for this extension: capture compatibility requirements as specs and generate implementation tasks. GitHub positions Spec Kit as a spec-driven development toolkit for AI coding workflows. citeturn4search3turn4search0

Concrete integration pattern:
- Add a `spec/` or `.speckit/` folder that contains:
  - a “compatibility constitution” (security + UX principles)
  - a “Skill Runner Spec” enumerating supported Claude frontmatter fields and runner tools
  - acceptance fixtures (sample skills + expected session traces)
- In the extension, add a dev-only command:
  - `copiRunner.dumpSessionAsSpecArtifact` → writes session JSON + a short summary markdown to a `spec/artifacts/` folder.

If you want to connect more directly, Spec Kit’s quickstart describes the workflow phases (constitution → spec → plan → tasks) and can be referenced in contributor docs. citeturn4search8turn4search0

## Telemetry, logging, session persistence, packaging, and enterprise deployment

### Logging + session persistence

- Use an `OutputChannel` for human-readable logs and progress. citeturn14view5
- Persist session traces:
  - recommended: workspace-local `.copi-runner/sessions/<id>.json` (easy to share internally)
  - optional: `globalStorageUri`/`storageUri` for private logs. citeturn14view0turn14view2

### Telemetry (opt-in and enterprise-aware)

- VS Code provides a telemetry author guide and recommends `@vscode/extension-telemetry` for safe, consistent telemetry collection, and stresses respecting user settings (`telemetry.telemetryLevel`). citeturn13view0turn13view1
- For enterprise, telemetry can be centrally managed via policy (`TelemetryLevel`). Your extension should either:
  - use the official telemetry module (preferred), or
  - fully respect `isTelemetryEnabled` / settings and provide an extension-level opt-in. citeturn13view0turn13view1

### Packaging and distribution

- Package as `.vsix` using `vsce package`; users/admins can install via `code --install-extension <vsix>`. citeturn9view0
- Enterprise controls:
  - allowed extensions allowlist (`extensions.allowed`) can restrict installation; plan for an allowlisted publisher/extension ID. citeturn9view1
  - private marketplace can host internal extensions and curated public ones; VS Code enterprise docs outline self-hosting and rollout. citeturn9view1

### Milestones and effort estimate

Assuming one experienced extension engineer; estimates include tests and documentation.

| Milestone | Deliverables | Est. hours |
|---|---|---:|
| Prototype loop | Command `Run Skill`, selects model, JSON-only planner/executor/reviewer/final, OutputChannel logs | 12–18 |
| Skill compatibility core | `.claude/skills` discovery, frontmatter parse, substitutions, argument UI, `.copi-runner.json` parsing | 10–16 |
| Safe patch application | Unified diff → WorkspaceEdit pipeline; sensitive file denylist; preview/confirm UI | 14–24 |
| Command execution | Task-based runner + exit code capture; confirmation prompts; cancel support | 10–18 |
| Enterprise hardening | Workspace Trust support (`capabilities.untrustedWorkspaces` + gating), policy-aware behavior, documentation | 10–16 citeturn5view3turn2view0turn9view1 |
| Testing + spec-kit plumbing | Unit tests, fixture replay, spec artifacts, CI pipeline | 12–20 citeturn1view0turn4search3 |
| Packaging + rollout | VSIX packaging, installation docs, internal deployment guidance | 6–10 citeturn9view0turn9view1 |

Total (MVP usable): ~**56–92 hours**.

## Minimal code scaffold and a small TypeScript snippet (≤120 LOC)

### Scaffold outline (files and key functions)

- `src/extension.ts`
  - activate: register commands
  - create OutputChannel
- `src/skills/discover.ts`
  - `findSkills(): Promise<SkillRef[]>`
- `src/skills/parse.ts`
  - `parseSkill(uri): SkillDefinition`
- `src/skills/render.ts`
  - `renderSubstitutions(skill, args, sessionId): string`
  - `preprocessBang(rendered): Promise<string>` (gated + confirmed)
- `src/lm/client.ts`
  - `selectModel(selector): Promise<LanguageModelChat>`
  - `requestJson(messages): Promise<any>`
- `src/runner/loop.ts`
  - `runSkill(def, args): Promise<SessionResult>`
  - state machine implementing plan/tool_call/apply_patch/run_cmd/review/final
- `src/runner/tools.ts`
  - `readFile(path)`
  - `applyUnifiedDiff(diff)` → `WorkspaceEdit`
  - `runTask(command)` → exitCode/output
- `src/store/sessionStore.ts`
  - `writeSession(session): Promise<void>`

### TypeScript snippet (≤120 LOC): call LM API and parse JSON

This code follows the official model-selection + sendRequest pattern and captures a streaming response, similar to the VS Code samples and the LM API guide. citeturn8view0turn3view0turn3view2

```ts
import * as vscode from 'vscode';

export async function requestJsonFromCopilot(prompt: string): Promise<any> {
  // Must be called from a user-initiated command; Copilot models require consent.
  // (VS Code will show an auth/consent dialog if needed.)
  const [model] = await vscode.lm.selectChatModels({ vendor: 'copilot', family: 'gpt-4o' });
  if (!model) {
    throw new Error('No Copilot model available. Ensure Copilot Chat is installed and enabled.');
  }

  const messages = [
    vscode.LanguageModelChatMessage.User(
      'Return ONE valid JSON object only. No markdown. No extra text.'
    ),
    vscode.LanguageModelChatMessage.User(prompt),
  ];

  const tokenSource = new vscode.CancellationTokenSource();
  const resp = await model.sendRequest(messages, {}, tokenSource.token);

  let text = '';
  for await (const fragment of resp.text) {
    text += fragment;
  }

  // Robust JSON extraction (basic)
  const trimmed = text.trim().replace(/^```json\s*|```$/g, '');
  try {
    return JSON.parse(trimmed);
  } catch {
    const a = trimmed.indexOf('{');
    const b = trimmed.lastIndexOf('}');
    if (a >= 0 && b > a) return JSON.parse(trimmed.slice(a, b + 1));
    throw new Error(`Model did not return JSON. Raw output: ${trimmed}`);
  }
}
```

This uses:
- `vscode.lm.selectChatModels({ vendor: 'copilot', family: 'gpt-4o' })` and handles “no models” cases citeturn8view0turn3view2  
- `model.sendRequest(...)` and streaming `resp.text` citeturn3view0turn8view0  
- A JSON-only instruction message, consistent with LM API examples that instruct “respond just with code / do not use markdown.” citeturn8view0turn3view0
