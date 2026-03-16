# jug
# A Minimal Prompt-Only Claude Code Skill Runner Using GitHub Copilot CLI

## Executive summary

It is feasible to build a very small, CLI-first “skill runner” that executes a **Claude Code** `SKILL.md` end-to-end by implementing your own **agent loop** and using **GitHub Copilot CLI** only for **prompt-mode** reasoning (no Copilot agent mode/autopilot). The runner loads and renders a skill, reads target files, asks Copilot CLI (via repeated `-p/--prompt` calls) to plan and produce patches as **strict JSON actions**, applies the resulting unified diff locally, runs a validation command (tests/build), and asks Copilot CLI for a final report. This design stays robust because it does not depend on Copilot’s internal tool execution; instead, it uses a narrow, deterministic local tool surface and enforces an allowlist. citeturn3view1turn3view0turn4view1

To maximize “Claude-like” compatibility, the runner should support the Claude Code skill fields that directly affect execution semantics—especially **`allowed-tools`**, **argument substitution** (`$ARGUMENTS`, `$0`…), and **dynamic context injection** (`!`command`). Claude Code explicitly treats `!`command as **preprocessing** that runs before the model sees the prompt, which maps naturally to a local runner that executes commands and injects their outputs. citeturn4view1turn4view4

The prototype below demonstrates the smallest viable loop: **parse `SKILL.md` → plan → request file read → generate unified diff → apply diff → run tests/build → review → final report**, using Copilot CLI prompt mode with `-s` (script-friendly output) and tool-deny flags to keep the model in “prompt-only” behavior. citeturn3view1turn5view3turn5view0

## Minimal Claude SKILL.md inventory and execution stages for a lightweight runner

### Fields and semantics to support

Claude Code documents the following frontmatter fields as relevant configuration knobs for skill behavior (a lightweight runner should support at least the “execution-affecting” ones): **`name`**, **`description`**, **`argument-hint`**, **`disable-model-invocation`**, **`user-invocable`**, **`allowed-tools`**, **`model`**, **`context`** (including `fork`), **`agent`** (subagent type when forked), and **`hooks`** (skill-scoped lifecycle hooks). citeturn4view1

For a minimal runner, treat these as follows:

- **Must support (for good parity):** `name`, `description`, `allowed-tools`, and argument substitution (below). citeturn4view1turn4view4  
- **Nice-to-have:** `model` (map to Copilot `--model`), `context: fork` (approximate by dropping conversational history), and `hooks` (approximate with runner lifecycle hooks). citeturn4view1turn5view2turn5view4  
- **Mostly UI/selection hints:** `argument-hint`, `disable-model-invocation`, `user-invocable` (still useful for listing/auto-selection). citeturn4view1turn3view2

### String substitution and preprocessing stages

Claude Code skill text supports well-defined substitutions:

- `$ARGUMENTS` (and if absent, Claude Code appends `ARGUMENTS: <value>`), plus `$ARGUMENTS[N]` and `$N` shorthand (`$0`, `$1`, …). citeturn4view2turn4view4  
- `${CLAUDE_SESSION_ID}` and `${CLAUDE_SKILL_DIR}` for session correlation and stable references to skill-bundled resources. citeturn4view4  
- Dynamic injection: `!`command syntax runs shell commands **before** skill content is sent to the model; output replaces the placeholder (explicitly **preprocessing**, not a tool the model runs). citeturn4view1  

### Minimal execution stages for a prompt-only runner

A lightweight runner can mimic Claude Code’s skill execution stages as:

1. **Discovery & load**: find `.claude/skills/<name>/SKILL.md` (and optionally `.github/skills`) and load YAML frontmatter + markdown body. Claude Code also treats `.claude/commands/*.md` and `.claude/skills/<name>/SKILL.md` as equivalent slash commands, so a runner can optionally support both for compatibility. citeturn3view3turn3view2  
2. **Render**: apply `$ARGUMENTS` / `$N` / `${CLAUDE_*}` substitutions; append `ARGUMENTS:` if needed. citeturn4view2turn4view4  
3. **Preprocess injections**: execute `!`command locally and insert output into rendered prompt. citeturn4view1  
4. **Plan**: prompt Copilot CLI (programmatic prompt mode) to output a step plan in strict JSON. Copilot CLI explicitly supports programmatic prompting via `-p` and script-friendly output via `-s`. citeturn3view1turn3view0  
5. **Execute**: loop tool calls and patches; run local tools (read file, apply patch, run tests). (In this design, you deny Copilot CLI tools and implement tools yourself.) Copilot CLI supports explicit tool allow/deny patterns; denying tools is a first-class feature. citeturn5view3turn5view4turn3view1  
6. **Review & finalize**: ask Copilot CLI to review diff + test output and emit a final user-facing summary in JSON.

## Feasibility mapping for prompt-only Copilot CLI orchestration

The table below marks whether each Claude skill capability is feasible when Copilot is used only in `-p` prompt mode and the runner owns all side effects.

| Claude skill capability | Feasible? | Rationale (prompt-only Copilot CLI + local runner) |
|---|---:|---|
| Parse `SKILL.md` (frontmatter + body) | Feasible | Runner parses locally; Copilot CLI is only for reasoning. Claude and Copilot both treat `SKILL.md` as markdown with YAML frontmatter. citeturn4view1turn3view2 |
| Skill locations: `.claude/skills` and `~/.claude/skills` | Feasible | Copilot docs explicitly recognize `.claude/skills` / `~/.claude/skills` as valid skill locations (Claude-compatible), alongside `.github/skills`. citeturn3view2 |
| Argument substitution (`$ARGUMENTS`, `$0`, etc.) | Feasible | Precisely specified by Claude Code docs; implement a deterministic renderer. citeturn4view2turn4view4 |
| `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}` | Feasible | Specify and inject locally; these are explicitly documented substitutions. citeturn4view4 |
| `!`command injections (dynamic context) | Feasible | Claude Code defines this as preprocessing before model sees prompt; runner can replicate exactly by executing shell and substituting output. citeturn4view1 |
| Supporting files/resources in skill directory | Feasible | Runner can expose a `read_file` tool and feed contents back via prompts; Copilot skill docs also allow bundling scripts/resources referenced by instructions. citeturn3view2turn4view1 |
| `allowed-tools` gating | Feasible | Treat as an allowlist for runner tools; Claude defines field purpose; runner can enforce it strongly. citeturn4view1 |
| `model` override | Partial | Copilot CLI supports `--model`; mapping Claude model names to Copilot models is runner-specific and may not be 1:1. citeturn5view2turn5view4 |
| `context: fork` + `agent` | Partial | You can approximate fork by running with no prior conversation history and a separate state file; you cannot replicate Claude subagent internals exactly. citeturn4view1 |
| `hooks` in skill frontmatter | Partial | Runner can implement its own lifecycle hooks; Claude’s hook system is richer and tool-integrated. Copilot CLI also has a hooks concept, but the runner need not depend on it for a minimal prototype. citeturn4view1turn3view1 |
| Full Claude Code tool semantics | Infeasible (exactly) | You can emulate behavior (read, diff, patch, run tests) but not reproduce Claude’s internal tool runtime identically. citeturn4view1 |

## Minimal JSON action protocol for model → runner messages

The runner should require **JSON-only** output from Copilot CLI prompts. Two practical reasons:

- Copilot CLI supports `-s` for scripting output (reduces extra text) and `--output-format=json` for JSONL logs, but the simplest reliable path is still “model prints one JSON object.” citeturn3view1turn5view0turn3view0  
- Strict protocol enables deterministic retries and safe execution boundaries.

### Schema (informal but implementable)

All messages include:
- `type`: one of `plan | tool_call | apply_patch | run_cmd | review | final | fail`
- `session_id`: string

**Plan**
```json
{
  "type": "plan",
  "session_id": "…",
  "steps": [
    {"id":"step-1","goal":"…","success_criteria":["…"],"allowed_runner_tools":["read_file","apply_patch","run_cmd"]}
  ]
}
```

**Tool call**
```json
{
  "type": "tool_call",
  "session_id": "…",
  "tool": "read_file | grep | glob",
  "args": { "path": "…" },
  "rationale": "…"
}
```

**Apply patch**
```json
{
  "type": "apply_patch",
  "session_id": "…",
  "diff": "UNIFIED_DIFF",
  "files_touched": ["path1", "path2"],
  "rationale": "…"
}
```

**Run command**
```json
{
  "type": "run_cmd",
  "session_id": "…",
  "command": "pytest -q",
  "cwd": ".",
  "timeout_sec": 600,
  "rationale": "…"
}
```

**Review**
```json
{
  "type": "review",
  "session_id": "…",
  "verdict": "pass | retry | fail",
  "issues": [{"severity":"high|med|low","detail":"…"}],
  "next_action": "…"
}
```

**Final**
```json
{
  "type": "final",
  "session_id": "…",
  "summary": "…",
  "diff": "…",
  "test_output_excerpt": "…"
}
```

## File and folder conventions

The runner should be **Claude-compatible by default**:

- Primary location: `.claude/skills/<name>/SKILL.md`  
- Optional compatibility: `.github/skills/<name>/SKILL.md`  
Copilot’s docs explicitly list both `.claude/skills` and `.github/skills` as valid project skill locations, and both `~/.claude/skills` and `~/.copilot/skills` as valid personal locations. citeturn3view2

### Minimal layout table

| Path | Purpose | Required |
|---|---|---|
| `.claude/skills/<skill>/SKILL.md` | Claude skill entrypoint (frontmatter + instructions) | Yes |
| `.claude/skills/<skill>/.copi-runner.json` | Optional runner sidecar (policy + default test command) | No |
| `.copi-runner/sessions/<session_id>.json` | Runner session persistence (plan, tool outputs, diffs) | No |

## Runtime loop design

### Mermaid flowchart

```mermaid
flowchart TD
  A[Load SKILL.md + sidecar] --> B[Render substitutions + preprocess !`cmd`]
  B --> C[Copilot prompt: planner -> JSON plan]
  C --> D[Copilot prompt: executor -> tool_call(read_file)]
  D --> E[Runner reads file, stores tool result]
  E --> F[Copilot prompt: executor -> apply_patch diff]
  F --> G[Runner applies diff safely]
  G --> H[Runner runs test/build command]
  H --> I[Copilot prompt: reviewer -> verdict]
  I -->|retry| F
  I -->|pass| J[Copilot prompt: final -> summary JSON]
  J --> K[Print results, persist session, exit]
```

### Concise pseudocode

```text
load_skill(skill_name)
rendered = render_args_and_vars(skill_text, args, session_id, skill_dir)
rendered = preprocess_bang_commands(rendered)  # !`cmd` injection

plan = copilot_prompt(PLANNER(rendered, policy))

# minimal loop
tool_req = copilot_prompt(EXEC_TOOLCALL(rendered, plan))
assert tool_req.tool == "read_file"
file_text = read_file(tool_req.args.path)

patch = copilot_prompt(EXEC_PATCH(rendered, plan, file_text))
apply_unified_diff(patch.diff)   # git apply or patch

test_cmd = sidecar.test_cmd or autodetect(pytest vs compileall)
test_out = run(test_cmd)

review = copilot_prompt(REVIEW(rendered, patch.diff, test_out))
if review.verdict == "retry": go back to EXEC_PATCH (bounded)

final = copilot_prompt(FINAL(rendered, patch.diff, test_out))
print(final.summary)
```

## Prompt templates optimized for Copilot CLI prompt mode

These templates are designed to work with `copilot -p` programmatic prompting and `-s` scripting output. citeturn3view1turn3view0

### Planner prompt

```text
Return ONE valid JSON object only. No markdown. No prose outside JSON.

You are planning execution of a Claude Code SKILL.md using a local runner.
You have NO tool access. You must produce a plan only.

Rendered skill instructions:
<<<
{RENDERED_SKILL}
>>>

Policy:
- Runner will perform file edits and command execution.
- Always assume later steps will supply file contents and command outputs.

Output schema:
{
  "type": "plan",
  "session_id": "{SESSION_ID}",
  "steps": [{"id":"step-1","goal":"...","success_criteria":["..."],"allowed_runner_tools":["read_file","apply_patch","run_cmd"]}]
}
```

### Executor prompt (tool_call)

```text
Return ONE valid JSON object only.

Task: request the minimum input needed to proceed.
Choose exactly ONE tool_call.

Rendered skill instructions:
<<<
{RENDERED_SKILL}
>>>

Invocation args:
{ARGS_JSON}

Output schema:
{
  "type": "tool_call",
  "session_id": "{SESSION_ID}",
  "tool": "read_file",
  "args": {"path": "{TARGET_PATH}"},
  "rationale": "short"
}
```

### Executor prompt (apply_patch)

```text
Return ONE valid JSON object only.

You must produce a minimal unified diff that implements the skill.
Do not invent file contents beyond what is provided.

Rendered skill instructions:
<<<
{RENDERED_SKILL}
>>>

File content:
<<<FILE {TARGET_PATH}
{FILE_TEXT}
FILE>>>

Output schema:
{
  "type": "apply_patch",
  "session_id": "{SESSION_ID}",
  "diff": "UNIFIED_DIFF",
  "files_touched": ["{TARGET_PATH}"],
  "rationale": "short"
}
```

### Reviewer prompt

```text
Return ONE valid JSON object only.

Review the diff and test output. Be strict.
If tests failed, verdict must be "retry" or "fail".

Diff:
<<<DIFF
{DIFF}
DIFF>>>

Test output:
<<<TEST
{TEST_OUTPUT}
TEST>>>

Output schema:
{
  "type": "review",
  "session_id": "{SESSION_ID}",
  "verdict": "pass|retry|fail",
  "issues": [{"severity":"high|med|low","detail":"..."}],
  "next_action": "short"
}
```

### Final reporter prompt

```text
Return ONE valid JSON object only.

Write a concise final summary for the user describing:
- what changed
- whether tests/build passed
- any follow-ups

Diff:
<<<DIFF
{DIFF}
DIFF>>>

Test output:
<<<TEST
{TEST_OUTPUT}
TEST>>>

Output schema:
{
  "type": "final",
  "session_id": "{SESSION_ID}",
  "summary": "string",
  "diff": "string",
  "test_output_excerpt": "string"
}
```

## Copilot CLI command patterns and safe shell snippets

Copilot CLI supports programmatic execution using `-p/--prompt` and recommends `-s/--silent` for scripting output. citeturn3view1turn3view0  
It also supports strong tool permission gating via `--deny-tool` (and allow patterns via `--allow-tool`). citeturn5view3turn5view4

### Recommended safe patterns (prompt-only)

**Snippet A: Basic prompt-only JSON call**
```bash
copilot -p "$PROMPT" -s --no-ask-user \
  --deny-tool='shell,write,read,url,memory'
```
`--no-ask-user` disables the `ask_user` tool (prevents the agent from pausing for questions). citeturn5view2

**Snippet B: Multiline prompt via heredoc (safe quoting)**
```bash
PROMPT="$(cat <<'EOF'
Return ONE JSON object only.
{"type":"ping","session_id":"demo","msg":"hello"}
EOF
)"
copilot -p "$PROMPT" -s --no-ask-user --deny-tool='shell,write,read,url,memory'
```
Copilot’s docs show `copilot -p ... -s` as the canonical programmatic/scripting combination. citeturn3view0turn3view1

**Snippet C: Capture output robustly**
```bash
OUT="$(copilot -p "$PROMPT" -s --no-ask-user --deny-tool='shell,write,read,url,memory')"
echo "$OUT" | jq .
```

**Snippet D: Emit JSONL for logs (optional)**
```bash
copilot -p "$PROMPT" -s --output-format=json \
  --no-custom-instructions --deny-tool='shell,write,read,url,memory'
```
`--output-format=json` outputs JSONL (“one JSON object per line”) and `--no-custom-instructions` disables loading instructions from `AGENTS.md` and related files (useful for reproducibility). citeturn5view0turn5view1

## Minimal runnable prototype script (≤200 LOC)

This script is intentionally small and conservative. It:

- loads `.claude/skills/<skill>/SKILL.md`  
- renders `$ARGUMENTS` and `$0` (minimal subset)  
- runs 4 Copilot CLI prompt calls (plan → tool_call/read_file → apply_patch → review → final)  
- applies patch via `git apply` if available (fallback: `patch`)  
- runs tests/build via sidecar config or simple autodetect  
- persists a session JSON file locally  

It uses only Python standard library. It relies on Copilot CLI being installed/authenticated and uses prompt mode (`-p`) with `-s` and tool-deny flags. citeturn3view1turn5view3turn3view0

```python
#!/usr/bin/env python3
import json, os, re, shutil, subprocess, sys, time
from pathlib import Path

DENY = "shell,write,read,url,memory"

def run(cmd, **kw):
    return subprocess.run(cmd, text=True, capture_output=True, **kw)

def copilot(prompt, model=None):
    cmd = ["copilot", "-s", "--no-ask-user", f"--deny-tool={DENY}", "-p", prompt]
    if model: cmd += ["--model", model]
    p = run(cmd)
    if p.returncode != 0:
        raise RuntimeError(f"copilot failed: {p.stderr.strip()}")
    return p.stdout.strip()

def parse_json(s):
    s = s.strip()
    s = re.sub(r"^```(?:json)?\s*|\s*```$", "", s, flags=re.M)
    try: return json.loads(s)
    except json.JSONDecodeError:
        a, b = s.find("{"), s.rfind("}")
        if a != -1 and b != -1 and b > a:
            return json.loads(s[a:b+1])
        raise

def load_skill(skill_name):
    p = Path(".claude/skills")/skill_name/"SKILL.md"
    if not p.exists():
        raise FileNotFoundError(f"Missing {p}")
    txt = p.read_text(encoding="utf-8")
    fm, body = {}, txt
    if txt.startswith("---"):
        parts = txt.split("---", 2)
        if len(parts) >= 3:
            fm_txt, body = parts[1], parts[2].lstrip()
            for line in fm_txt.splitlines():
                if ":" in line:
                    k,v = line.split(":",1)
                    fm[k.strip()] = v.strip()
    side = p.parent/".copi-runner.json"
    sidecfg = {}
    if side.exists():
        sidecfg = json.loads(side.read_text(encoding="utf-8"))
    return p, fm, body, sidecfg

def render_skill(body, args, session_id, skill_dir):
    rendered = body.replace("${CLAUDE_SESSION_ID}", session_id)\
                   .replace("${CLAUDE_SKILL_DIR}", str(skill_dir))
    rendered = rendered.replace("$ARGUMENTS", " ".join(args))
    for i,a in enumerate(args):
        rendered = rendered.replace(f"$ARGUMENTS[{i}]", a).replace(f"${i}", a)
    if "$ARGUMENTS" not in body and args:
        rendered += f"\n\nARGUMENTS: {' '.join(args)}\n"
    return rendered

def safe_paths_from_diff(diff):
    paths = []
    for ln in diff.splitlines():
        if ln.startswith("+++ ") or ln.startswith("--- "):
            p = ln[4:].strip()
            if p == "/dev/null": continue
            p = re.sub(r"^[ab]/", "", p)
            if p.startswith("/") or ".." in Path(p).parts:
                raise ValueError(f"Unsafe path in diff: {p}")
            paths.append(p)
    return sorted(set(paths))

def apply_diff(diff):
    safe_paths_from_diff(diff)
    patchfile = Path(".copi-runner")/"tmp.patch"
    patchfile.parent.mkdir(parents=True, exist_ok=True)
    patchfile.write_text(diff, encoding="utf-8")
    if shutil.which("git") and Path(".git").exists():
        p = run(["git","apply","--whitespace=nowarn", str(patchfile)])
        if p.returncode == 0: return "git apply"
        raise RuntimeError(f"git apply failed:\n{p.stderr}")
    if shutil.which("patch"):
        p = run(["patch","-p1","--forward","--silent"], input=diff)
        if p.returncode == 0: return "patch"
        raise RuntimeError(f"patch failed:\n{p.stderr}")
    raise RuntimeError("No git or patch available to apply unified diff")

def choose_test_cmd(sidecfg):
    if isinstance(sidecfg, dict) and sidecfg.get("test_cmd"):
        return sidecfg["test_cmd"]
    if shutil.which("pytest"):
        return "pytest -q"
    return f"{shutil.which('python3') or 'python'} -m compileall ."

def main():
    if len(sys.argv) < 3:
        print("Usage: copi_runner.py <skill-name> <target-file> [args...]", file=sys.stderr)
        return 2
    skill_name, target = sys.argv[1], sys.argv[2]
    args = sys.argv[2:]
    session_id = f"s{int(time.time())}"
    skill_path, fm, body, side = load_skill(skill_name)
    rendered = render_skill(body, args, session_id, skill_path.parent)

    os.makedirs(".copi-runner/sessions", exist_ok=True)
    sess_path = Path(".copi-runner/sessions")/f"{session_id}.json"

    planner = f"""Return ONE valid JSON object only.
Rendered skill instructions:
<<<
{rendered}
>>>
Output schema: {{"type":"plan","session_id":"{session_id}","steps":[{{"id":"step-1","goal":"...","success_criteria":["..."],"allowed_runner_tools":["read_file","apply_patch","run_cmd"]}}]}}"""
    plan = parse_json(copilot(planner, model=side.get("model") if isinstance(side, dict) else None))

    toolcall_prompt = f"""Return ONE valid JSON object only.
Choose exactly ONE tool_call to read the target file.
Schema: {{"type":"tool_call","session_id":"{session_id}","tool":"read_file","args":{{"path":"{target}"}}, "rationale":"short"}}"""
    toolreq = parse_json(copilot(toolcall_prompt))
    file_text = Path(toolreq["args"]["path"]).read_text(encoding="utf-8")

    exec_prompt = f"""Return ONE valid JSON object only.
Produce a minimal unified diff implementing the skill.
Rendered instructions:
<<<
{rendered}
>>>
File content:
<<<FILE {target}
{file_text}
FILE>>>
Schema: {{"type":"apply_patch","session_id":"{session_id}","diff":"UNIFIED_DIFF","files_touched":["{target}"],"rationale":"short"}}"""
    patch = parse_json(copilot(exec_prompt))
    how = apply_diff(patch["diff"])

    test_cmd = choose_test_cmd(side if isinstance(side, dict) else {})
    t = run(test_cmd, shell=True)
    test_out = (t.stdout + "\n" + t.stderr).strip()[:8000]

    review_prompt = f"""Return ONE valid JSON object only.
Diff:
<<<DIFF
{patch["diff"]}
DIFF>>>
Test output:
<<<TEST
{test_out}
TEST>>>
Schema: {{"type":"review","session_id":"{session_id}","verdict":"pass|retry|fail","issues":[{{"severity":"high|med|low","detail":"..."}}],"next_action":"short"}}"""
    review = parse_json(copilot(review_prompt))

    final_prompt = f"""Return ONE valid JSON object only.
Write a concise final summary for the user.
Applied via: {how}
Test command: {test_cmd} (exit={t.returncode})
Diff:
<<<DIFF
{patch["diff"]}
DIFF>>>
Test output:
<<<TEST
{test_out}
TEST>>>
Schema: {{"type":"final","session_id":"{session_id}","summary":"string","diff":"string","test_output_excerpt":"string"}}"""
    final = parse_json(copilot(final_prompt))

    sess_path.write_text(json.dumps({
        "session_id": session_id,
        "skill": skill_name,
        "skill_path": str(skill_path),
        "frontmatter": fm,
        "plan": plan,
        "tool_request": toolreq,
        "apply_patch": {"how": how, "files": patch.get("files_touched", []), "diff": patch["diff"]},
        "test": {"cmd": test_cmd, "exit": t.returncode, "output": test_out},
        "review": review,
        "final": final
    }, indent=2), encoding="utf-8")

    print(final["summary"])
    return 0 if review.get("verdict") == "pass" and t.returncode == 0 else 1

if __name__ == "__main__":
    raise SystemExit(main())
```

## Worked example: a tiny Claude skill + the prompt/command sequence

### Example SKILL.md

Create `.claude/skills/fix-add-bug/SKILL.md`:

```markdown
---
name: fix-add-bug
description: Fix a simple logic bug in a Python function and ensure tests/build pass.
allowed-tools: Read, Edit
argument-hint: [file-path]
---

Fix the bug in the Python file at $0.

Rules:
- Make the minimal change to correct behavior.
- Do not refactor unrelated code.
- After changes, ensure the project still builds/tests by running a lightweight command.

Output:
- A short summary of the bug and the fix.
```

This uses Claude’s documented `$0` positional argument convention. citeturn4view2turn4view4

Create a buggy file `demo/math_utils.py`:

```python
def add(a, b):
    return a - b  # BUG: should add
```

Optionally add `.claude/skills/fix-add-bug/.copi-runner.json`:

```json
{"test_cmd":"python -m compileall ."}
```

### What the script runs (conceptually)

The script will execute this exact high-level sequence:

1. **Planner (Copilot CLI prompt mode):** `copilot -p "<planner prompt>" -s --no-ask-user --deny-tool=...` citeturn3view1turn5view3  
2. **Tool-call request:** prompt Copilot to output `{"type":"tool_call","tool":"read_file","args":{"path":"demo/math_utils.py"}}`  
3. **Local file read:** runner reads file content  
4. **Patch generation:** prompt Copilot to output `{"type":"apply_patch","diff":"..."}` (unified diff)  
5. **Patch apply:** runner applies unified diff (via `git apply` or `patch`)  
6. **Test/build command:** runner executes sidecar `test_cmd` (or autodetect)  
7. **Review prompt:** Copilot reviews diff + test output and returns `pass|retry|fail` JSON  
8. **Final prompt:** Copilot emits the final user-facing summary JSON

This is aligned with Copilot’s official programmatic mode (`-p`) and scripting output (`-s`). citeturn3view1turn3view0

## Minimal implementation plan, effort estimate, and key risks

### Plan (smallest useful version)

- Implement `SKILL.md` parsing + minimal frontmatter extraction
- Implement Claude substitutions (`$ARGUMENTS`, `$0`, `${CLAUDE_*}`) and `ARGUMENTS:` append behavior (recommended for parity) citeturn4view2turn4view4  
- Implement 4 prompt templates (plan/toolcall/patch/review/final)
- Implement safe diff application (prefer `git apply`, fallback to `patch`) + path safety checks
- Implement one validation command runner
- Persist session JSON in `.copi-runner/sessions`

### Effort estimate (prototype)

- **4–8 hours**: working single-script prototype in Python (similar to above), including basic safety checks and JSON-parse heuristics.
- **+6–12 hours**: broaden compatibility (more frontmatter fields, `!`command preprocessing, more tools like grep/glob, bounded retries).
- **+1–2 days**: harden patch application, add robust tool allowlists, add a small fixture suite of “real” skills.

### Top risks and mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| Non-JSON responses or partial JSON | Breaks deterministic orchestration | Enforce “JSON only” prompts; use `-s`; add extraction/repair heuristic; implement bounded retry. citeturn3view1turn3view0 |
| Unsafe diffs (path traversal, unexpected file edits) | Security + repo integrity | Parse diff headers and reject absolute/`..` paths; optionally restrict to target file(s). |
| Dangerous shell commands | Skills may request execution | Keep runner command allowlist; deny by default; in Copilot CLI, continue denying tools with `--deny-tool`. citeturn5view3turn5view4 |
| Environment variability (tools installed, test commands differ) | Prototype may fail on some repos | Sidecar config `.copi-runner.json` for `test_cmd`; fallback to simple checks (e.g., `python -m compileall`). |
| Compatibility drift across ecosystems | Claude/Copilot docs evolve | Keep protocol stable; isolate prompt templates; add a small regression suite. |

### Optional: Spec Kit integration (recommended for validation)

If you use Spec Kit, treat “Claude skill runner compatibility” as a spec with acceptance tests. GitHub positions Spec Kit as a toolkit for spec-driven development that works across agent workflows (including Copilot). citeturn0search7turn0search3turn0search22  
For a lightweight runner, the most valuable Spec Kit output is a small suite of fixtures:
- skill parsing tests (frontmatter + substitutions),
- patch application tests,
- and 2–3 end-to-end skills that run in a temp repo.

## Prioritized sources to consult

1. GitHub Copilot CLI programmatic usage (`-p`, `-s`, CI examples). citeturn3view0turn3view1  
2. GitHub Copilot CLI command reference (tool allow/deny, output formats, `--no-custom-instructions`, `--no-ask-user`). citeturn5view3turn5view0turn5view2  
3. GitHub: Creating agent skills for Copilot CLI (skill directory locations; `SKILL.md` structure). citeturn3view2  
4. Claude Code skills documentation (frontmatter fields, substitutions, `!`command preprocessing, tool restriction). citeturn4view1turn4view4  
5. GitHub Spec Kit repo + documentation (spec-driven workflow framing). citeturn0search3turn0search22turn0search7
