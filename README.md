# Claude Prompt Optimizer Agent

A Claude Code agent that scores and revises LLM prompts against a research-backed checklist, refreshed April 2026 for current frontier models. The goal is prompts the model actually executes instead of silently skipping over directives.

## The Problem

Most LLM prompts are written by feel. Frontier models in April 2026 do not refuse tasks, they silently omit them. The research shows where:

- **Frontier models still drop 25 to 40 percent of multi-constraint directives** on novel out-of-domain instructions. Qwen3.6 Plus scores 75.8 percent on IFBench; Claude Opus 4.5 scores 58 percent. Structural prompt design closes the gap. (IFBench 2026)
- **Reasoning quality degrades around 3,000 tokens** even on models with 256K to 1M context windows. Focused prompts beat long ones regardless of available context. (Prompt-bloat study, MLOps Community 2026)
- **58.8 percent of initially correct answers get flipped wrong** by naive "check your work" validation prompts. (ACL 2025)
- **One-shot often beats few-shot** for LLM-as-judge tasks. The old "3 to 5 diverse examples" rule is retired; 1 to 3 calibrated PASS+FAIL pairs per criterion is current. (Confident AI 2026, Label Your Data 2026)
- **GPT-4 reaches 91.7 percent zero-shot** on native-language identification when the prompt names the linguistic features to attend to. Linguistic analysis prompts need their own playbook. (Lotfi et al.)
- **~29 percent sycophancy reduction** is achievable through prompt structure alone, no fine-tuning needed. (sparkco.ai)
- **All frontier judges are unreliable on a single pass** ("rating roulette") — high-stakes judge calls need N>=3 sampling. On Gemini 2.5 Pro and Gemini 3.x, T=0 + seed does not give reproducible output; on Gemini 3.x, T=0 is actively discouraged. Multi-model consensus beats single-model tuning. (Rating Roulette EMNLP 2025; Sage benchmark Dec 2025; Google AI Forum Jan 2026)

## What This Agent Does

When invoked, the prompt-optimizer agent:

1. Reads `PROMPT_BEST_PRACTICES.md` (bundled)
2. Scores your prompt against a **12-item checklist**
3. Returns a **revised version** with every violation fixed and annotated

### The 12-Item Checklist

| # | Item | What it checks |
|---|---|---|
| 1 | Tagged blocks | Distinct sections in XML-style tags |
| 2 | Numbered directives | All instructions numbered for traceability |
| 3 | Length and placement | Focused under ~3K tokens, critical directives at start AND end, decomposed if multi-stage |
| 4 | Gate examples, calibrated count | 1 to 3 PASS+FAIL pairs per criterion, balanced ordering |
| 5 | Machine-parseable output | Every verdict extractable with regex |
| 6 | Skeptical role | Critical evaluator, not helpful assistant |
| 7 | Do-instead-of-don't | Prohibitions paired with alternatives |
| 8 | Validation model | Same-model validation uses gates + "Wait" + recency fix |
| 9 | Original task in validation | Validation includes original task + end reminder |
| 10 | One criterion per call (high-stakes) | High-stakes scoring isolates each criterion; low-stakes may bundle up to 3 |
| 11 | Linguistic-analysis path | If the prompt evaluates properties of writing itself: enumerate features, reason before verdict, cite evidence |
| 12 | Sampling and determinism for judges | High-stakes judges use N>=3 sampling + majority vote; do not rely on T=0 + seed (broken on Gemini 2.5/3.x); prefer multi-model consensus when stakes warrant |

Items 8 to 10 only apply to validation or second-pass prompts. Item 11 only applies to linguistic-analysis prompts (style, register, L1 transfer, authorship, human-vs-AI stylometry, genre fit). Item 12 only applies to high-stakes judge prompts.

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
## Checklist Score: 6/12

