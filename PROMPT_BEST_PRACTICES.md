# LLM Prompt Best Practices

A reference guide for writing and revising prompts used by LLM agents. Refreshed April 2026 against current frontier models (Claude Opus 4.6, GPT-5.4, Gemini 3.1 Pro). The goal is not abstract "compliance," it is prompts the model actually executes instead of silently skipping over directives. Covers task execution gates, anti-sycophancy, linguistic-analysis prompts, and validation pass design. All examples use generic placeholders.

---

## 1. The Empirical Case

These numbers justify the techniques in this document. The headline problem is not that models refuse tasks, it is that they silently omit steps.

| Finding | Source | Implication |
|---|---|---|
| Frontier models still skip 25 to 40 percent of multi-constraint directives on novel out-of-domain instructions (Qwen3.6 Plus 75.8%, Claude Opus 4.5 58%) | IFBench leaderboard, April 2026 | Even 2026 frontier models need structural scaffolding on real prompts |
| Reasoning performance starts to degrade around 3,000 tokens even on models with 256K to 1M context windows | Prompt-bloat study, MLOps Community 2026 | Focus beats raw length; longer prompts degrade, they do not help |
| One-shot often beats few-shot for LLM-as-judge tasks; additional examples hurt when label balance or order is off | Confident AI 2026, Label Your Data 2026 | Calibrate 1 to 3 examples per criterion, not 3 to 5 |
| GPT-4 reaches 91.7 percent zero-shot accuracy on TOEFL11 native-language identification | Lotfi et al., arxiv 2312.07819 | Zero-shot is strong for linguistic-analysis prompts when features are named |
| Self-correction flips 58.8 percent of initially correct answers to wrong | ACL 2025 | Naive "check your work" prompts actively harm outputs |
| ~29 percent sycophancy reduction achievable through prompt design alone, without fine-tuning | sparkco.ai, 2025 | Anti-sycophancy is an engineering problem, not a model problem |
| The "Wait" prefix before self-correction prompts reduces blind-spot rate by 89.3 percent | arxiv 2507.02778, 2025 | A single word can unlock dormant self-correction capability |
| LLMLingua-2 compresses prompts 3x to 6x with maintained accuracy | LLMLingua-2, NAACL 2025 | Compress before decomposing when a prompt has grown heavy |

---

## 2. First-Pass Prompt Structure

### 2.1 Use Tagged Blocks

Wrap distinct prompt components in descriptive XML-style tags. This makes each section independently addressable and reduces misinterpretation.

```
<role>
You are a skeptical evaluator applying rigorous standards to {task_type}.
Do not affirm or praise content before evaluating it.
Begin immediately with the evaluation.
</role>

<instructions>
1. Evaluate each item against the gates defined below.
2. Output a VERDICT line for every item — no exceptions.
3. Do not rewrite items. Judge only what is submitted.
</instructions>

<context>
{BACKGROUND_INFORMATION}
</context>

<input>
{USER_CONTENT_TO_EVALUATE}
</input>

<output_format>
One line per item: CRITERION_A=yes CRITERION_B=no → VERDICT N: KEEP or DROP
</output_format>
```

Consistent tag names across all prompts in a system make the prompts programmable — parsers can extract sections by tag.

### 2.2 Number Every Directive

Plain lists of instructions are processed as narrative. Numbered lists are treated as discrete, individually trackable obligations.

```
WRONG:
Evaluate each item. Drop failing items. Do not rewrite. Output one line per item.

CORRECT:
1. Evaluate each item against all gates.
2. Drop items that fail any gate.
3. Do not rewrite failing items — drop only.
4. Output exactly one VERDICT line per item.
```

Numbered directives also allow targeted self-checking: "Before finishing, confirm you have followed directives 1–4."

### 2.3 Manage Prompt Length and Placement, Not a Hard Word Cap

Earlier guidance in this document capped prompts at 1,500 words. That rule was derived from 2024 and 2025 models. It is no longer the right framing. Current frontier models accept 10,000-plus word prompts without structural failure, but output quality still peaks on focused prompts.

The real 2026 constraints are three:

1. **Reasoning degradation starts around 3,000 tokens**, regardless of whether the model advertises a 256K or 1M context window. Past that point, extra tokens rarely help and often hurt (Prompt-bloat study, MLOps Community 2026).
2. **Lost-in-the-middle effects persist** on RoPE-based models. Information in the first 32K and last 16K tokens is retrieved reliably; the middle band is not. Place critical directives at both the start and the end of the prompt.
3. **Practical high-quality retrieval window is 16K to 32K tokens** even for 128K and 1M context models (elvex 2026, devtk.ai 2026).

Order of operations when a prompt is getting heavy:

- **Focus first.** Strip context that is not load-bearing for the current step. Most prompt bloat is irrelevant background, not essential instruction.
- **Compress second.** LLMLingua-2 and similar task-agnostic compressors cut prompt length 3x to 6x with no accuracy loss (LLMLingua-2, NAACL 2025). Summarization and keyphrase extraction are also valid.
- **Decompose third.** Split into chained calls where earlier outputs feed later stages. Still the strongest single lever for multi-stage tasks.
- **Place, always.** Put load-bearing directives in the first and last sections, never buried in the middle.

### 2.4 Place Long Context Before Instructions

When your prompt includes a long document (transcript, article, submission), place it before the instructions and the query. Research showed up to 30% quality improvement from this ordering.

```
WRONG:
<instructions>...</instructions>
<context>{LONG_DOCUMENT}</context>
Evaluate this document.

CORRECT:
<context>{LONG_DOCUMENT}</context>
<instructions>...</instructions>
Evaluate this document.
```

### 2.5 Explain the "Why" Behind Each Directive

Models generalize from explanations. A directive with a reason attached is more robust than a bare prohibition.

```
WEAKER:
Do not use ellipses.

STRONGER:
Do not use ellipses, because the text-to-speech engine does not know how to pronounce them.
```

### 2.6 State What TO Do, Not Just What NOT to Do

Prohibition-only instructions leave the model with no alternative path. Pair every "do not" with a "instead, do."

```
WEAKER:
Do not use markdown formatting.

STRONGER:
Write in flowing prose paragraphs. Do not use markdown headers, bullet points, or bold text.
```

### 2.7 Over-Generate Then Filter

For any task where you need exactly N outputs, generate N+2 and filter the weakest 2 in a separate step. This guarantees the target count is always met even after filtering failures or edge cases are removed.

### 2.8 Few-Shot Examples: Use 1 to 3, Not 3 to 5

Earlier guidance suggested 3 to 5 diverse examples. The 2026 research overrules that. Key findings:

- **One-shot often beats few-shot** on LLM-as-judge tasks. Adding examples past the first can measurably hurt performance across major models (Confident AI 2026).
- **Few-shot is unstable** with respect to example order, label balance, and count. Bias in the examples propagates directly into the model's judgments.
- **When few-shot does help** (GPT-4 judge consistency went from 65.0 to 77.5 percent in one study), it only helps when the examples reflect the natural distribution of scores you expect at inference.

New rule for evaluation prompts: **1 to 3 calibrated examples per criterion. Always pair PASS with FAIL. Balance ordering, and rotate which comes first across criteria.** If a single clear PASS-FAIL pair communicates the criterion, stop there. Do not pad with more examples hoping it will help, because in 2026 it often does not.

---

## 3. Task Execution Gates

The most common failure mode in evaluation prompts is not that the model gives wrong answers. It is that the model silently omits a criterion entirely. Task execution gates are the fix: they turn each directive into an explicit, testable obligation the model must address in its output.

### 3.1 The Core Pattern

A task execution gate is a named, binary criterion that an LLM output item must pass. Gates replace vague quality judgments with explicit, testable questions, and they force the model to answer each one rather than skipping past them.

**Structure of one gate:**

```
GATE NAME: Plain-English question that resolves to YES or NO.
PASS example: [concrete example that would earn YES]
FAIL example: [concrete example that would earn NO, and why]
```

**Output format (machine-parseable):**

```
VERDICT N: KEEP
VERDICT N: FAIL
REWRITE N: [corrected version if rewriting is appropriate]
```

Parse with: `re.finditer(r'VERDICT\s+(\d+)\s*:\s*(KEEP|DROP|PASS|FAIL)', raw, re.IGNORECASE)`

