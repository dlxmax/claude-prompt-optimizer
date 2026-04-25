# Prompt Engineering Research Archive

Compiled March 2026, refreshed April 2026 with IFBench, LLMLingua-2, 2026 few-shot findings, linguistic-analysis literature, and prompt-bloat results. Older entries that have been partially superseded are tagged in place. Indexed by topic for fast recall in future prompt-related tasks.

---

## Topic 1: Sycophancy / Rubber-Stamping

### Root Cause
Sycophancy is a byproduct of RLHF training. Models learn that agreeable, validating responses earn higher human satisfaction scores during feedback collection. This is the intended optimization target for chat models — not an accident. The result is a systematic bias toward agreement and flattery that persists even in evaluation tasks.

**Production incident (April 2025):** OpenAI rolled back a ChatGPT (GPT-4o) update after it became excessively sycophantic — generating overly flattering responses and validating bad decisions. This confirmed sycophancy as an active production risk, not a theoretical concern.

### Quantified Reduction from Prompt Design

| Technique | Sycophancy reduction | Source |
|---|---|---|
| Skeptical role assignment | Baseline improvement | GovTech Singapore, Jan 2026 |
| Evidence-first question framing | Additive | sparkco.ai, 2025 |
| Forced counterargument | Additive | sparkco.ai, 2025 |
| Explicit anti-flattery instruction | Additive | GovTech Singapore, Jan 2026 |
| All four combined | ~29% total reduction | sparkco.ai, 2025 |
| RLHF-level mitigations (fine-tuning) | ~20% additional | sparkco.ai, 2025 |

### Training-Level Mitigations (for reference)

- **Sparse Activation Fusion (SAF):** Reduces sycophancy from 63% to 39% by subtracting user-induced bias in the feature space. Does not require labeled sycophancy data.
- **Structured Sycophancy Mitigation (SSM):** Uses causal models to disentangle sycophantic embeddings. Does not require explicit user-preference prompts.

### Sources

- GovTech Singapore sycophancy mini-survey, Jan 2026: medium.com/dsaid-govtech/yes-youre-absolutely-right-right-a-mini-survey-on-llm-sycophancy-02a9a8b538cf
- sparkco.ai — 69% improvement strategies: sparkco.ai/blog/reducing-llm-sycophancy-69-improvement-strategies
- MLOps lessons from ChatGPT sycophancy rollback: leehanchung.github.io/blogs/2025/04/30/ai-ml-llm-ops/
- SAF paper: openreview.net/pdf?id=BCS7HHInC2
- SSM/causally motivated mitigation, ICLR 2025: proceedings.iclr.cc/paper_files/paper/2025/file/a52b0d191b619477cc798d544f4f0e4b-Paper-Conference.pdf
- CONSENSAGENT (multi-agent anti-sycophancy), ACL 2025: aclanthology.org/2025.findings-acl.1141/

---

## Topic 2: Directive Compliance Benchmarks

> **Status note (April 2026):** The AGENTIF headline numbers below were derived on GPT-4o-class models and are partially superseded by IFBench 2026 (see Topic 6). Keep AGENTIF as the origin story for the "decompose long instructions" finding, which has held up. Do not lead with the 58.5 percent stat in new writing.

### AGENTIF (NeurIPS 2025 Spotlight)

**Authors:** Tsinghua KEG lab
**Paper:** arxiv.org/abs/2505.16944
**GitHub:** github.com/THU-KEG/AgentIF

First benchmark for agentic instruction following using real-world, long, multi-constraint instructions.

**Key results:**
- GPT-4o: **87.0%** on IFEval (simple, synthetic) → **58.5%** on AGENTIF (real-world, long)
- Best model achieves only **27.2% Instruction Success Rate** on full multi-constraint instructions
- Hardest constraint types: condition constraints (if-then triggers), tool constraints (which tools to use/avoid), format constraints on long specifications
- When instructions exceed 6,000 words, instruction success rates approach zero

**Top recommendation from authors:** Decompose long instructions into shorter sub-tasks. This single change has the largest empirical effect on compliance.

### ReasonIF (October 2025)

**Paper:** arxiv.org/abs/2510.15211

Benchmark for instruction following in reasoning traces (not just final outputs).

**Key result:** Fewer than **25% of reasoning traces comply** with given instructions across open-source models, even when the final outputs look compliant. Compliance degrades with task difficulty (r=0.86 correlation).