[x] Tagged blocks: sections wrapped in <role>, <instructions>, <output_format>
[x] Numbered directives: 5 directives numbered
[ ] Length and placement: 4,200 tokens of bloat, critical directive buried in the middle band
[ ] Gate examples, calibrated count: 5 diverse examples (older 3-5 pattern); should be 1 to 3 PASS+FAIL pairs
[ ] Machine-parseable output: no regex-extractable verdict format
[x] Skeptical role: "rigorous evaluator" framing
[ ] Do-instead-of-don't: 2 bare prohibitions without alternatives
[N/A] Validation model
[N/A] Original task in validation
[ ] One criterion per call: 3 criteria bundled in one high-stakes prompt
[N/A] Linguistic-analysis path: prompt evaluates content, not writing properties
[N/A] Sampling and determinism for judges: this prompt is generation, not high-stakes judging

## Key Changes
- Stripped ~1,500 tokens of non-load-bearing background (item 3)
- Moved the governing directive to both the start and the end of the prompt (item 3)
- Reduced gate examples from 5 to 2 calibrated PASS+FAIL pairs (item 4)
- Split combined criteria into separate evaluation calls (item 10)
- Added VERDICT format with regex pattern (item 5)
- Paired "do not" instructions with "instead" alternatives (item 7)

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

**2026 refresh:**
- [IFBench leaderboard, April 2026](https://benchlm.ai/benchmarks/ifBench): current frontier instruction-following scores
- [LLMLingua-2, NAACL 2025](https://llmlingua.com/llmlingua2.html): task-agnostic prompt compression, 3x to 6x
- [Prompt-bloat study, MLOps Community 2026](https://mlops.community/the-impact-of-prompt-bloat-on-llm-output-quality/): the ~3K token degradation threshold
- [Label Your Data LLM-as-judge 2026 guide](https://labelyourdata.com/articles/llm-as-a-judge): few-shot instability and one-shot dominance
- [Confident AI LLM-as-judge overview](https://www.confident-ai.com/blog/why-llm-as-a-judge-is-the-best-llm-evaluation-method): current best practices
- [Native Language Identification with LLMs (Lotfi et al.)](https://arxiv.org/abs/2312.07819): GPT-4 zero-shot 91.7 percent TOEFL11
- [Multilingual NLI with LLMs, NAACL-SRW 2025](https://aclanthology.org/2025.naacl-srw.19.pdf): feature-aware linguistic analysis prompting
- [Fast-DetectGPT](https://openreview.net/forum?id=Bpcgcr8E8Z) and the [DetectGPT family](https://arxiv.org/abs/2301.11305): feature menu for human-vs-AI stylometry prompts
- [Rating Roulette, EMNLP 2025](https://arxiv.org/pdf/2510.27106): single-pass judges are unreliable across all models; need N>=3 sampling
- [Sage benchmark, Dec 2025](https://arxiv.org/html/2512.16041v1): Gemini 2.5 Pro is strong on easy cases but degrades ~200% on hard pairwise
- [Google AI Forum: Gemini 2.5 Pro non-determinism, Jan 2026](https://discuss.ai.google.dev/t/the-gemini-api-is-exhibiting-non-deterministic-behavior-for-the-gemini-2-5-pro-model-it-is-producing-different-outputs-for-identical-requests-even-when-a-fixed-seed-is-provided-along-with-a-constant-temperature-this-behavior-has-been-reliably-rep/101331): T=0 + seed is not reproducible on Gemini
- [Gemini 2.5 Thinking Model Updates, Feb 2026](https://developers.googleblog.com/en/gemini-2-5-thinking-model-updates/): Gemini 3.x recommends T=1.0 default; thinking_level guidance
- [Judging the Judges, ACL/IJCNLP 2025](https://arxiv.org/html/2406.07791v7): Gemini position bias is incoherent, swap-and-count is less effective

**Still load-bearing:**
- [AGENTIF](https://arxiv.org/abs/2505.16944): NeurIPS 2025 decomposition finding (headline compliance numbers superseded by IFBench 2026)
- [ReasonIF](https://arxiv.org/abs/2510.15211): reasoning trace compliance
- [Self-Correction Blind Spot](https://arxiv.org/abs/2507.02778): the "Wait" prefix discovery
- [Dark Side of Self-Correction](https://aclanthology.org/2025.acl-long.1314/): ACL 2025 recency bias fix
- [HuggingFace LLM-as-Judge cookbook](https://huggingface.co/learn/cookbook/en/llm_judge): scoring templates
- [Anthropic Claude Prompting Guide](https://docs.anthropic.com): XML tags and document-first ordering

## License

MIT