### 3.2 Embed PASS and FAIL Examples in the Prompt

Abstract gate definitions are less reliable than definitions accompanied by concrete examples. Embed both inside the prompt — not in a separate document.

```
SUBSTITUTION-PROOF: Can NO other item in the available set fit this blank without
changing the meaning?

PASS: "After three days without food, the hikers suffered from famine."
      (Only "famine" fits — the three-day context locks it.)

FAIL: "The situation was __________."
      (Many words fit — the blank is not locked.)
```

### 3.3 Additive Scoring vs. Binary Pass/Fail

| Format | Pros | Cons |
|---|---|---|
| Binary PASS/FAIL | Simple to parse, clear | Hides which criterion failed; no gradation |
| Additive score (1 point per criterion, TOTAL=N/M) | Shows failure pattern; enables threshold tuning | Slightly more complex to parse |

**Additive format:**

```
1. "item": CRITERION_A=yes CRITERION_B=yes CRITERION_C=no CRITERION_D=yes TOTAL=3/4 → VERDICT 1: KEEP
2. "item": CRITERION_A=no CRITERION_B=no CRITERION_C=no CRITERION_D=yes TOTAL=1/4 → VERDICT 2: DROP
```

The VERDICT still parses with the same regex. TOTAL is for human review and threshold tuning.

### 3.4 Hard Rules

For any criterion where failure should trigger immediate DROP regardless of other gates, state this explicitly in the prompt:

```
CRITERION_X=no → automatic DROP regardless of all other gates.
```

This prevents the model from averaging across criteria to "rescue" a fundamentally flawed item.

### 3.5 Programmatic Fallback

Always build a fallback for when LLM gate output is ambiguous or malformed:

- **Primary:** Parse VERDICT lines with regex
- **Fallback:** Parse a secondary format (e.g., `DROP: 3, 7, 9`)
- **Final fallback:** Return the unfiltered original list and log a warning

Never hard-fail on gate parsing. The output of a gate pass is a filtered list, not an error state.

### 3.6 Self-Check Append

For high-stakes generation, append a verification instruction as the final item. This is a separate API call — not appended to the generation call.

```
Before you finish, verify each requirement in <instructions> has been addressed.
List any requirement you were unable to fulfill, and why.
```

### 3.7 Output-Length Trap in Per-Item Gates

Per-item gates make output grow linearly with item count. Near the response cap, tail items get truncated and the parser silently drops them, so it looks like a format regression rather than a length limit.

Rules:
- **Budget worst-case output** (`items x lines_per_item`, all-FAIL path) against the response cap before adding a gate.
- **Don't stack pre-commitment and post-hoc self-check** on the same item. One visible commitment per item captures most of the rigor.
- **Fire the expensive block on FAIL only.** PASS items don't need fresh candidate reasoning; rewrites do.
- **Split the call before dropping rigor.** If the gate doesn't fit, chain it (§2.3) rather than thin it.
- **Detect truncation explicitly.** Log raw response length and assert parsed-count equals expected-count on every batch.

---

## 4. Anti-Sycophancy

**Why models rubber-stamp:** RLHF training teaches models that agreeable, validating responses earn higher human satisfaction ratings. This is the optimization target — not truthfulness. The result is a systematic bias toward agreement, flattery, and generous evaluations.

### 4.1 Skeptical Role Assignment

Assign a role that makes skepticism the default behavior, not an override.

```
WEAK role:
You are a helpful assistant reviewing these materials.

STRONG role:
You are a rigorous evaluator applying strict criteria. Your job is to find
items that fail, not to validate items that pass. Reject any item that does
not clearly meet all criteria.
```

### 4.2 Explicit Anti-Flattery Instruction

Include this (or a version of it) in every evaluation prompt:

```
Do not open with praise or agreement. Do not affirm the quality of the content before
evaluating it. Begin immediately with the evaluation.
```

### 4.3 Forced Counterargument

For tasks where the model must reach a conclusion or recommendation, require it to produce at least one counterargument before the conclusion:

```
Before stating your conclusion, identify at least two ways your proposed answer
could be wrong or incomplete.
```

### 4.4 Evidence-First Question Framing

Rephrase evaluative questions from validation-seeking to critique-seeking:

```
VALIDATION-SEEKING (triggers sycophancy):
Is this a good warm-up question?

CRITIQUE-SEEKING (triggers evaluation):
What are the weaknesses in this warm-up question? Could any student refuse to answer it?
```

### 4.5 Combined Effect

These four techniques combined reduce sycophancy by approximately 29% without any model fine-tuning (sparkco.ai, 2025). They stack — each one adds independent mitigation.

---

## 5. Second-Pass Validation

### 5.1 The Core Finding

> "Current LLMs cannot improve their reasoning performance through intrinsic self-correction."
> — ICLR 2024

Intrinsic self-correction = same model, no external signal, freeform "check your work" instruction.

**Quantified:** GPT-3.5-turbo overturns up to **58.8% of initially correct answers** during self-correction (ACL 2025). Models change answers more than 6 times in 10 rounds for 80%+ of samples without converging.

**Three failure mechanisms:**
1. **Recency bias** — the model focuses on the validation instruction rather than the original task
2. **Answer wavering** — repeated rounds produce oscillation, not improvement
3. **Self-correction blind spot** — average 64.5% blind spot rate: models reliably correct identical errors in external text but miss them in their own output

### 5.2 When Validation Works

| Condition | Reliability |
|---|---|
| Different/stronger model as judge | High — recommended default |
| External verification (code execution, math checker, regex) | High — best for verifiable criteria |
| Structured gate scoring (named criteria, binary verdicts, examples in prompt) | High — structured gates eliminate the ambiguity that causes wavering |
| Domain-specific fine-tuned validator | High |
| Same model, freeform "check your work" | Unreliable — often degrades output |
| Reasoning models (o1, DeepSeek-R1 style) | Already embed self-correction; second pass wastes tokens |

**The key insight:** Structured gate scoring is reliable because the model is not reasoning about quality — it is scoring against pre-defined criteria with examples already in the prompt. This eliminates the ambiguity that causes answer wavering.

### 5.3 The "Wait" Prefix

For cases where the same model must validate its own output, prepend a single word before the validation prompt:

```python
validation_prompt = "Wait.\n\n" + validation_prompt
```

Research (arxiv 2507.02778, 2025) found this reduces the self-correction blind spot by 89.3%. Mechanism: activates dormant self-correction capability already present in the model.

### 5.4 Recency Bias Fix

When using the same model for validation, append the original task at the END of the validation prompt (after all content input blocks):

```
[validation criteria]
[content to evaluate]

Reminder: The original task was: {ORIGINAL_TASK_SUMMARY}
```

This alone reduces correct-answer flips by 5–11% (ACL 2025). The reminder counteracts the model's tendency to focus on the validation instruction rather than the original goal.

### 5.5 Structural Requirements for Reliable Validation Prompts

**Always include the original task.** The judge must see original instructions + output, never just the output. Without task context, the validator is guessing what "correct" means.

**One criterion per call (high-stakes), up to 3 bundled (low-stakes).** Combining "check accuracy, safety, and style" in one prompt degrades all three for high-stakes scoring. On current 2026 frontier models, bundling 2 to 3 named criteria in one call is acceptable for low-stakes filtering tasks. Keep it to one criterion per call whenever the score drives a downstream action.

**Atomic checklist scoring.** Decompose vague criteria into binary sub-questions:

```
Award 1 point if the answer is relevant to the question.
Award 1 additional point if the answer is factually accurate.
Award 1 further point if the answer is under 200 words.
Total: _/3
```

This format outperforms holistic quality scoring by ~30% correlation with human ratings (HuggingFace LLM-as-judge cookbook, 2025).

**Labeled integer scale (1–4).** Not float, not 1–10. Concrete text anchors at each level:

```
1 = Fails the criterion entirely
2 = Partially meets the criterion
3 = Meets the criterion with minor issues
4 = Fully meets the criterion
```

**Require reasoning before verdict.** The reasoning field must be completed before the score. This creates an audit trail and activates chain-of-thought:

```json
{
  "reasoning": "step-by-step explanation of rating",
  "score": 3
}
```

### 5.6 LLM-as-Judge Template

Use when a different/stronger model evaluates the output of a generation model.

