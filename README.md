# Claude Code — Token Self-Optimization Prompt

> **Usage :** Paste this prompt directly into a Claude Code session.  
> Claude will run the audit and optimization autonomously, phase by phase.  
> **Series :** Companion to [claude-code-best-practice-playbook](https://github.com/papasega/claude-code-best-practice-playbook) — full setup & workflow reference.

> **Version :** Claude Code ≥ 2.1 · Last updated: 2026-03 · Verify with `claude --version`

---
![Claude Code Best Practice](./ndapli/ccto_psw.png)

## Context & Objective

You are tasked with performing a comprehensive self-optimization of this Claude Code environment to **minimize startup context cost and per-turn token waste** while maintaining full task performance. This is a systematic audit-and-refactor mission.

Work through each phase below **in order**. Create all files. Report token estimates at each step. Do not ask for confirmation between phases — execute autonomously.

---

## PHASE 1 — AUDIT (measure before optimizing)

Run the following diagnostics and report results:

```bash
# 1a. Estimate current CLAUDE.md token cost
echo "=== Global CLAUDE.md ===" && wc -l ~/.claude/CLAUDE.md 2>/dev/null || echo "No global CLAUDE.md"
echo "=== Project CLAUDE.md ===" && wc -l .claude/CLAUDE.md 2>/dev/null || echo "No project CLAUDE.md"

# 1b. Count all @imported files referenced in CLAUDE.md
grep -h "@" ~/.claude/CLAUDE.md .claude/CLAUDE.md 2>/dev/null | grep -v "^#" | grep -v "<!--"

# 1c. List all skills loaded at startup
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
print('MAX_THINKING_TOKENS:', d.get('env', {}).get('MAX_THINKING_TOKENS', 'not set'))
" 2>/dev/null || echo "No ~/.claude/settings.json"

# 1f. Check project hooks
cat .claude/settings.json 2>/dev/null | python3 -c "
import json, sys
d = json.load(sys.stdin)
hooks = d.get('hooks', {})
print('Hooks configured:', list(hooks.keys()) if hooks else 'none')
" 2>/dev/null || echo "No .claude/settings.json"

# 1g. Estimate total startup token cost
python3 -c "
import os, glob

def tokens(path):
    try:
        with open(os.path.expanduser(path)) as f:
            return len(f.read()) // 4
    except:
        return 0

g = tokens('~/.claude/CLAUDE.md')
p = tokens('.claude/CLAUDE.md')
skills_startup = 0  # skills are on-demand
print(f'Startup context cost today: ~{g + p} tokens per session')
print(f'  Global CLAUDE.md : ~{g} tokens')
print(f'  Project CLAUDE.md: ~{p} tokens')
"
```

**Report the audit summary before proceeding to Phase 2.**

---

## PHASE 2 — CLAUDE.md OPTIMIZATION

> **Rule : CLAUDE.md must be ≤ 200 lines. Everything workflow-specific goes into on-demand Skills.**

### 2a. Analyze and identify what to remove

In the current CLAUDE.md files, identify and tag each section:

| Category | Action |
|---|---|
| Workflow-specific (PR review, DB migrations, test patterns) | → Move to Skill |
| Verbose explanations (>3 lines for one rule) | → Compress to 1 line |
| Context Claude already knows (basic Python, git, etc.) | → Delete |
| Old/deprecated patterns | → Delete |
| HTML comments `<!-- notes -->` inside CLAUDE.md | → Keep (they are stripped from context automatically) |

### 2b. Rewrite `~/.claude/CLAUDE.md` (global)

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
    except:
        return 0

print(f'Global CLAUDE.md after: ~{tokens(\"~/.claude/CLAUDE.md\")} tokens')
print(f'Project CLAUDE.md after: ~{tokens(\".claude/CLAUDE.md\")} tokens')
"
```

---

## PHASE 3 — SKILLS MIGRATION (on-demand = zero startup cost)

> Skills load **only when invoked**. Moving workflow docs from CLAUDE.md to skills = **free tokens at startup**.  
> Each SKILL.md must be ≤ 80 lines, concise, no padding.

### 3a. Create skill : `pdf-to-context`

**Path :** `.claude/skills/pdf-to-context/SKILL.md`

```markdown
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
4. Hard cap: max 5 000 tokens of document content per question

### Token budget per task type
| Task | Max document tokens |
|------|-------------------|
| Simple factual question | 2 000 |
| Analysis / comparison | 10 000 |
| Full summary | Use subagent (isolated context) |

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
```

### 3b. Create skill : `context-manager`

**Path :** `.claude/skills/context-manager/SKILL.md`

```markdown
---
name: context-manager
description: Manage context window health during long sessions.
  Triggers: "context getting big", "clear context", "start fresh", "compact session",
  long debugging sessions >30 min, Claude repeating mistakes.
---

# Context Window Management

## When to act
| Signal | Action |
|--------|--------|
| Session > 30 min active work | `/compact` |
| Switching to unrelated task | `/clear` (use `/rename` first) |
| Claude repeating mistakes / forgetting | `/compact` mandatory |
| Quick one-off question | `/btw [question]` — never enters context |

## Optimal /compact command
```
/compact Focus on: modified files list, current task state, failing test names, key decisions.
Discard: exploration history, verbose tool outputs, failed attempts.
```

## Subagent delegation (zero main context cost)
For ANY research/exploration task, use this pattern:
> "Use a subagent to investigate [TOPIC] and return a 200-word summary:
> key findings, relevant file paths, recommended approach."

Subagents run in **separate context windows** → your main context stays clean.

## Session hygiene checklist
- [ ] `/clear` between unrelated tasks
- [ ] `/compact` every ~30 min on long sessions
- [ ] Use subagents for codebase exploration
- [ ] Never `cat` large files — always `grep/head/tail` first
- [ ] Prefer Bash commands over Read tool for large files
```

### 3c. Create skill : `model-selector`

**Path :** `.claude/skills/model-selector/SKILL.md`

```markdown
---
name: model-selector
description: Choose the right model and effort level per task to minimize cost.
  Triggers: "which model", "save tokens", "quick task", "simple fix", "complex architecture".
---

# Model & Effort Selection Matrix

| Task type | Model | Effort | Cost vs default |
|-----------|-------|--------|----------------|
| Typo fix, rename variable | haiku | low | ~10x cheaper |
| Write function, add test | sonnet | low | baseline |
| Debug complex bug | sonnet | medium | +30% |
| Architecture / design decision | sonnet high or opus | high | +3-5x, justified |
| Multi-file refactor | sonnet | medium | baseline |
| Subagent exploration tasks | haiku | low | ~10x cheaper |

## Session commands
```bash
/model          # Open model picker
/effort low     # Fast, cheap — routine tasks
/effort medium  # Default balanced (recommended)
/effort high    # Deep reasoning — justify the cost
```

## Subagent model override (in agent frontmatter)
```yaml
---
model: haiku
effort: low
---
```

## Cost multipliers reference
- Extended thinking (high effort) = output tokens billed → 3–5× more expensive
- Haiku ≈ 10× cheaper than Sonnet for same token count
- `MAX_THINKING_TOKENS=8000` caps runaway thinking cost on simple tasks
```

### 3d. Create skill : `fetch-not-read`

**Path :** `.claude/skills/fetch-not-read/SKILL.md`

```markdown
---
name: fetch-not-read
description: Use targeted Bash commands instead of full file reads to minimize context tokens.
  Triggers: before reading any file >100 lines, "show me the code", "read this file",
  "what does X do".
---

# Fetch-not-read Pattern

## Core principle
`Read(large_file.py)` dumps EVERYTHING into context.
Bash commands let you extract **exactly** what you need.

## Replacement patterns

### Structure overview (instead of reading whole file)
```bash
grep -n "^def \|^class " file.py          # All functions/classes with line numbers
grep -n "^export\|^const\|^function" file.ts
```

### Extract one function
```bash
sed -n '/^def target_function/,/^def /p' file.py | head -60
```

### Imports only
```bash
head -30 file.py
```

### Search for specific logic
```bash
grep -n -A10 "keyword" file.py
grep -rn "function_name" --include="*.py" | head -20
```

### Large test output (never cat)
```bash
grep -E "FAILED|ERROR|passed [0-9]+" pytest_output.txt | tail -30
grep -A5 "FAILED" pytest_output.txt
```

### Directory survey (never read all files)
```bash
find . -name "*.py" | head -20
grep -r "function_name" --include="*.py" -l   # Files containing it
```

## Token budget for file reads
| File size | Strategy |
|-----------|----------|
| < 50 lines | OK to Read directly |
| 50–200 lines | Read only if full context needed |
| > 200 lines | Use Bash extraction ALWAYS |
| Entire directory | NEVER read all — use grep/find |
```

---

## PHASE 4 — SETTINGS OPTIMIZATION

### 4a. Update `~/.claude/settings.json` (global defaults)

Read the current file, then merge these optimizations (**do not overwrite — merge**):

```bash
python3 << 'EOF'
import json, os

path = os.path.expanduser("~/.claude/settings.json")
try:
    with open(path) as f:
        settings = json.load(f)
except FileNotFoundError:
    settings = {}

# Merge token-saving defaults
settings.setdefault("model", "claude-sonnet-4-6")
settings.setdefault("effortLevel", "medium")
settings.setdefault("env", {})
settings["env"]["MAX_THINKING_TOKENS"] = "8000"

settings.setdefault("permissions", {})
settings["permissions"].setdefault("deny", [])
deny_rules = [
    "Read(./.env)", "Read(./.env.*)",
    "Read(./secrets/**)", "Read(./.git/objects/**)"
]
for rule in deny_rules:
    if rule not in settings["permissions"]["deny"]:
        settings["permissions"]["deny"].append(rule)

with open(path, "w") as f:
    json.dump(settings, f, indent=2)

print("~/.claude/settings.json updated")
print(f"   model: {settings['model']}")
print(f"   effortLevel: {settings['effortLevel']}")
print(f"   MAX_THINKING_TOKENS: {settings['env']['MAX_THINKING_TOKENS']}")
EOF
```

### 4b. Create `.claude/hooks/guard-large-read.sh` (large file interceptor)

This hook **blocks accidental reads of files > 300 lines** and suggests Bash alternatives.  
Estimated savings: **50,000+ tokens per long session**.

> **Implementation :** External script (not inline JSON) — aligned with the [playbook §6](./claude-code-best-practice-playbook.md#6-hooks--deterministic-guardrails). Easier to test independently with `echo '{...}' | bash .claude/hooks/guard-large-read.sh`.

```bash
mkdir -p .claude/hooks

cat > .claude/hooks/guard-large-read.sh << 'HOOKEOF'
#!/bin/bash
# Block Read tool on files > 300 lines — enforce grep/sed extraction
set -euo pipefail

INPUT=$(cat)
FILE=$(echo "$INPUT" | python3 -c "
import json, sys
print(json.load(sys.stdin).get('tool_input', {}).get('file_path', ''))
" 2>/dev/null || echo "")

# Early exit: no file path or file doesn't exist
if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
  exit 0
fi

LINES=$(wc -l < "$FILE" 2>/dev/null || echo 0)
if [ "$LINES" -gt 300 ]; then
  python3 -c "
import json
print(json.dumps({
  'decision': 'block',
  'reason': (
    '$FILE has $LINES lines (~' + str($LINES * 5) + ' tokens). '
    'Use targeted extraction: '
    'grep -n \\"pattern\\" $FILE | head -20 '
    'or: sed -n \\"/^def target/,/^def /p\\" $FILE | head -50'
  )
}))
"
  exit 0
fi

exit 0
HOOKEOF

chmod +x .claude/hooks/guard-large-read.sh
echo "guard-large-read.sh created and made executable"
```

Then reference it in `.claude/settings.json` hooks section:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-large-read.sh"
          }
        ]
      }
    ]
  }
}
```

---

## PHASE 5 — SUBAGENT FOR CODEBASE EXPLORATION

Create a Haiku-powered research subagent (~10x cheaper than Sonnet) that explores the codebase in an isolated context and returns compact summaries.

> **Note :** The full subagent specification is in the [playbook §8](./claude-code-best-practice-playbook.md#8-subagents--isolated-context-delegation). The command below creates the file if it doesn't already exist.

```bash
mkdir -p .claude/agents

# Only create if not already present (don't overwrite customized versions)
if [ ! -f .claude/agents/code-explorer.md ]; then
cat > .claude/agents/code-explorer.md << 'AGENTEOF'
---
name: code-explorer
description: >
  Explore codebase structure and find relevant files/patterns.
  Use for: understanding how a system works, finding where X is implemented,
  mapping dependencies, surveying a module before editing it.
  Returns compact summary — keeps main context clean.
model: haiku
effort: low
maxTurns: 10
tools: Read, Bash, Glob, Grep
---

You are a fast, efficient code navigator. Explore, then summarize compactly.

## Exploration rules
- Use `grep -n "pattern" file` instead of reading entire files
- Use `grep -rn "symbol" src/ -l` to find files before reading them
- Stop when you have enough to answer — do not explore exhaustively
- Never read files > 200 lines in full without justification

## Response format (ALWAYS)
Return a JSON block:
```json
{
  "relevant_files": ["src/auth/middleware.ts:15-45"],
  "key_findings": ["JWT validation happens at line 23", "No refresh token logic found"],
  "recommended_approach": "Add refresh endpoint in src/auth/ alongside existing middleware",
  "files_read": 3,
  "grep_calls": 5
}
```
AGENTEOF
echo "code-explorer.md created"
else
echo "code-explorer.md already exists — skipped"
fi
```

---

## PHASE 6 — VERIFICATION & FINAL REPORT

Run the full verification suite and generate the savings report:

```bash
python3 << 'EOF'
import os, glob

def tokens(path):
    try:
        with open(os.path.expanduser(path)) as f:
            return len(f.read()) // 4
    except:
        return 0

def lines(path):
    try:
        with open(os.path.expanduser(path)) as f:
            return sum(1 for _ in f)
    except:
        return 0

print("=" * 60)
print("CLAUDE CODE TOKEN OPTIMIZATION — FINAL REPORT")
print("=" * 60)

g_tokens = tokens("~/.claude/CLAUDE.md")
p_tokens = tokens(".claude/CLAUDE.md")
g_lines  = lines("~/.claude/CLAUDE.md")
p_lines  = lines(".claude/CLAUDE.md")

print(f"\nCLAUDE.md files (loaded every session)")
print(f"  Global  : {g_lines} lines — ~{g_tokens} tokens")
print(f"  Project : {p_lines} lines — ~{p_tokens} tokens")
print(f"  Total startup context: ~{g_tokens + p_tokens} tokens")

skill_files = glob.glob(".claude/skills/*/SKILL.md")
skill_tokens = sum(tokens(f) for f in skill_files)
print(f"\nSkills (on-demand — zero startup cost)")
for f in skill_files:
    t = tokens(f)
    print(f"  {os.path.basename(os.path.dirname(f))}: ~{t} tokens")
print(f"  Total skill content: ~{skill_tokens} tokens (loaded only when invoked)")

agent_files = glob.glob(".claude/agents/*.md")
print(f"\nSubagents configured: {len(agent_files)}")
for f in agent_files:
    print(f"  {os.path.basename(f)}")

settings_path = os.path.expanduser("~/.claude/settings.json")
try:
    import json
    with open(settings_path) as f:
        s = json.load(f)
    print(f"\nSettings")
    print(f"  model       : {s.get('model', 'not set')}")
    print(f"  effortLevel : {s.get('effortLevel', 'not set')}")
    print(f"  MAX_THINKING: {s.get('env', {}).get('MAX_THINKING_TOKENS', 'not set')}")
except:
    print("\nCould not read settings.json")

print(f"\nOPTIMIZATION COMPLETE")
print(f"   Startup context      : ~{g_tokens + p_tokens} tokens (target ≤ 800)")
print(f"   Skills offloaded     : ~{skill_tokens} tokens (zero cost until invoked)")
print(f"   Large file hook      : active (blocks files > 300 lines)")
print(f"   Thinking cap         : MAX_THINKING_TOKENS=8000")
print(f"   Cheap exploration    : haiku subagent at ~10x lower cost")
EOF
```

Produce a final savings summary. Cross-reference with the full [Token Optimization Table in the playbook §14](./claude-code-best-practice-playbook.md#14-token-optimization-table) for the complete list.

| Optimization lever | Mechanism | Estimated savings |
|---|---|---|
| CLAUDE.md < 200 lines | Removed workflow bloat | 2,000-10,000 tokens/session |
| Skills on-demand | Zero cost at startup | 1,000-5,000 tokens/session |
| Large file hook (>300 lines) | PreToolUse external script | 10,000-50,000 tokens/session |
| `MAX_THINKING_TOKENS=8000` | Cap thinking budget | 30-50% thinking token reduction |
| `effortLevel: medium` | No default deep reasoning | Baseline efficiency |
| Haiku subagent exploration | 10x cheaper model for research | 80-90% on exploration calls |
| `/btw` for quick lookups | Never enters conversation history | 500-2,000 tokens/question |
| `/clear` between tasks | Eliminate stale context | Variable, often largest gain |

---

## References

- [Manage costs — Claude Code Docs](https://code.claude.com/docs/en/costs)
- [Best practices — Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- [Memory & CLAUDE.md](https://code.claude.com/docs/en/memory)
- [Model configuration & effort levels](https://code.claude.com/docs/en/model-config)

---

## Author

**Papa Sega WADE** — AI Research Engineer
Bridging software engineering and AI research — building tools, workflows, and systems that make LLM-assisted development practical at scale.

[papasegawade.com](https://papasegawade.com/) · [LinkedIn](https://www.linkedin.com/in/papa-s%C3%A9ga-wade-phd-a5727513a)

---
