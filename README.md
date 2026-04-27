# Claude Prompt Optimizer Agent

A Claude Code agent that scores and revises LLM prompts against a research-backed checklist, refreshed April 2026 for current frontier models. The goal is prompts the model actually executes instead of silently skipping over directives.

**Primary workflow:** Use this agent inside Claude Code to optimize prompts that will be sent to Gemini (or any other LLM). The optimizer runs on Claude, which means when it writes a rubric for a Gemini judge prompt, Claude is authoring the rubric and Gemini applies it — a cross-model rubric generation pattern that research shows equals or outperforms same-model self-generation.

## The Problem

Most LLM prompts are written by feel. Frontier models in April 2026 do not refuse tasks — they silently omit them. The research shows where:

- **Frontier models still drop 25–40% of multi-constraint directives** on novel out-of-domain instructions. Qwen3.6 Plus scores 75.8% on IFBench; Claude Opus 4.5 scores 58%. Structural prompt design closes the gap. (IFBench 2026)
- **Reasoning quality degrades around 3,000 tokens** even on models with 256K–1M context windows. Focused prompts beat long ones regardless of available context. (Prompt-bloat study, MLOps Community 2026)
- **58.8% of initially correct answers get flipped wrong** by naive "check your work" validation prompts. (ACL 2025)
- **One-shot often beats few-shot** for LLM-as-judge tasks. The old "3–5 diverse examples" rule is retired; 1–3 calibrated PASS+FAIL pairs per criterion is current. (Confident AI 2026, Label Your Data 2026)
- **GPT-4 reaches 91.7% zero-shot** on native-language identification when the prompt names the linguistic features to attend to. Linguistic analysis prompts need their own playbook. (Lotfi et al.)
- **~29% sycophancy reduction** is achievable through prompt structure alone, no fine-tuning required. (sparkco.ai)
- **A concrete rubric is the single highest-return change for judge prompts** — GPT-4o +17.7 pts on JudgeBench, Llama-405B +7.4 pts, Sage aggregate +16.1% IPI. A ~27-point "Rubric Gap" (self-generated vs. human rubrics) is consistent across Gemini, GPT, and DeepSeek. (Rethinking Rubric Generation 2026; RubricBench 2026; Sage Dec 2025)
- **All frontier judges are unreliable on a single pass** ("rating roulette"). High-stakes judge calls need N>=5 majority vote. On Gemini 2.5 Pro and Gemini 3.x, T=0 + seed is not reproducible; on Gemini 3.x, T=0 is actively discouraged. Debate-style prompts (ChatEval) are actively harmful: -158% worst-case consistency. Multi-model consensus is the strongest deployment lever. (Rating Roulette EMNLP 2025; Sage Dec 2025; Google AI Forum Jan 2026)

## Claude + Gemini Workflow

This optimizer runs on Claude. When it fixes a prompt you will send to Gemini, two things happen that the research validates:

**For judge prompts:** The optimizer writes a concrete rubric directly into the revised prompt. This is cross-model rubric generation (Claude authors, Gemini applies) — shown by the Rethinking Rubric Generation paper (arxiv 2602.05125) to be at least as effective as same-model self-generation, and often better. The embedded `<rubric_generation>` instruction block (asking Gemini to generate its own rubric at inference time) is the right path only when the criterion must adapt per-input at runtime.

**For Gemini non-determinism:** Gemini 2.5 Pro and 3.x have weaker determinism guarantees than Claude or GPT — seed is best-effort, T=0 costs accuracy, and debate-style judge prompts are actively harmful. The checklist catches these patterns and flags the deployment-level fixes (N>=5 sampling, structured output, multi-model consensus). See the Gemini-specific notes in Section 5.8 of `PROMPT_BEST_PRACTICES.md`.

## What This Agent Does

When invoked, the prompt-optimizer agent:

1. Reads `PROMPT_BEST_PRACTICES.md` (bundled)
2. Scores your prompt against a **13-item checklist**
3. Returns a **revised version** with every violation fixed and annotated

### The 13-Item Checklist

| # | Item | What it checks |
|---|---|---|
| 1 | Tagged blocks | Distinct sections in XML-style tags |
| 2 | Numbered directives | All instructions numbered for traceability |
| 3 | Length and placement | Focused under ~3K tokens, critical directives at start AND end, decomposed if multi-stage |
| 4 | Gate examples, calibrated count | 1–3 PASS+FAIL pairs per criterion, balanced ordering |
| 5 | Machine-parseable output | Every verdict extractable with regex |
| 6 | Skeptical role | Critical evaluator, not helpful assistant |
| 7 | Do-instead-of-don't | Prohibitions paired with alternatives |
| 8 | Validation model | Same-model validation uses gates + "Wait" + recency fix |
| 9 | Original task in validation | Validation includes original task + end reminder |
| 10 | One criterion per call (high-stakes) | High-stakes scoring isolates each criterion; low-stakes may bundle up to 3 |
| 11 | Linguistic-analysis path | If the prompt evaluates properties of writing itself: enumerate features, reason before verdict, cite evidence |
| 12 | **Judge prompt: rubric** ★ | Optimizer writes a concrete rubric directly (cross-model generation); or embeds `<rubric_generation>` instruction if criterion is dynamic. Small integer scale (1–4); `<reasoning>` field before verdict. Highest single-change ROI. |
| 13 | Judge prompt: sampling and anti-patterns | N>=5 majority vote; no debate-style (ChatEval) prompts; no T=0+seed on Gemini; multi-model consensus for highest-stakes ranking |