**Implication:** For high-stakes agents, verify intermediate reasoning steps, not just final outputs. Available via extended thinking / thinking tokens APIs.

### IFEval++ (2025)

**Paper:** arxiv.org/html/2512.14754v1

Reliability study of the IFEval benchmark.

**Key result:** Performance drops **61.8%** with nuanced prompt modifications vs. the original benchmark phrasing. Models are more sensitive to exact wording than benchmark scores suggest.

**Implication:** Exact wording of directives matters more than benchmark performance implies. Test prompt wording carefully.

---

## Topic 3: Self-Correction Reliability

### Core Finding

> "Current LLMs cannot improve their reasoning performance through intrinsic self-correction."
> — ICLR 2024, "Large Language Models Cannot Self-Correct Reasoning Yet"
> openreview.net/forum?id=IkmD3fKBPQ

Intrinsic self-correction = same model, no external signal, freeform critique prompt.

### Quantified Failure Rates

**ACL 2025 — "Understanding the Dark Side of LLMs' Intrinsic Self-Correction"**
aclanthology.org/2025.acl-long.1314/

- GPT-3.5-turbo changes answers more than **6 times in 10 correction rounds** for 80%+ of samples
- Models overturn **58.8% of initially correct answers** during self-correction
- Three failure mechanisms identified:
  1. **Recency bias:** Model focuses on the validation prompt rather than the original task
  2. **Answer wavering:** Oscillation without convergence across rounds
  3. **Overthinking:** Excessive reasoning on already-correct answers
- **Fix for recency bias:** Append the original task at the END of the validation prompt — reduces correct-answer flips by 5–11%

**Self-Correction Bench (2025)**
arxiv.org/abs/2507.02778

- Average **64.5% blind spot rate** across 14 models: LLMs reliably correct identical errors in external text but fail to correct them in their own output
- Prepending a minimal **"Wait"** prompt reduces blind spots by **89.3%** — activates dormant self-correction capability already present in the model

### When Self-Correction Works

| Condition | Result |
|---|---|
| External feedback / oracle signal | Reliable improvement |
| Verifiable ground truth (code execution, math checker) | Effective |
| Different/stronger model as judge | More reliable than self-judging |
| Structured gate scoring (named criteria, examples in prompt) | Reliable — eliminates ambiguity that causes wavering |
| Fine-tuned correction (domain-specific examples) | Strong improvement |
| Same model, freeform "check your work" | Often degrades accuracy |
| Reasoning models (o1, DeepSeek-R1 style) | Already embed error-checking; second pass wastes tokens |

### When It Fails

- Simple factual questions: prompt bias causes unnecessary answer flips
- Long-context tasks: cognitive overload causes model to forget original constraints
- Multiple rounds: oscillation increases, no convergence

### TACL Survey

"When Can LLMs Actually Correct Their Own Mistakes? A Critical Survey"
direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00713/125177/

---

## Topic 4: LLM-as-Judge Patterns

### Scoring Formats

| Format | When to use |
|---|---|
| Pointwise/absolute (single response vs. rubric) | Default for evaluation pipelines |
| Pairwise comparison (A vs. B) | Preference ranking; more reliable but requires (A,B) and (B,A) both |
| Reference-guided (vs. gold standard) | Only when gold standard answers exist |

### Proven Template (HuggingFace, 2025)

Achieves **0.843 Pearson correlation** with human ratings.

```
You will be given a user_question and system_answer couple.
Your task is to provide a 'total rating' scoring how well the system_answer
answers the user concerns expressed in the user_question.

Scale (1–4):
1: Terrible — completely irrelevant or very partial
2: Mostly unhelpful — misses key aspects
3: Mostly helpful — provides support but could improve
4: Excellent — relevant, direct, detailed, addresses all concerns

Feedback:::
Evaluation: (your rationale for the rating)
Total rating: (your rating, as a number between 1 and 4)

You MUST provide values for 'Evaluation:' and 'Total rating:' in your answer.

Question: {question}
Answer: {answer}
Feedback:::
Evaluation:
```

Source: huggingface.co/learn/cookbook/en/llm_judge

### Bias Types and Mitigations

| Bias | Description | Mitigation |
|---|---|---|
| Position bias | Model prefers A over B when A comes first | Evaluate both (A,B) and (B,A); count only consistent wins |
| Verbosity bias | Longer answers rated higher regardless of quality | State: "Length is not a quality signal" |
| Self-preference bias | Models rate their own output higher when anonymized | Use a different model as judge |
| Sycophancy bias | Model rates highly if told you prefer it | Never reveal preferences before scoring |
| Bandwagon bias | Model agrees with majority if shown prior votes | Never include prior scores in prompt |

