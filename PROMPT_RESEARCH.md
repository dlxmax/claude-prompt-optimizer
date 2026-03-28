# Prompt Engineering Research Archive

Compiled March 2026. Indexed by topic for fast recall in future prompt-related tasks.

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

AGENTIF (NeurIPS 2025) found compliance degrades sharply with prompt length:
- Instructions exceeding ~6,000 words: compliance approaches zero
- Practical threshold for generation prompts: ~1,500 words before decomposing into chained calls

### Few-Shot Examples

Models perform measurably better on example-inferred constraints than specification-only constraints (AGENTIF findings). Include 3–5 diverse examples in `<examples>` tags. More than 5 can introduce unintended pattern matching.

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