Items 8–10 apply only to validation or second-pass prompts. Item 11 applies only to linguistic-analysis prompts. Item 12 applies to any judge prompt (always check). Item 13 applies to high-stakes judge deployments.

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
"Optimize this Gemini judge prompt for the essay evaluation pipeline."
```

### Example Output

```
## Checklist Score: 6/13

[x] Tagged blocks: sections wrapped in <role>, <instructions>, <output_format>
[x] Numbered directives: 5 directives numbered
[ ] Length and placement: 4,200 tokens; critical directive buried in the middle
[ ] Gate examples, calibrated count: 5 diverse examples (older 3-5 pattern); should be 1-3 PASS+FAIL pairs
[ ] Machine-parseable output: no regex-extractable verdict format
[x] Skeptical role: "rigorous evaluator" framing
[ ] Do-instead-of-don't: 2 bare prohibitions without alternatives
[N/A] Validation model: not a second-pass prompt
[N/A] Original task in validation: not a second-pass prompt
[ ] One criterion per call: 3 criteria bundled in one high-stakes prompt
[N/A] Linguistic-analysis path: evaluates content, not writing properties
[ ] Judge prompt — rubric: no rubric present; will write concrete criteria for each score level
[ ] Judge prompt — sampling: single-pass design; N>=5 needed; Gemini T=0+seed flagged as unreliable

## Key Changes
- Stripped ~1,500 tokens of non-load-bearing background (item 3)
- Moved governing directive to both start and end (item 3)
- Reduced gate examples from 5 to 2 calibrated PASS+FAIL pairs (item 4)
- Split combined criteria into 3 separate evaluation calls (item 10)
- Added VERDICT format with regex pattern (item 5)
- Paired prohibitions with alternatives (item 7)
- Wrote rubric with observable 1-4 criteria directly into the prompt (item 12)
- Added note: run N=5 with majority vote; avoid T=0+seed on Gemini (item 13)

## Revised Prompt
[full revised prompt text...]
```

## Included Files

| File | Purpose |
|---|---|
| `agents/prompt-optimizer.md` | The Claude Code agent definition |
| `PROMPT_BEST_PRACTICES.md` | Best practices guide (7 sections + 13-item checklist) |
| `PROMPT_RESEARCH.md` | Full research archive with 35+ sources (2024–2026) |

## Key Research Sources

**2026 refresh:**
- [IFBench leaderboard, April 2026](https://benchlm.ai/benchmarks/ifBench): current frontier instruction-following scores
- [Rethinking Rubric Generation (RRD), arxiv 2602.05125](https://arxiv.org/abs/2602.05125): GPT-4o +17.7 pts, Llama-405B +7.4 pts from rubric design; cross-model generation validated
- [RubricBench, arxiv 2603.01562](https://arxiv.org/abs/2603.01562): ~27-pt Rubric Gap is equal across Gemini, GPT, DeepSeek — universal bottleneck
- [Same Input, Different Scores, arxiv 2603.04417](https://arxiv.org/abs/2603.04417): Gemini shows highest single-model variance among major families
- [LLMLingua-2, NAACL 2025](https://llmlingua.com/llmlingua2.html): task-agnostic prompt compression, 3x to 6x
- [Prompt-bloat study, MLOps Community 2026](https://mlops.community/the-impact-of-prompt-bloat-on-llm-output-quality/): the ~3K token degradation threshold
- [Label Your Data LLM-as-judge 2026](https://labelyourdata.com/articles/llm-as-a-judge): few-shot instability and one-shot dominance
- [Native Language Identification with LLMs (Lotfi et al.)](https://arxiv.org/abs/2312.07819): GPT-4 zero-shot 91.7% TOEFL11
- [Rating Roulette, EMNLP 2025](https://arxiv.org/pdf/2510.27106): single-pass judges unreliable; N>=5 needed
- [Sage benchmark, Dec 2025](https://arxiv.org/html/2512.16041v1): rubric generation +16.1% IPI; debate prompts -158%; Gemini degrades 200% on hard cases
- [Google AI Forum: Gemini 2.5 Pro non-determinism, Jan 2026](https://discuss.ai.google.dev/t/the-gemini-api-is-exhibiting-non-deterministic-behavior-for-the-gemini-2-5-pro-model-it-is-producing-different-outputs-for-identical-requests-even-when-a-fixed-seed-is-provided-along-with-a-constant-temperature-this-behavior-has-been-reliably-rep/101331): T=0 + seed is not reproducible; seed is best-effort
- [Gemini 3.x thinking model updates, Feb 2026](https://developers.googleblog.com/en/gemini-2-5-thinking-model-updates/): T=1.0 recommended default on Gemini 3.x
- [Judging the Judges, ACL/IJCNLP 2025](https://arxiv.org/html/2406.07791v7): Gemini position bias is incoherent; swap-and-count less effective

**Still load-bearing:**
- [AGENTIF](https://arxiv.org/abs/2505.16944): NeurIPS 2025 decomposition finding (headline numbers superseded by IFBench 2026)
- [Self-Correction Blind Spot](https://arxiv.org/abs/2507.02778): the "Wait" prefix discovery
- [Dark Side of Self-Correction](https://aclanthology.org/2025.acl-long.1314/): ACL 2025 recency bias fix
- [HuggingFace LLM-as-Judge cookbook](https://huggingface.co/learn/cookbook/en/llm_judge): 1-4 scale; evaluation field before verdict; 0.563→0.843 correlation improvement
- [Anthropic Claude Prompting Guide](https://docs.anthropic.com): XML tags and document-first ordering

## License

MIT