Position bias alone causes ~40% inconsistency in GPT-4 pairwise evaluations (IJCNLP 2025):
aclanthology.org/2025.ijcnlp-long.18.pdf

### Additive Checklist Scoring

Decompose vague criteria into binary sub-questions, each worth 1 point. Outperforms holistic scoring by ~30% Pearson correlation.

```
Award 1 point if the answer is relevant to the question.
Award 1 additional point if the answer is factually accurate.
Award 1 further point if the answer is appropriately concise.
```

Source: HuggingFace LLM-as-judge cookbook (see above)

### Advanced: MAJ-Eval (2025)

Multi-agent group debate over candidate outputs. Multiple LLMs independently score, then debate disagreements. Achieves higher alignment with human ratings than single-agent judging.

### Sources

- Evidently AI LLM-as-judge guide: evidentlyai.com/llm-guide/llm-as-a-judge
- Patronus AI LLM-as-judge: patronus.ai/llm-testing/llm-as-a-judge
- Agenta AI LLM-as-judge: agenta.ai/blog/llm-as-a-judge-guide-to-llm-evaluation-best-practices
- HuggingFace cookbook: huggingface.co/learn/cookbook/en/llm_judge
- Justice or Prejudice (bias survey): arxiv.org/html/2410.02736v1
- Self-Preference Bias: arxiv.org/html/2410.21819v2
- Position Bias, IJCNLP 2025: aclanthology.org/2025.ijcnlp-long.18.pdf

---

## Topic 5: First-Pass Prompt Architecture

### XML Tag Separation

Anthropic's official guidance confirms that XML-style tags reduce misinterpretation significantly. Consistent naming conventions:

```
<role>         — Evaluator identity and disposition
<instructions> — Numbered task directives
<context>      — Background information, examples
<input>        — Content to process
<output_format>— Exact format specification with example
<examples>     — Few-shot examples
<documents>    — Multiple documents (nest as <document index="N">)
```

Tags also provide a security benefit — they create trusted instruction boundaries that reduce prompt injection surface.

Source: Anthropic XML tag guidance — docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags

### Document-First Ordering

Placing long context documents before instructions and queries improves response quality by up to 30%.

```
CORRECT: <context>{LONG_DOC}</context> → <instructions> → query
WRONG:   <instructions> → query → <context>{LONG_DOC}</context>
```

Source: Anthropic Claude 4.x prompting guide — docs.anthropic.com

### Prompt Length and Compliance

> **Status note (April 2026):** The 1,500-word threshold below is superseded. See Topic 6 for current guidance.

AGENTIF (NeurIPS 2025) found compliance degrades sharply with prompt length:
- Instructions exceeding ~6,000 words: compliance approaches zero
- Practical threshold for generation prompts: ~1,500 words before decomposing into chained calls (2024 to 2025 models)

### Few-Shot Examples

> **Status note (April 2026):** The "3 to 5 diverse" rule below is superseded. See Topic 8.

Models perform measurably better on example-inferred constraints than specification-only constraints (AGENTIF findings). Original recommendation was 3 to 5 diverse examples. Current guidance: 1 to 3 calibrated PASS+FAIL pairs per criterion (see Topic 8 for full 2026 evidence).

### Explain the "Why"

Models generalize from explanations. Directives with reasons are more robust to edge cases than bare prohibitions.

```
LESS ROBUST: "Never use ellipses."
MORE ROBUST: "Never use ellipses, because the TTS engine does not know how to pronounce them."
```

Source: Anthropic Claude 4.x prompting guide

### Self-Refine (Iterative Refinement)

Self-Refine (Madaan et al., 2023) showed iterative refinement with structured feedback can improve outputs. The critical requirement: feedback must be specific and criterion-based, not generic ("this could be better").

- Self-Refine paper: arxiv.org/abs/2303.17651
- Socratic Self-Refine (SSR): arxiv.org/html/2511.10621v1

### Chain-of-Thought Forcing

CoT improves compliance with multi-step directives by making reasoning visible and checkable:
- Simple: append "Think step by step before answering"
- Structured: use `<thinking>` / `<answer>` tags to separate reasoning from output
- Evaluation gate: "Determine whether [condition] is true. Then, based on that determination, [action]."

---

