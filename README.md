# Claude Code — Token Self-Optimization Prompt

> **Usage :** Paste this prompt directly into a Claude Code session.
> Claude will run the audit and optimization autonomously, phase by phase.
> **Series :** Companion to [claude-code-best-practice-playbook](https://github.com/papasega/claude-code-best-practice-playbook) — full setup & workflow reference.

> **Compatibility :** use the version documented by the canonical
> [playbook](https://github.com/papasega/claude-code-best-practice-playbook#claude-code--best-practice-playbook)
> and verify locally with `claude --version`.

---

![Claude Code Best Practice](./ndapli/ccto_psw.jpg)

## Context & Objective

You are tasked with performing a comprehensive self-optimization of this Claude Code environment to **reduce startup context and per-turn token waste** while measuring and minimizing regressions in task completion quality. This is a systematic audit-and-refactor mission.

Work through each phase below **in order**. Create only the files specified here or
in the linked canonical Playbook sections. Report token estimates at each step. Do
not ask for confirmation between phases — execute autonomously.

> **On the numbers in this document.** Token figures derived from character counts
> are rough estimates, not measurements of model tokenization or your workload.
> This prompt ships **no evaluation harness**, so quality neutrality is *not*
> demonstrated: treat every reduction as a hypothesis to validate on your own
> representative tasks before adopting it broadly.

---

## PHASE 1 — AUDIT (measure before optimizing)

Run the following diagnostics and report results:

```bash
# 1a. Estimate current CLAUDE.md token footprint
echo "=== Global CLAUDE.md ===" && wc -l ~/.claude/CLAUDE.md 2>/dev/null || echo "No global CLAUDE.md"
echo "=== Project CLAUDE.md ===" && wc -l .claude/CLAUDE.md 2>/dev/null || echo "No project CLAUDE.md"

# 1b. Count all @imported files referenced in CLAUDE.md
grep -h "@" ~/.claude/CLAUDE.md .claude/CLAUDE.md 2>/dev/null | grep -v "^#" | grep -v "<!--"

# 1c. List discoverable skills (descriptions load at startup; bodies load on use)
echo "=== User skills ===" && ls ~/.claude/skills/ 2>/dev/null || echo "none"
echo "=== Project skills ===" && ls .claude/skills/ 2>/dev/null || echo "none"

# 1d. List all subagents
echo "=== User agents ===" && ls ~/.claude/agents/ 2>/dev/null || echo "none"
echo "=== Project agents ===" && ls .claude/agents/ 2>/dev/null || echo "none"

# 1e. Check model and effort level
cat ~/.claude/settings.json 2>/dev/null | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('model:', d.get('model', 'not set'))
print('effortLevel:', d.get('effortLevel', 'not set'))
mtt = d.get('env', {}).get('MAX_THINKING_TOKENS', 'not set')
print('MAX_THINKING_TOKENS:', mtt, '(0 disables thinking; nonzero values apply to fixed-budget modes)')
" 2>/dev/null || echo "No ~/.claude/settings.json"

# 1f. Check project hooks
cat .claude/settings.json 2>/dev/null | python3 -c "
import json, sys
d = json.load(sys.stdin)
hooks = d.get('hooks', {})
print('Hooks configured:', list(hooks.keys()) if hooks else 'none')
" 2>/dev/null || echo "No .claude/settings.json"

# 1g. Estimate total startup token footprint
python3 -c "
import os

def tokens(path):
    try:
        with open(os.path.expanduser(path)) as f:
            return len(f.read()) // 4
    except (OSError, UnicodeError):
        return 0

g = tokens('~/.claude/CLAUDE.md')
p = tokens('.claude/CLAUDE.md')
print(f'CLAUDE.md context estimate: ~{g + p} tokens per session')
print(f'  Global CLAUDE.md : ~{g} tokens')
print(f'  Project CLAUDE.md: ~{p} tokens')
print('  Skill descriptions and other startup context are not included; inspect /context.')
"
```

**Report the audit summary before proceeding to Phase 2.**

---

## PHASE 2 — CLAUDE.md OPTIMIZATION

> **Rule : CLAUDE.md must be ≤ 200 lines. Everything workflow-specific goes into on-demand Skills.**

### 2a. Analyze and identify what to remove

In the current CLAUDE.md files, identify and tag each section:

| Category                                                    | Action                                                 |
| ----------------------------------------------------------- | ------------------------------------------------------ |
| Workflow-specific (PR review, DB migrations, test patterns) | → Move to Skill                                       |
| Verbose explanations (>3 lines for one rule)                | → Compress to 1 line                                  |
| Context Claude already knows (basic Python, git, etc.)      | → Delete                                              |
| Old/deprecated patterns                                     | → Delete                                              |
| HTML comments `<!-- notes -->` inside CLAUDE.md           | → Keep (they are stripped from context automatically) |

### 2b. Rewrite `~/.claude/CLAUDE.md` (global) <> [see an example here to `CCBPP`](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#4-claudemd--the-right-way)

**Hard limit : 200 lines.** Content to keep:

- Identity and communication style
- Universal code standards (language, formatting, logging)
- Tool preferences (e.g. loguru over stdlib, pathlib over os.path)
- Production-ready code checklist (condensed to bullet list)

**Always include this compact instruction block at the top:**

```markdown
# Compact instructions
When compacting, preserve: modified file list, test commands, active task state, TODO markers.
Discard: exploration history, failed attempts, verbose tool output logs.
```

### 2c. Rewrite `.claude/CLAUDE.md` (project)

**Hard limit : 150 lines.** Content to keep:

- Project architecture (1 paragraph max)
- Key commands: build / test / deploy / lint
- Directory structure (top-level only)
- Critical project-specific rules (3–5 rules max)

**Do NOT copy global rules** — they are already loaded from `~/.claude/CLAUDE.md`.
**Move all PR / DB / deploy workflows** → Skills (Phase 3).

### 2d. Report token delta

```bash
python3 -c "
import os

def tokens(path):
    try:
        with open(os.path.expanduser(path)) as f:
            return len(f.read()) // 4
    except (OSError, UnicodeError):
        return 0

print(f'Global CLAUDE.md after: ~{tokens(\"~/.claude/CLAUDE.md\")} tokens')
print(f'Project CLAUDE.md after: ~{tokens(\".claude/CLAUDE.md\")} tokens')
"
```

---

## PHASE 3 — SKILLS MIGRATION (on-demand body)

> A skill's **body** loads only when the skill is invoked. Its **name and description**
> still contribute to the discovery context Claude sees at startup, so skills are
> not free context. Keep descriptions short and move detail into the body.
> Moving a long workflow doc out of CLAUDE.md and into a skill body is what pays.
>
> CLAUDE.md is loaded at startup and remains in the context of following turns.
> Prompt caching does not remove those tokens from the context window.
>
> Keep each SKILL.md concise and limited to instructions the task actually needs.

### 3a. Create skill : `pdf-to-context`

**Path :** `.claude/skills/pdf-to-context/SKILL.md`

````markdown
---
name: pdf-to-context
description: Convert PDF or large document to token-efficient markdown before analysis.
  Triggers: "read this PDF", "analyze document", "summarize this file", any .pdf or .docx reference.
---

# PDF → Markdown Preprocessing

## ALWAYS do this before reading any PDF or large document into context

### Step 1: Extract text (never read raw PDF binary into context)
```bash
# Method A — pdftotext (fastest)
pdftotext "$FILE" - | head -c 50000

# Method B — pdfplumber (preserves tables)
python3 -c "
import pdfplumber, sys
with pdfplumber.open(sys.argv[1]) as pdf:
    text = '\n'.join(p.extract_text() or '' for p in pdf.pages)
    print(text[:50000])
" "$FILE"

```

### Step 2: Targeted extraction — answer the question, don't dump everything

1. Identify the USER'S QUESTION first
2. Extract only sections relevant to that question
3. Filter: `grep -A5 -B2 "keyword" extracted.md`
4. State any extraction or truncation limit in the answer
5. For a full-document task, process all sections rather than silently truncating

### For DOCX files

```bash
python3 -c "import docx2txt, sys; print(docx2txt.process(sys.argv[1])[:50000])" "$FILE"

```

### For large logs / test output — never cat the full file

```bash
tail -100 logfile.log                        # Last 100 lines
grep -i "error\|warning\|fail" log.txt       # Errors only
grep -A3 "FAILED" test_output.txt            # Failed tests with context

```
````

### 3b. Create skill : `context-manager`

**Path :** `.claude/skills/context-manager/SKILL.md`

````markdown
---
name: context-manager
description: Manage context window health during long sessions.
  Triggers: "context getting big", "clear context", "start fresh", "compact session",
  Claude repeating mistakes or losing earlier constraints.
---

# Context Window Management

## When to act
| Signal | Action |
|--------|--------|
| `/context` shows a crowded window | `/compact` |
| Switching to unrelated task | `/clear` (use `/rename` first) |
| Claude repeating mistakes / forgetting | `/compact` mandatory |
| Quick one-off question | `/btw [question]` — never enters context |

## Optimal /compact command
```
/compact Focus on: modified files list, current task state, failing test names, key decisions.
Discard: exploration history, verbose tool outputs, failed attempts.
```

## Subagent delegation (isolated context, not free)
For ANY research/exploration task, use this pattern:
> "Use a subagent to investigate [TOPIC] and return a 200-word summary:
> key findings, relevant file paths, recommended approach."

Subagents run in **separate context windows** → your main context stays clean.
The subagent uses its own context, so this keeps exploration out of the main
window rather than eliminating token use.

## Session hygiene checklist
- [ ] `/clear` between unrelated tasks
- [ ] Use `/context`; compact when the current task needs room
- [ ] Use subagents for codebase exploration
- [ ] Use targeted extraction to locate content in large files
- [ ] Read the full file when correctness depends on global context
````

### 3c. Use the built-in model and effort controls

Do not generate a `model-selector` skill or copy a model matrix into this repository.
Model availability and capabilities evolve. Use `/model` and `/effort`, and follow
the canonical [playbook model and effort reference](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#model-and-effort).

Keep these stable rules:

- inherit the active model unless the task has an evaluated reason to override it;
- use `/effort auto` to return to the model default;
- treat lower effort as a capability trade-off and validate it on representative tasks;
- do not use `MAX_THINKING_TOKENS` as a cap on adaptive-reasoning models.

### 3d. Use the canonical targeted-reading guidance

Do not create a second file-reading skill. Use the canonical large-file advisory
hook from [Playbook §6](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#hook-2--advise-on-large-file-reads)
and the canonical `code-explorer` subagent from
[Playbook §8](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#subagent-code-explorer).
Use targeted extraction to locate content; read the complete file whenever
correctness depends on its global invariants or control flow.

---

## PHASE 4 — SETTINGS & SECURITY OPTIMIZATION

### 4a. Update `~/.claude/settings.json` (global defaults + git safety). [Find the full example here](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#2-project-configuration--settingsjson)

This file applies to **every project on your machine**, so the script below is
written to be **preservative, backed up, and atomic**:

- it never overwrites a key you already set — it only adds what is missing;
- it refuses to touch a file it could not parse, rather than replacing it with a
  fresh one and silently losing your configuration;
- it copies the current file to a timestamped `.bak` before changing anything, and
  skips the backup entirely when the merge produces no change;
- it preserves an explicit attribution choice; empty commit and PR attribution are
  defaults only when those fields are absent;
- it writes to a temporary file in the same directory, `fsync`s it, re-parses it to
  confirm it is valid JSON, and only then swaps it into place with `os.replace()`,
  so an interrupted run cannot leave a truncated settings file;
- running it twice is a no-op: no duplicated rules, no second backup.

This step configures **three layers of permission** :
- `deny` — hard block (git push, git reset --hard, rm -rf, read/edit secrets)
- `ask` — Claude asks for your confirmation first (git commit, checkout, curl, wget)
- `allow` — pre-approved read-only commands (git diff, grep, find)

```bash
python3 << 'EOF'
import json, os, shutil, sys, tempfile
from datetime import datetime
from pathlib import Path

path = Path("~/.claude/settings.json").expanduser().resolve()
path.parent.mkdir(parents=True, exist_ok=True)

# Read the existing config. An unparseable file stops the run: we never replace
# a configuration we could not read.
if path.exists():
    try:
        original = path.read_text(encoding="utf-8")
        settings = json.loads(original)
    except (OSError, UnicodeError) as exc:
        sys.exit(f"ERROR: cannot read {path}: {exc}")
    except json.JSONDecodeError as exc:
        sys.exit(
            f"ERROR: {path} is not valid JSON "
            f"(line {exc.lineno}, column {exc.colno}): {exc.msg}\n"
            f"Nothing was modified. Fix the file or move it aside, then re-run."
        )
    if not isinstance(settings, dict):
        sys.exit(f"ERROR: {path} must contain a JSON object. Nothing was modified.")
else:
    original = None
    settings = {}


def add_missing(container, key, values):
    """Append only values that are absent, so re-running adds no duplicates."""
    existing = container.setdefault(key, [])
    if not isinstance(existing, list):
        sys.exit(
            f"ERROR: settings.permissions.{key} must be a JSON array. "
            "Nothing was modified."
        )
    for value in values:
        if value not in existing:
            existing.append(value)


# effortLevel is deliberately NOT set here. Inherit the active model's default;
# any override is a capability decision to make explicitly and validate per task.

# ── PERMISSIONS ─────────────────────────────────────────────
permissions = settings.setdefault("permissions", {})
if not isinstance(permissions, dict):
    sys.exit("ERROR: settings.permissions must be a JSON object. Nothing was modified.")

# Allow: pre-approved read-only commands (no confirmation prompt)
add_missing(permissions, "allow", [
    "Bash(git diff *)", "Bash(git log *)", "Bash(git status *)",
    "Bash(wc *)", "Bash(grep *)", "Bash(find *)",
    "Bash(head *)", "Bash(tail *)", "Bash(sed -n *)",
])

# Deny: hard block. A trailing " *" also matches the bare command, so
# "Bash(git push *)" covers "git push" with no arguments.
add_missing(permissions, "deny", [
    # Protect secrets
    "Read(./.env)", "Read(./.env.*)",
    "Read(./secrets/**)", "Read(./.git/objects/**)",
    "Edit(.env)", "Edit(.env.*)",
    "Edit(./secrets/**)", "Edit(.git/**)",
    # Destructive or history-rewriting git operations
    "Bash(git push *)",
    "Bash(git reset --hard *)",
    # Destructive system commands
    "Bash(rm -rf *)",
])

# Ask: Claude requests your confirmation before executing.
# curl and wget sit here rather than in deny: they are legitimate in many
# workflows, and blocking them is a control on explicit shell network access,
# NOT a general defence against data exfiltration (see the note below).
add_missing(permissions, "ask", [
    "Bash(git commit *)",
    "Bash(git checkout *)",
    "Bash(git branch -d *)",
    "Bash(git stash *)",
    "Bash(curl *)",
    "Bash(wget *)",
])

# ── ATTRIBUTION ─────────────────────────────────────────────
# Default to no commit or PR attribution without replacing an explicit preference.
attribution = settings.setdefault("attribution", {})
if not isinstance(attribution, dict):
    sys.exit("ERROR: settings.attribution must be a JSON object. Nothing was modified.")
attribution.setdefault("commit", "")
attribution.setdefault("pr", "")

# ── GITIGNORE ───────────────────────────────────────────────
settings.setdefault("respectGitignore", True)

# ── WRITE: no-op check, backup, atomic replace ──────────────
updated = json.dumps(settings, indent=2, ensure_ascii=False) + "\n"

if original is not None and updated == original:
    print(f"{path} already up to date — nothing changed, no backup written.")
    raise SystemExit(0)

if path.exists():
    backup = path.with_name(f"{path.name}.{datetime.now():%Y%m%d-%H%M%S-%f}.bak")
    shutil.copy2(path, backup)
    print(f"Backup written: {backup}")

mode = (path.stat().st_mode & 0o777) if path.exists() else 0o600
fd, tmp_name = tempfile.mkstemp(dir=path.parent, prefix=path.name + ".", suffix=".tmp")
try:
    with os.fdopen(fd, "w", encoding="utf-8") as handle:
        handle.write(updated)
        handle.flush()
        os.fsync(handle.fileno())
    # Re-parse before swapping: never promote a file we cannot read back.
    json.loads(Path(tmp_name).read_text(encoding="utf-8"))
    os.chmod(tmp_name, mode)
    os.replace(tmp_name, path)
except BaseException:
    Path(tmp_name).unlink(missing_ok=True)
    raise

print(f"{path} updated")
print(f"   model           : {settings.get('model', 'not set (inherits account default)')}")
print(f"   effortLevel     : {settings.get('effortLevel', 'not set (inherits model default)')}")
print(f"   deny rules      : {len(permissions['deny'])}")
print(f"   ask rules       : {len(permissions['ask'])}")
print(f"   allow rules     : {len(permissions['allow'])}")
print(f"   attribution     : commit='{attribution['commit']}' pr='{attribution['pr']}'")
print(f"   respectGitignore: {settings['respectGitignore']}")
EOF
```

> **Scope of the network rules.** Putting `curl` and `wget` behind `ask` controls
> *explicit shell network access*. It is not an exfiltration policy: package
> managers, language runtimes, `npx`, MCP servers and any other tool with a socket
> can still reach the network, and these rules deliberately do not block them.
> Treat this as one narrow control among several, not as a boundary.

### 4b. Create `.claude/hooks/advise-large-read.sh` (large file advisor)

Use the canonical implementation in
[Playbook §6, Hook 2](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#hook-2--advise-on-large-file-reads).
Ensure `.claude/hooks/` exists, then create the target project's file exactly as
specified there and make it executable. This prompt links to the implementation
instead of embedding a second source copy. The canonical hook advises without
denying the read and does not bypass the normal permission flow.

### 4c. Create `.claude/hooks/validate-bash.sh` (dangerous command interceptor)

Use the canonical implementation and limitations documented in
[Playbook §6, Hook 1](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#hook-1--intercept-known-dangerous-bash-commands).
Create the target project's file exactly as specified there and make it executable.
This prompt links to the implementation instead of embedding a second source copy.
Keep the permission rules from Phase 4a as a separate layer; the hook is defence in
depth, not a complete shell parser.

### 4d. Register both hooks in the **project** `.claude/settings.json`

These hooks go in the **project** file, not the global one. Their commands resolve
through `$CLAUDE_PROJECT_DIR/.claude/hooks/...`, which only exists in a project that
ran Phase 4b and 4c. Registering them in `~/.claude/settings.json` would make every
other project on your machine invoke a script that is not there.

| File | Scope | What belongs there |
|------|-------|--------------------|
| `~/.claude/settings.json` | every project | permissions, attribution and file-picker behavior (Phase 4a) |
| `.claude/settings.json` | this project | hooks pointing at this project's scripts (below) |

The script below **merges** into any existing project config: it keeps every hook and
matcher already present, adds only what is missing, refuses to overwrite a file it
cannot parse, and writes atomically. Running it twice changes nothing the second time.

```bash
python3 << 'EOF'
import json, os, shutil, sys, tempfile
from datetime import datetime
from pathlib import Path

path = Path(".claude/settings.json").resolve()
path.parent.mkdir(parents=True, exist_ok=True)

if path.exists():
    try:
        original = path.read_text(encoding="utf-8")
        settings = json.loads(original)
    except (OSError, UnicodeError) as exc:
        sys.exit(f"ERROR: cannot read {path}: {exc}")
    except json.JSONDecodeError as exc:
        sys.exit(
            f"ERROR: {path} is not valid JSON "
            f"(line {exc.lineno}, column {exc.colno}): {exc.msg}\n"
            f"Nothing was modified."
        )
    if not isinstance(settings, dict):
        sys.exit(f"ERROR: {path} must contain a JSON object. Nothing was modified.")
else:
    original = None
    settings = {}

WANTED = {
    "Bash": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.sh",
    "Read": "$CLAUDE_PROJECT_DIR/.claude/hooks/advise-large-read.sh",
}

hooks = settings.setdefault("hooks", {})
if not isinstance(hooks, dict):
    sys.exit("ERROR: settings.hooks must be a JSON object. Nothing was modified.")
pre_tool_use = hooks.setdefault("PreToolUse", [])
if not isinstance(pre_tool_use, list):
    sys.exit(
        "ERROR: settings.hooks.PreToolUse must be a JSON array. Nothing was modified."
    )

for matcher, command in WANTED.items():
    # Find an existing entry for this matcher instead of appending a rival one.
    entry = next(
        (e for e in pre_tool_use
         if isinstance(e, dict) and e.get("matcher") == matcher),
        None,
    )
    if entry is None:
        pre_tool_use.append({
            "matcher": matcher,
            "hooks": [{"type": "command", "command": command}],
        })
        continue

    entry_hooks = entry.setdefault("hooks", [])
    if not isinstance(entry_hooks, list):
        sys.exit(
            f"ERROR: hooks for matcher {matcher!r} must be a JSON array. "
            "Nothing was modified."
        )
    already = any(
        isinstance(h, dict) and h.get("command") == command for h in entry_hooks
    )
    if not already:
        entry_hooks.append({"type": "command", "command": command})

updated = json.dumps(settings, indent=2, ensure_ascii=False) + "\n"

if original is not None and updated == original:
    print(f"{path} already registers both hooks — nothing changed.")
    raise SystemExit(0)

if path.exists():
    backup = path.with_name(f"{path.name}.{datetime.now():%Y%m%d-%H%M%S-%f}.bak")
    shutil.copy2(path, backup)
    print(f"Backup written: {backup}")

mode = (path.stat().st_mode & 0o777) if path.exists() else 0o600
fd, tmp_name = tempfile.mkstemp(dir=path.parent, prefix=path.name + ".", suffix=".tmp")
try:
    with os.fdopen(fd, "w", encoding="utf-8") as handle:
        handle.write(updated)
        handle.flush()
        os.fsync(handle.fileno())
    json.loads(Path(tmp_name).read_text(encoding="utf-8"))
    os.chmod(tmp_name, mode)
    os.replace(tmp_name, path)
except BaseException:
    Path(tmp_name).unlink(missing_ok=True)
    raise

print(f"{path} updated — PreToolUse entries: {len(pre_tool_use)}")
EOF
```

---

## PHASE 5 — SUBAGENT FOR CODEBASE EXPLORATION

Use the canonical `code-explorer` specification from
[Playbook §8](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#subagent-code-explorer).
Create `.claude/agents/code-explorer.md` exactly as specified there unless the file
already exists. This prompt links to the specification instead of embedding a second
source copy: the Playbook is the single source of truth. The relevant optimization
is context isolation; inherit the active model and effort unless project evaluations
justify an override.

---

## PHASE 6 — VERIFICATION & FINAL REPORT

Run the full verification suite and generate the context report:

```bash
python3 << 'EOF'
import glob, json, os, subprocess, sys, tempfile
from pathlib import Path

failures = []


def tokens(path):
    try:
        with open(os.path.expanduser(path), encoding="utf-8") as f:
            return len(f.read()) // 4
    except (OSError, UnicodeError):
        return 0


def lines(path):
    try:
        with open(os.path.expanduser(path), encoding="utf-8") as f:
            return sum(1 for _ in f)
    except (OSError, UnicodeError):
        return 0


def load_json(path):
    try:
        return json.loads(Path(path).expanduser().read_text(encoding="utf-8"))
    except (OSError, UnicodeError, json.JSONDecodeError):
        return None


def run_hook(script, payload):
    """Feed one JSON payload to a hook. Returns (exit code, stdout)."""
    proc = subprocess.run(
        ["bash", script],
        input=payload if isinstance(payload, str) else json.dumps(payload),
        capture_output=True,
        text=True,
    )
    return proc.returncode, proc.stdout


def check(label, ok, detail=""):
    suffix = f" — {detail}" if detail and not ok else ""
    print(f"  {'[OK]  ' if ok else '[FAIL]'} {label}{suffix}")
    if not ok:
        failures.append(label)
    return ok


print("=" * 60)
print("CLAUDE CODE TOKEN OPTIMIZATION — FINAL REPORT")
print("=" * 60)

# ── CLAUDE.md ─────────────────────────────────────────────
g_tokens = tokens("~/.claude/CLAUDE.md")
p_tokens = tokens(".claude/CLAUDE.md")
g_lines  = lines("~/.claude/CLAUDE.md")
p_lines  = lines(".claude/CLAUDE.md")

print(f"\nCLAUDE.md files (loaded every session)")
print(f"  Global  : {g_lines} lines — ~{g_tokens} tokens")
print(f"  Project : {p_lines} lines — ~{p_tokens} tokens")
print(f"  Total CLAUDE.md estimate: ~{g_tokens + p_tokens} tokens")

# ── Skills ────────────────────────────────────────────────
skill_files = glob.glob(".claude/skills/*/SKILL.md")
skill_tokens = sum(tokens(f) for f in skill_files)
print(f"\nSkills (regular session: body loads on invocation; description stays discoverable)")
for f in skill_files:
    t = tokens(f)
    print(f"  {os.path.basename(os.path.dirname(f))}: ~{t} tokens")
print(f"  Total skill content: ~{skill_tokens} tokens (regular session: loaded on invocation)")

# ── Subagents ─────────────────────────────────────────────
agent_files = glob.glob(".claude/agents/*.md")
print(f"\nSubagents configured: {len(agent_files)}")
for f in agent_files:
    print(f"  {os.path.basename(f)}")
check(
    "code-explorer subagent exists",
    any(os.path.basename(f) == "code-explorer.md" for f in agent_files),
)

# ── Global settings ───────────────────────────────────────
s = load_json("~/.claude/settings.json")
if s is None:
    print("\n[FAIL] Could not read or parse ~/.claude/settings.json")
    failures.append("global settings readable")
    perms, attr = {}, {}
else:
    perms = s.get("permissions", {})
    attr = s.get("attribution", {})
    print(f"\nSettings (~/.claude/settings.json)")
    print(f"  model            : {s.get('model', 'not set')}")
    print(f"  effortLevel      : {s.get('effortLevel', 'not set (inherits model default)')}")
    print(f"  respectGitignore : {s.get('respectGitignore', 'not set')}")
    print(f"  attribution      : commit='{attr.get('commit', 'not set')}' pr='{attr.get('pr', 'not set')}'")
    print(f"\nPermissions")
    for bucket, mark in (("deny", "x"), ("ask", "?"), ("allow", "+")):
        rules = perms.get(bucket, [])
        print(f"  {bucket} rules : {len(rules)}")
        for r in rules:
            print(f"    {mark} {r}")

# ── Hook verification ─────────────────────────────────────
# A file on disk is not an active hook. Each one must exist, be executable,
# be registered under the right matcher, and actually behave as documented.
print(f"\nHook verification")

project = load_json(".claude/settings.json")
registered = {}
if isinstance(project, dict):
    for entry in project.get("hooks", {}).get("PreToolUse", []):
        if not isinstance(entry, dict):
            continue
        commands = [
            h.get("command", "")
            for h in entry.get("hooks", [])
            if isinstance(h, dict)
        ]
        registered.setdefault(entry.get("matcher"), []).extend(commands)
else:
    check("project .claude/settings.json readable", False, "missing or invalid JSON")

BASH_HOOK = ".claude/hooks/validate-bash.sh"
READ_HOOK = ".claude/hooks/advise-large-read.sh"

for hook, matcher in ((BASH_HOOK, "Bash"), (READ_HOOK, "Read")):
    name = os.path.basename(hook)
    if check(f"{name}: file exists", os.path.isfile(hook)):
        check(f"{name}: executable", os.access(hook, os.X_OK), "run chmod +x")
    check(
        f"{name}: registered under matcher '{matcher}'",
        any(name in command for command in registered.get(matcher, [])),
        "not referenced in .claude/settings.json",
    )

if os.path.isfile(BASH_HOOK):
    code, _ = run_hook(BASH_HOOK, {"tool_name": "Bash", "tool_input": {"command": "rm -rf /tmp/probe"}})
    check("validate-bash.sh: blocks rm -rf with exit 2", code == 2, f"exit {code}")
    code, _ = run_hook(BASH_HOOK, {"tool_name": "Bash", "tool_input": {"command": "git status"}})
    check("validate-bash.sh: allows git status with exit 0", code == 0, f"exit {code}")

if os.path.isfile(READ_HOOK):
    with tempfile.TemporaryDirectory() as directory:
        big = Path(directory) / "big.py"
        big.write_text("x = 1\n" * 301, encoding="utf-8")
        small = Path(directory) / "small.py"
        small.write_text("x = 1\n" * 20, encoding="utf-8")

        code, out = run_hook(READ_HOOK, {"tool_name": "Read", "tool_input": {"file_path": str(big)}})
        check("advise-large-read.sh: 301-line file advised, not refused", code == 0, f"exit {code}")
        emitted = json.loads(out) if out.strip() else {}
        hook_output = emitted.get("hookSpecificOutput", {})
        check("advise-large-read.sh: emits additionalContext", bool(hook_output.get("additionalContext")))
        check(
            "advise-large-read.sh: returns no permissionDecision",
            "permissionDecision" not in hook_output,
            "must not short-circuit the permission flow",
        )

        code, out = run_hook(READ_HOOK, {"tool_name": "Read", "tool_input": {"file_path": str(small)}})
        check("advise-large-read.sh: 20-line file stays silent", code == 0 and not out.strip(), f"exit {code}")

        code, out = run_hook(READ_HOOK, {"tool_name": "Read", "tool_input": {"file_path": "/nonexistent/path.py"}})
        check("advise-large-read.sh: missing path stays silent", code == 0 and not out.strip(), f"exit {code}")

        code, out = run_hook(READ_HOOK, "not valid json")
        check("advise-large-read.sh: invalid JSON stays silent", code == 0 and not out.strip(), f"exit {code}")

# ── Permission coverage ───────────────────────────────────
print(f"\nPermission coverage (rules present, not a proof of enforcement)")
check("git push denied", any("git push" in r for r in perms.get("deny", [])))
check("rm -rf denied", any("rm -rf" in r for r in perms.get("deny", [])))
check("git commit requires confirmation", any("git commit" in r for r in perms.get("ask", [])))
check(
    "curl/wget require confirmation",
    any("curl" in r for r in perms.get("ask", []))
    and any("wget" in r for r in perms.get("ask", [])),
)
if attr.get("commit") == "" and attr.get("pr") == "":
    print("  [OK]   commit and PR attribution default to empty")
else:
    print("  [INFO] explicit existing attribution was preserved")

print(f"\n{'=' * 60}")
print(f"   CLAUDE.md estimate: ~{g_tokens + p_tokens} tokens")
print(f"   Skill bodies      : ~{skill_tokens} tokens, loaded on invocation")
print(f"   Isolated exploration: code-explorer subagent configured")

if failures:
    print(f"\nRESULT: {len(failures)} check(s) FAILED")
    for item in failures:
        print(f"  - {item}")
    print("Nothing above is 'active' until these pass.")
    sys.exit(1)

print(f"\nRESULT: all checks passed")
print(f"{'=' * 60}")
EOF
```

Produce a final context summary from the Phase 6 output. The prompt's numerical
figures are rough character-count estimates for the local `CLAUDE.md` files and
skill bodies, not universal claims. Use the canonical
[Token Optimization Table in Playbook §14](https://github.com/papasega/claude-code-best-practice-playbook?tab=readme-ov-file#14-token-optimization-table)
for the mechanisms, trade-offs, and verification guidance; do not reproduce that
table in this repository.

---

## References

- [Hooks reference — JSON output and decision control](https://code.claude.com/docs/en/hooks)
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Best practices — Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- [Memory &amp; CLAUDE.md](https://code.claude.com/docs/en/memory)
- [Model configuration &amp; effort levels](https://code.claude.com/docs/en/model-config)

---

## Author

**Papa Sega WADE** — AI Research Engineer
Bridging software engineering and AI research — building tools, workflows, and systems that make LLM-assisted development practical at scale.

[papasegawade.com](https://papasegawade.com/) · [LinkedIn](https://www.linkedin.com/in/papa-s%C3%A9ga-wade-phd-a5727513a)

---