```
You are an objective evaluator. Assess only the criterion listed.
Ignore stylistic differences unless they affect comprehension.
Longer answers are not necessarily better.

Original task:
<task>{ORIGINAL_TASK_INSTRUCTIONS}</task>

Response to evaluate:
<response>{MODEL_OUTPUT}</response>

Criterion: {SINGLE_CRITERION}

Scale:
1 = Fails the criterion entirely
2 = Partially meets the criterion
3 = Meets the criterion with minor issues
4 = Fully meets the criterion

Output as JSON:
{
  "reasoning": "step-by-step explanation of your rating",
  "score": <1-4>
}

You MUST complete "reasoning" before providing "score".

Reminder: The original task was: {ORIGINAL_TASK_INSTRUCTIONS}.
Evaluate only the criterion above, not overall quality.
```

### 5.7 Bias Mitigations for Pairwise Evaluation

When choosing between two candidate outputs:

- **Position bias:** Evaluate both (A, B) and (B, A) orderings. Only count consistent wins (~40% inconsistency on position alone in GPT-4 on pairwise tasks).
- **Verbosity bias:** State explicitly: `"Length is not a quality signal. A shorter answer that fully addresses the criterion scores higher than a longer one that does not."`
- **Self-preference bias:** Use a different model as judge when possible. Most models rate their own output higher when sources are anonymized.

---

## 6. Pre-Flight Checklist

Apply before deploying any prompt to an agent.

1. **Tagged blocks.** Does every distinct section (role, instructions, context, input, output format) have its own descriptive XML tag?
2. **Numbered directives.** Are all instructions numbered for individual traceability?
3. **Length and placement.** Is the prompt focused under ~3,000 tokens where feasible? Are critical directives placed at both the start and the end (not buried in the middle)? If the task is genuinely multi-stage, is it decomposed into chained calls?
4. **Gate examples, calibrated count.** Does each evaluation criterion have 1 to 3 examples, with PASS and FAIL paired, ordering balanced? Not 3 to 5 diverse examples.
5. **Machine-parseable output.** Can every gate verdict be extracted with a regex? Is the output format defined with a concrete example?
6. **Skeptical role.** Does the prompt assign a critical/evaluator role rather than a helpful/assistant role?
7. **Do-instead-of-don't.** Are all prohibition instructions paired with an "instead, do" statement?
8. **Validation model.** Is the validation pass using the same model as the generation pass? If yes: use structured gate scoring (not freeform critique) + "Wait" prefix + recency reminder at end.
9. **Original task in validation.** Does the validation prompt include the original task at the top AND a reminder at the end?
10. **One criterion per validation call (high-stakes).** Is each high-stakes evaluation criterion assessed separately? Low-stakes filtering may bundle up to 3 criteria per call.
11. **Linguistic-analysis path (conditional).** If the prompt evaluates properties of the writing itself (style, register, L1 transfer, authorship, human-vs-AI stylometry), does it: (a) enumerate explicit linguistic feature categories, (b) force reasoning before verdict, (c) require cited token or phrase evidence for each feature? See Section 7. This item is N/A for prompts that are not linguistic evaluations.

---

## 7. Prompts for Linguistic Analysis

A growing class of evaluation prompts asks an LLM to judge the properties of writing itself rather than the content it conveys. Examples include native-language identification, register and style classification, L1 transfer fingerprinting, authorship attribution, genre fit, and human-versus-AI stylometry. These prompts have different failure modes from content-evaluation prompts, and they need their own construction rules.

### 7.1 Why Linguistic Analysis Prompts Fail

When asked for a holistic judgment ("is this L1 English speaker writing?"), frontier models produce confident answers that are poorly calibrated. They appear to succeed on easy cases and fail silently on hard ones. The mechanism is that the model is matching surface features without surfacing which features it used, so errors are invisible to the caller.

At the same time, LLMs are genuinely strong at this class of task when prompted correctly. GPT-4 hit 91.7 percent zero-shot accuracy on the TOEFL11 native-language identification benchmark (Lotfi et al., arxiv 2312.07819). The gap between failure and success is prompt construction, not model capability.

### 7.2 Five Rules for Linguistic Analysis Prompts