## Topic 6: Modern Prompt Length, Placement, and Compression (2026 refresh)

This topic replaces the 1,500-word cap in Topic 5.

### Reasoning Degradation Starts Around 3,000 Tokens

Source: MLOps Community, "The Impact of Prompt Bloat on LLM Output Quality" (2026).
URL: mlops.community/the-impact-of-prompt-bloat-on-llm-output-quality/

Key finding: reasoning quality starts to drop around 3,000 tokens even on models that advertise 128K to 1M context windows. Longer prompts do not help and often hurt when the extra tokens are not load-bearing.

**Implication:** The relevant metric for prompt length is *focused tokens*, not the model's maximum context. "Room to grow" is not permission to grow.

### Practical High-Quality Window Is 16K to 32K Tokens

Sources:
- elvex, "Context Length Comparison: Leading AI Models in 2026": elvex.com/blog/context-length-comparison-ai-models-2026
- DevTk.AI, "LLM Context Windows Explained: 4K to 1M Tokens (2026)": devtk.ai/en/blog/llm-context-window-explained/
- Prompt Quorum, "Long Context Local LLMs 2026": promptquorum.com/local-llms/long-context-local-llms

Key finding: models with 128K to 1M advertised windows have a practical high-quality retrieval window of 16K to 32K tokens. A 128K model may reliably answer questions about content in the first 32K and last 16K tokens but miss the 40K to 80K middle band.

### Lost-in-the-Middle Still Active on RoPE Models

Source: "Lost in the Middle: How Language Models Use Long Contexts" (lecture slide set referencing 2023 original).
URL: teapot123.github.io/files/CSE_5610_Fall25/Lecture_12_Long_Context.pdf

Key finding: models trained with rotary positional encodings (Llama, Qwen, Mistral family) retrieve information best when it sits at the beginning or end of the context, worst when it sits in the middle. Some 2026 architectures reduce this effect but do not eliminate it.

**Practical rule:** Place load-bearing directives in the first and last sections of a prompt. If you have one critical instruction, repeat it at the end.

### IFBench 2026 Leaderboard

Source: BenchLM, "IFBench Benchmark 2026": benchlm.ai/benchmarks/ifBench

Key results as of April 7, 2026:
- Qwen3.6 Plus: **75.8%**
- Claude Opus 4.5: **58%**

IFBench tests precise instruction following on 58 verifiable out-of-domain constraints. Unlike IFEval, it measures novel instruction compliance rather than familiar patterns. Frontier models are better than the 2025 AGENTIF numbers but still drop roughly 25 to 40 percent of directives on novel prompts.

### Prompt Compression

Sources:
- LLMLingua-2, NAACL 2025: llmlingua.com/llmlingua2.html
- "Prompt Compression for Large Language Models: A Survey", NAACL 2025: aclanthology.org/2025.naacl-long.368.pdf
- "Prompt Compression in the Wild", arxiv 2604.02985

Key findings:
- LLMLingua-2 uses a BERT-level encoder for task-agnostic token-level compression. 3x to 6x faster than original LLMLingua.
- Core compression techniques (summarization, keyphrase extraction, semantic chunking) achieve 5x to 20x compression while maintaining or improving accuracy.
- Up to 18 percent faster inference and 75 percent lower GPU memory once prompts exceed 5K tokens.
- Cost savings of 70 to 94 percent reported in production settings.

**Order of operations when a prompt is heavy:** focus (strip non-load-bearing context), compress (LLMLingua-2 or summarization), decompose (chain calls). In that order.

---

## Topic 7: Prompts for Linguistic Analysis

"Linguistic analysis" prompts are the class of evaluation prompts where the LLM judges properties of writing itself: native language, register, style, L1 transfer, authorship, genre, human versus AI origin. These tasks have different failure modes from content evaluation and deserve their own playbook.

### Native Language Identification With LLMs

Source: Lotfi, Maladry, Hoste, "Native Language Identification with Large Language Models", arxiv 2312.07819

Key findings:
- **GPT-4 reaches 91.7 percent zero-shot accuracy on the TOEFL11 benchmark** for native-language identification, setting a new state of the art.
- LLMs can justify their predictions by pointing to spelling errors, syntactic patterns, and direct-translation artifacts.
- LLMs are not constrained to a predefined label set, and iterative prompting can correct out-of-class predictions by feeding feedback back and asking for a refined label.

**Implication:** Zero-shot works well when the task's linguistic features are clearly named. Use few-shot only when a specific criterion is ambiguous.

