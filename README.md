# Claude Prompt Optimizer Agent

A Claude Code agent that scores and revises LLM prompts against a research-backed checklist. Based on 2025-2026 findings from AGENTIF (NeurIPS 2025), ACL 2025 self-correction research, and HuggingFace LLM-as-judge benchmarks.

## The Problem

Most LLM prompts are written by feel. Research shows this leads to predictable failures:

- **58.5%** compliance on real multi-constraint instructions (down from 87% on simple benchmarks) — AGENTIF, NeurIPS 2025
- **58.8%** of correct answers flipped wrong by naive "check your work" validation prompts — ACL 2025
- **~29%** sycophancy reduction achievable through prompt structure alone — no fine-tuning needed

## What This Agent Does

When invoked, the prompt-optimizer agent:

1. Reads `PROMPT_BEST_PRACTICES.md` (bundled)
2. Scores your prompt against a **10-item checklist**
3. Returns a **revised version** with every violation fixed and annotated

### The 10-Item Checklist

| # | Item | What it checks |
|---|---|---|
| 1 | Tagged blocks | Distinct sections in XML-style tags |
| 2 | Numbered directives | All instructions numbered for traceability |
| 3 | Length | Under ~1,500 words per call |
| 4 | Gate examples | PASS and FAIL examples for each criterion |
| 5 | Machine-parseable output | Every verdict extractable with regex |
| 6 | Skeptical role | Critical evaluator, not helpful assistant |
| 7 | Do-instead-of-don't | Prohibitions paired with alternatives |
| 8 | Validation model | Same-model validation uses gates + "Wait" + recency fix |
| 9 | Original task in validation | Validation includes original task + end reminder |
| 10 | One criterion per call | Each criterion assessed separately |

Items 8-10 only apply to validation/second-pass prompts.

## Installation

### As a Claude Code Plugin (Recommended)

```bash
# From the Claude Code CLI
/plugin marketplace add dlxmax/claude-prompt-optimizer
/plugin install claude-prompt-optimizer
/reload-plugins
```

### Manual Installation

Copy the `agents/` folder and `PROMPT_BEST_PRACTICES.md` into your Claude Code config:

```bash
cp agents/prompt-optimizer.md ~/.claude/agents/
cp PROMPT_BEST_PRACTICES.md ~/.claude/
```

### Auto-Invocation (Optional)

Add this line to `~/.claude/rules/agents.md` under "Automatic Agent Invocation":

```
6. Writing or revising an LLM prompt → **prompt-optimizer**
```

This makes Claude invoke the optimizer automatically whenever prompt work comes up.

## Usage

The agent triggers automatically when you write or revise LLM prompts (if auto-invocation is configured), or you can reference it explicitly:

```
"Run the prompt-optimizer agent on this grading prompt."
"Score my system prompt against the checklist."
"Optimize this validation prompt for better compliance."
```

### Example Output

```
## Checklist Score: 6/10

[x] Tagged blocks — sections wrapped in <role>, <instructions>, <output_format>
[x] Numbered directives — 5 directives numbered
[ ] Length — 2,100 words, exceeds 1,500 threshold
[x] Gate examples — PASS/FAIL examples for 3 criteria
[ ] Machine-parseable output — no regex-extractable verdict format
[x] Skeptical role — "rigorous evaluator" framing
[ ] Do-instead-of-don't — 2 bare prohibitions without alternatives
[N/A] Validation model
[N/A] Original task in validation
[ ] One criterion per call — 3 criteria bundled in one prompt

## Key Changes
- Split combined criteria into separate evaluation blocks (item 10)
- Added VERDICT format with regex pattern (item 5)
- Paired "do not" instructions with "instead" alternatives (item 7)
- Flagged length for decomposition (item 3)

## Revised Prompt
[full revised prompt text...]
```

## Included Files

| File | Purpose |
|---|---|
| `agents/prompt-optimizer.md` | The Claude Code agent definition |
| `PROMPT_BEST_PRACTICES.md` | Standalone best practices guide (6 sections + checklist) |
| `PROMPT_RESEARCH.md` | Full research archive with 30+ sources (2024-2026) |

## Key Research Sources

- [AGENTIF](https://arxiv.org/abs/2505.16944) — NeurIPS 2025 Spotlight on agentic instruction following
- [ReasonIF](https://arxiv.org/abs/2510.15211) — Reasoning trace compliance benchmark
- [Self-Correction Blind Spot](https://arxiv.org/abs/2507.02778) — The "Wait" prefix discovery
- [Dark Side of Self-Correction](https://aclanthology.org/2025.acl-long.1314/) — ACL 2025, recency bias fix
- [HuggingFace LLM-as-Judge](https://huggingface.co/learn/cookbook/en/llm_judge) — Additive scoring template
- [Anthropic Claude Prompting Guide](https://docs.anthropic.com) — XML tags, document-first ordering

## License

MIT