1. **Enumerate feature categories explicitly.** Do not ask for a holistic judgment. Tell the model which linguistic features to inspect: spelling error patterns, syntactic structures, article and preposition usage, direct-translation artifacts, cohesion markers, lexical choice, morphological errors, punctuation habits. The named categories act as task execution gates (Section 3).

2. **Force reasoning before verdict.** Require a `<reasoning>` block that must be completed before any verdict. Linguistic judgments produced without explicit reasoning are uncalibrated. This is a chain-of-thought requirement, not a politeness.

3. **Require cited evidence.** Every feature the model claims to have observed must come with a cited token or phrase from the input. This creates an audit trail and surfaces hallucinated reasoning. If the model cannot cite the evidence, the feature observation is unreliable.

4. **Prefer zero-shot when features are named.** When the feature categories are clearly defined, zero-shot performs well (TOEFL11 91.7 percent zero-shot). Add examples only when a specific criterion is genuinely ambiguous from its name alone, and then keep to 1 to 3 examples per criterion (Section 2.8).

5. **Build an in-class iteration loop for closed-set outputs.** If the output must be from a fixed set (a specific L1 label, a specific genre), wrap the prompt in a loop: if the model returns a label outside the set, feed the response back with "that label is not in the set {VALID_LABELS}, choose again" until it converges. This is one of the few cases where same-model self-correction is reliable, because the check is a deterministic set-membership test.

### 7.3 Template

```
<role>
You are a forensic linguist. Your job is to identify which linguistic features
are present in a piece of writing and cite the specific evidence. Do not
produce a verdict until you have completed the reasoning block. Do not
affirm or praise the writing before analyzing it.
</role>

<instructions>
1. Read the sample in <input> carefully.
2. For each feature listed in <features>, decide whether it is present.
3. For every feature you mark as present, cite at least one specific token
   or phrase from the input as evidence.
4. Complete the <reasoning> block fully before producing <features_cited>
   or <verdict>.
5. Your verdict must be a single label drawn only from the set in <labels>.
</instructions>

<features>
SPELLING_ERRORS: orthographic mistakes inconsistent with target variety
SYNTACTIC_L1_TRANSFER: word order, article, or preposition patterns that
   reflect another language's grammar
DIRECT_TRANSLATION: multiword expressions calqued from another language
COHESION_MARKERS: register or discourse markers atypical for the target
LEXICAL_CHOICE: word choices that are unusual for a native writer
</features>

<labels>
{VALID_LABEL_SET}
</labels>

<input>
{WRITING_SAMPLE}
</input>

<output_format>
<reasoning>
step-by-step inspection of each feature category,
with cited tokens or phrases from <input>
</reasoning>

<features_cited>
SPELLING_ERRORS: yes|no, evidence: "..."
SYNTACTIC_L1_TRANSFER: yes|no, evidence: "..."
DIRECT_TRANSLATION: yes|no, evidence: "..."
COHESION_MARKERS: yes|no, evidence: "..."
LEXICAL_CHOICE: yes|no, evidence: "..."
</features_cited>

<verdict>LABEL</verdict>
```

Note: this template uses commas as delimiters. Avoid em dashes in real prompts, since downstream text-to-speech tooling cannot pronounce them.

### 7.4 Feature Category Starter Pack

Use these as a starting point and prune to the subset relevant to the task:

| Task | Feature categories to enumerate |
|---|---|
| Native language identification | spelling errors, article/preposition usage, syntactic L1 transfer, direct-translation artifacts, punctuation habits |
| Register / style classification | lexical formality, sentence length variance, cohesion markers, hedging, terminology density |
| L1 transfer fingerprinting | word order, prepositional patterns, tense and aspect usage, determiner errors |
| Human vs AI stylometry | sentence-length variance, lexical diversity, burstiness, hedge-word density, formulaic transitions, token-rank entropy patterns |
| Authorship attribution | function-word frequencies, punctuation habits, signature vocabulary, sentence-start patterns |
| Genre fit | register markers, discourse moves, audience-address patterns, typical structural slots |

### 7.5 When Not To Use This Pattern

This pattern is specifically for prompts that evaluate the properties of writing. Do not use it for prompts that evaluate the content of writing (factual accuracy, task completion, code correctness). Those belong in Sections 3 and 5.