### Multilingual Native Language Identification

Source: "Multilingual Native Language Identification with Large Language Models", NAACL-SRW 2025: aclanthology.org/2025.naacl-srw.19.pdf

Key finding: LLMs handle NLI across multiple target languages when prompted to attend to L1-specific feature categories. Performance is stronger when the prompt enumerates categories than when it asks for a single holistic judgment.

### Native Language Prompting (NatLan)

Source: "Unlocking the Non-Native Language Context Limitation: Native Language Prompting Facilitates Knowledge Elicitation", arxiv 2408.03544

Key finding: decomposing native-language transfer simulation into semantic-transferring and answer-generating steps (handled by two distinct multilingual LLMs) improves non-English reasoning. Mentioned for completeness; less directly relevant to linguistic analysis prompt construction than Lotfi et al.

### PEEM Framework

Source: "PEEM: Prompt Engineering Evaluation Metrics for Interpretable Joint Evaluation of Prompts and Responses", arxiv 2603.10477

Key finding: formal nine-axis rubric for evaluating prompt-response pairs. Prompt-level axes: clarity/structure, linguistic quality, fairness. Response-level axes: accuracy, coherence, relevance, objectivity, clarity, conciseness. Useful as a self-audit checklist for linguistic analysis prompts.

### Linguistic Features Affect Prompt Effectiveness

Source: "A comprehensive taxonomy of prompt engineering techniques for large language models", Frontiers of Computer Science, 2025: link.springer.com/article/10.1007/s11704-025-50058-z

Key finding: morphological, syntactic, and lexico-semantic properties of prompt wording meaningfully change task performance. Exact phrasing matters, especially for linguistic tasks where the model is being asked to reason about those same categories.

### Zero-Shot AI-Generated Text Detection (Feature Menu)

Sources:
- DetectGPT, arxiv 2301.11305
- Fast-DetectGPT, OpenReview Bpcgcr8E8Z
- Implicit Reward Models (IRM) for detection, OpenReview 2VdsYVXLDl
- DetectLLM (log-rank information): github.com/mbzuai-nlp/DetectLLM
- GPT-who (psycholinguistic UID features): referenced in ICTMCG Awesome-Machine-Generated-Text
- ICTMCG Awesome-Machine-Generated-Text (living literature list): github.com/ICTMCG/Awesome-Machine-Generated-Text

Key features that separate machine-generated from human text (useful as a feature menu for linguistic-analysis prompts):
- Log-likelihood curvature (DetectGPT, Fast-DetectGPT)
- Token log-rank distribution (DetectLLM)
- Entropy
- LLM-Deviation statistical signal (Multi-Feature Detection work)
- Uniform Information Density (UID) and other psycholinguistic features (GPT-who)
- Sentence-length burstiness and formulaic transitions

**Implication:** When writing an LLM-as-judge prompt for human-versus-AI stylometry, enumerate these features in the prompt rather than asking for a holistic judgment. The model is more reliable when it knows what to look at.

---

## Topic 8: Few-Shot Calibration for LLM-as-Judge (2026)

This topic replaces the "3 to 5 diverse examples" guidance in Topic 5.

### One-Shot Often Beats Few-Shot

Source: Confident AI, "LLM-as-a-Judge Simply Explained": confident-ai.com/blog/why-llm-as-a-judge-is-the-best-llm-evaluation-method

Key finding: across major models on code evaluation tasks, one-shot outperforms few-shot, and performance declines as more examples are added. The best count is usually the smallest count that conveys the criterion.

### Few-Shot Instability

Source: Label Your Data, "LLM as a Judge: A 2026 Guide to Automated Model Assessment": labelyourdata.com/articles/llm-as-a-judge

Key findings:
- Performance with few-shot prompts is unstable when changing label balance, example order, or number of examples.
- Biased examples propagate directly into the model's judgments. Too many negatives skews negative, a single trailing negative skews negative.
- Few-shot did lift GPT-4 judge consistency from 65.0 to 77.5 percent in one calibrated study, but only when examples reflected the natural distribution of scores.

**Practical rule:** Use 1 to 3 examples per criterion, always pair PASS with FAIL, balance the ordering, and rotate which one comes first across criteria. If a single PASS-FAIL pair conveys the criterion, stop there.

### Sources

- Confident AI (LLM-as-judge overview, 2026): confident-ai.com/blog/why-llm-as-a-judge-is-the-best-llm-evaluation-method
- Label Your Data (LLM-as-judge 2026 guide): labelyourdata.com/articles/llm-as-a-judge
- Evidently AI LLM-as-judge guide: evidentlyai.com/llm-guide/llm-as-a-judge

---

## Topic 9: Gemini 2.5 / 3.x Judge Behavior (2026)

Gemini-specific findings on LLM-as-judge non-determinism. Where Gemini diverges from Claude/GPT-derived best practices, the Gemini findings take precedence for Gemini deployments. The general patterns in Topics 3, 4, and 8 still hold for Claude and GPT judges.

### Determinism Controls Are Weaker Than on Claude/GPT

Source: Google AI Developers Forum, "The Gemini API is Exhibiting Non-Deterministic Behavior for the Gemini-2.5-Pro Model" (Jan 2026):
discuss.ai.google.dev/t/the-gemini-api-is-exhibiting-non-deterministic-behavior-for-the-gemini-2-5-pro-model.../101331

Key findings:
- Gemini 2.5 Pro returns different outputs for identical requests with fixed `seed`, low `temperature`, and fixed `thinking_budget`. Reported example: same JSON-schema request returned an empty array on the first call and `["11"]` on the second.
- Google documents `seed` as best-effort, not guaranteed. Changing models or parameters can vary results even with identical seed.
- **Gemini 3.x docs explicitly recommend `temperature: 1.0` (default).** Setting T below 1.0 can cause looping or degraded output. The standard "set T=0 for reproducible eval" pattern fails on Gemini 3.x.

Implication: Do not rely on T=0 + seed for judge reproducibility on Gemini. Use multi-sample voting or multi-model consensus instead.

### Rating Roulette: Single-Pass Judges Are Unreliable Across All Models

Source: Haldar & Hockenmaier, "Rating Roulette: Self-Inconsistency in LLM-As-A-Judge Frameworks", EMNLP 2025: arxiv.org/pdf/2510.27106

Key finding: All major LLM judges, Gemini included, show low intra-rater reliability. Repeated runs of the same judge prompt on identical input produce inconsistent ratings, in some setups close to random. Affects single-pass judging across families.

Implication: For high-stakes judge calls, sample N>=3 and aggregate by majority or confidence-weighted vote (CISC).

### Gemini 2.5 Pro: Strong on Easy, Collapses on Hard

Source: Feng et al., "Are We on the Right Way to Assessing LLM-as-a-Judge?" (Sage benchmark), arxiv 2512.16041, Dec 2025: arxiv.org/html/2512.16041v1

Key findings:
- On Sage local-consistency (IPI) and global-consistency (TOV) metrics, Gemini 2.5 Pro is among the most consistent judges on the easy split.
- On Sage-Hard (subtle pairwise differences), Gemini 2.5 Pro consistency degrades roughly 200%, matching GPT-5. Even top judges fail ~25% of hard pairwise comparisons.
- Hyperparameter settings (including temperature) significantly affect judge behavior; results are not stable across configurations.

Implication: Gemini judges are acceptable on obvious distinctions but unreliable on nuanced comparisons. Multi-sample or human-verify on fine-grained quality differences.

### Gemini Position Bias Is Incoherent, Not Directional

Source: Shi et al., "Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge", ACL/IJCNLP 2025: arxiv.org/html/2406.07791v7

Key finding: Gemini judges show "rather low mutual agreement and minimal familial property" relative to GPT-4 family and Claude-3-Opus. Position-bias direction is not consistent (sometimes first, sometimes last, no coherent pattern), so the standard A→B and B→A swap-and-count debiasing is less effective than on Claude or GPT.

Implication: Position-swap remains useful as a baseline, but Gemini judges benefit more from multi-sampling and rubric-based scoring than from positional debiasing alone.

### Extended Thinking on Gemini Does Not Stabilize Judge Verdicts

Sources:
- Gemini 3.1 Pro `thinking_level` documentation, Feb 2026: developers.googleblog.com/en/gemini-2-5-thinking-model-updates/
- "How to Use Thinking Mode in Gemini 3 for Complex Reasoning Tasks", Feb 2026: oneuptime.com/blog/post/2026-02-17-how-to-use-thinking-mode-in-gemini-3-for-complex-reasoning-tasks/view

Key findings:
- Gemini 3.1 Pro replaces `thinking_budget` with `thinking_level` (LOW/MEDIUM/HIGH).
- Official guidance reserves HIGH for tasks where extended reasoning directly drives output quality (proofs, complex synthesis), not generic eval.
- Gemini 2.5 Deep Think runs parallel hypotheses, qualitatively different from sequential CoT in Claude/GPT. No published ablation shows this pattern improves judge consistency; it may add variance.

Implication: Do not assume HIGH thinking on Gemini judges improves verdict stability. Validate against a no-thinking baseline on a small dataset before enabling it on a judge prompt.

### Multi-Model Consensus Beats Single-Model Tuning

Source: Practitioner consensus across 2025 and 2026 eval-platform writeups (Vellum, Braintrust, Langfuse, Promptfoo) summarized in Sage benchmark and Rating Roulette discussions.

Key finding: Combining two or more judges from different families (e.g., Gemini Flash + Claude Sonnet, or Gemini + GPT) reduces single-model bias more than any per-model tuning. Family-specific evaluation personalities partially cancel.

Implication: For high-stakes judge calls, multi-model consensus is the strongest single lever, ahead of parameter tuning, swap-and-count, or thinking-level adjustment.

---

## Full Source List

| Source | Type | URL | Date |
|---|---|---|---|
| AGENTIF paper | NeurIPS 2025 Spotlight | arxiv.org/abs/2505.16944 | 2025 |
| AGENTIF GitHub | Code/benchmark | github.com/THU-KEG/AgentIF | 2025 |
| ReasonIF benchmark | Paper | arxiv.org/abs/2510.15211 | Oct 2025 |
| IFEval++ reliability | Paper | arxiv.org/html/2512.14754v1 | 2025 |
| LLMs Cannot Self-Correct | ICLR 2024 | openreview.net/forum?id=IkmD3fKBPQ | 2024 |
| Dark Side of Self-Correction | ACL 2025 | aclanthology.org/2025.acl-long.1314/ | 2025 |
| Self-Correction Bench | arxiv | arxiv.org/abs/2507.02778 | 2025 |
| TACL self-correction survey | TACL | direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00713/125177/ | 2025 |
| CorrectBench | arxiv | arxiv.org/html/2510.16062v1 | 2025 |
| HuggingFace LLM-as-judge cookbook | Guide | huggingface.co/learn/cookbook/en/llm_judge | 2025 |
| Evidently AI LLM-as-judge | Guide | evidentlyai.com/llm-guide/llm-as-a-judge | 2025 |
| Patronus AI LLM-as-judge | Guide | patronus.ai/llm-testing/llm-as-a-judge | 2025 |
| Agenta AI LLM-as-judge | Guide | agenta.ai/blog/llm-as-a-judge-guide-to-llm-evaluation-best-practices | 2025 |
| GovTech Singapore sycophancy survey | Article | medium.com/dsaid-govtech/yes-youre-absolutely-right-right-a-mini-survey-on-llm-sycophancy-02a9a8b538cf | Jan 2026 |
| sparkco.ai sycophancy reduction | Article | sparkco.ai/blog/reducing-llm-sycophancy-69-improvement-strategies | 2025 |
| ChatGPT sycophancy rollback (MLOps) | Post | leehanchung.github.io/blogs/2025/04/30/ai-ml-llm-ops/ | Apr 2025 |
| SAF (Sparse Activation Fusion) | Paper | openreview.net/pdf?id=BCS7HHInC2 | 2025 |
| SSM (Structured Sycophancy Mitigation) | ICLR 2025 | proceedings.iclr.cc/paper_files/paper/2025/file/a52b0d191b619477cc798d544f4f0e4b-Paper-Conference.pdf | 2025 |
| CONSENSAGENT | ACL 2025 | aclanthology.org/2025.findings-acl.1141/ | 2025 |
| Position Bias in LLM-as-judge | IJCNLP 2025 | aclanthology.org/2025.ijcnlp-long.18.pdf | 2025 |
| Justice or Prejudice (bias survey) | arxiv | arxiv.org/html/2410.02736v1 | 2025 |
| Self-Preference Bias | arxiv | arxiv.org/html/2410.21819v2 | 2025 |
| Self-Refine | arxiv | arxiv.org/abs/2303.17651 | 2023 |
| Socratic Self-Refine (SSR) | arxiv | arxiv.org/html/2511.10621v1 | Nov 2025 |
| Constitutional AI | Anthropic | anthropic.com/research/constitutional-ai-harmlessness-from-ai-feedback | 2022 |
| Reflexion | arxiv | arxiv.org/abs/2303.11366 | 2023 |
| Anthropic Claude 4.x prompting guide | Docs | docs.anthropic.com | Mar 2026 |
| Anthropic XML tag guidance | Docs | docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags | 2025 |
| OpenAI evaluation best practices | Docs | platform.openai.com/docs/guides/evaluation-best-practices | 2025 |
| Google Gemini prompting strategies | Docs | ai.google.dev/gemini-api/docs/prompting-strategies | 2025 |
| Gemini 3 prompting guide | Google Cloud | docs.cloud.google.com/vertex-ai/generative-ai/docs/start/gemini-3-prompting-guide | 2025 |
| Lakera prompt engineering guide | Guide | lakera.ai/blog/prompt-engineering-guide | 2026 |
| IFBench leaderboard (2026) | Benchmark | benchlm.ai/benchmarks/ifBench | April 2026 |
| LLMLingua-2 | Paper/site | llmlingua.com/llmlingua2.html | NAACL 2025 |
| Prompt Compression Survey | NAACL 2025 | aclanthology.org/2025.naacl-long.368.pdf | 2025 |
| Prompt Compression in the Wild | arxiv | arxiv.org/abs/2604.02985 | 2026 |
| MLOps Community prompt-bloat study | Post | mlops.community/the-impact-of-prompt-bloat-on-llm-output-quality/ | 2026 |
| elvex context length comparison 2026 | Post | elvex.com/blog/context-length-comparison-ai-models-2026 | 2026 |
| DevTk.AI LLM context windows 2026 | Post | devtk.ai/en/blog/llm-context-window-explained/ | 2026 |
| Prompt Quorum local long-context LLMs | Post | promptquorum.com/local-llms/long-context-local-llms | 2026 |
| NLI with LLMs (Lotfi et al.) | arxiv | arxiv.org/abs/2312.07819 | 2023 (baseline) |
| Multilingual NLI with LLMs | NAACL-SRW 2025 | aclanthology.org/2025.naacl-srw.19.pdf | 2025 |
| Native Language Prompting (NatLan) | arxiv | arxiv.org/abs/2408.03544 | 2024 |
| PEEM framework | arxiv | arxiv.org/html/2603.10477 | 2026 |
| Frontiers of Computer Science prompt taxonomy | Journal | link.springer.com/article/10.1007/s11704-025-50058-z | 2025 |
| DetectGPT | arxiv | arxiv.org/abs/2301.11305 | 2023 (baseline) |
| Fast-DetectGPT | OpenReview | openreview.net/forum?id=Bpcgcr8E8Z | 2024 |
| Implicit Reward Models for detection (IRM) | OpenReview | openreview.net/forum?id=2VdsYVXLDl | 2024 |
| DetectLLM | GitHub | github.com/mbzuai-nlp/DetectLLM | 2024 |
| ICTMCG Machine-Generated Text resources | GitHub | github.com/ICTMCG/Awesome-Machine-Generated-Text | ongoing |
| Confident AI LLM-as-judge 2026 | Guide | confident-ai.com/blog/why-llm-as-a-judge-is-the-best-llm-evaluation-method | 2026 |
| Label Your Data LLM-as-judge 2026 | Guide | labelyourdata.com/articles/llm-as-a-judge | 2026 |
| Gemini API non-determinism (Google Forum) | Forum | discuss.ai.google.dev/t/the-gemini-api-is-exhibiting-non-deterministic-behavior-for-the-gemini-2-5-pro-model-it-is-producing-different-outputs-for-identical-requests-even-when-a-fixed-seed-is-provided-along-with-a-constant-temperature-this-behavior-has-been-reliably-rep/101331 | Jan 2026 |
| Rating Roulette (judge self-inconsistency) | EMNLP 2025 | arxiv.org/pdf/2510.27106 | 2025 |
| Sage benchmark (Are We on the Right Way to Assessing LLM-as-a-Judge?) | arxiv | arxiv.org/html/2512.16041v1 | Dec 2025 |
| Judging the Judges (position bias systematic study) | ACL/IJCNLP 2025 | arxiv.org/html/2406.07791v7 | 2025 |
| Gemini 2.5 Thinking Model Updates | Google Devs Blog | developers.googleblog.com/en/gemini-2-5-thinking-model-updates/ | Feb 2026 |
| Gemini 3 Thinking Mode usage notes | Blog | oneuptime.com/blog/post/2026-02-17-how-to-use-thinking-mode-in-gemini-3-for-complex-reasoning-tasks/view | Feb 2026 |
