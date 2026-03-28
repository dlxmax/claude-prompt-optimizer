# LLM Prompt Best Practices

A reference guide for writing and revising prompts used by LLM agents. Covers directive compliance, anti-sycophancy, structured gates, and validation pass design. All examples use generic placeholders.

---

## 1. The Empirical Case

These numbers justify the techniques in this document.

| Finding | Source | Implication |
|---|---|---|
| State-of-the-art LLM: 87% compliance on simple benchmarks → 58.5% on real multi-constraint agentic instructions | AGENTIF, NeurIPS 2025 | Prompt structure is not optional for agent prompts |
| Best model achieves only 27.2% full success on multi-constraint instructions | AGENTIF, NeurIPS 2025 | Condition, tool, and format constraints are hardest |
| Fewer than 25% of reasoning traces comply with instructions, even when final outputs look correct | ReasonIF, Oct 2025 | Final output compliance alone is insufficient |
| Performance drops 61.8% when prompts are rephrased with nuanced wording changes | IFEval++, 2025 | Exact wording of directives matters more than assumed |
| Self-correction flips 58.8% of initially correct answers to wrong | ACL 2025 | Naive "check your work" prompts actively harm outputs |
| ~29% sycophancy reduction achievable through prompt design alone, without fine-tuning | sparkco.ai, 2025 | Anti-sycophancy is an engineering problem, not just a model problem |
| The "Wait" prefix before self-correction prompts reduces blind spot rate by 89.3% | arxiv 2507.02778, 2025 | A single word can unlock dormant self-correction capability |

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

### 2.3 Stay Under ~1,500 Words Per Call

AGENTIF research found instruction compliance degrades sharply as prompt length grows. When your prompt (role + instructions + context + input) exceeds ~1,500 words:

- **Decompose:** Split into chained calls where earlier outputs feed later stages
- **Summarize context:** Use only the context directly relevant to the current stage
- **Separate evaluation:** Move validation to a dedicated second call rather than appending to the generation call

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

---

## 3. Compliance Gates

### 3.1 The Core Pattern

A compliance gate is a named, binary criterion that an LLM output item must pass. Gates replace vague quality judgments with explicit, testable questions.

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

**One criterion per call.** Combining "check accuracy, safety, and style" in one prompt degrades all three. Each additional criterion reduces scoring reliability.

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
```

### 5.7 Bias Mitigations for Pairwise Evaluation

When choosing between two candidate outputs:

- **Position bias:** Evaluate both (A, B) and (B, A) orderings. Only count consistent wins (~40% inconsistency on position alone in GPT-4 on pairwise tasks).
- **Verbosity bias:** State explicitly: `"Length is not a quality signal. A shorter answer that fully addresses the criterion scores higher than a longer one that does not."`
- **Self-preference bias:** Use a different model as judge when possible. Most models rate their own output higher when sources are anonymized.

---

## 6. Pre-Flight Checklist

Apply before deploying any prompt to an agent.

1. **Tagged blocks** — Does every distinct section (role, instructions, context, input, output format) have its own descriptive XML tag?
2. **Numbered directives** — Are all instructions numbered for individual traceability?
3. **Length** — Is the prompt under ~1,500 words? If not, which stages can be split into separate calls?
4. **Gate examples** — Does each evaluation criterion have a PASS example and a FAIL example embedded in the prompt?
5. **Machine-parseable output** — Can every gate verdict be extracted with a regex? Is the output format defined with a concrete example?
6. **Skeptical role** — Does the prompt assign a critical/evaluator role rather than a helpful/assistant role?
7. **Do-instead-of-don't** — Are all prohibition instructions paired with a "instead, do" statement?
8. **Validation model** — Is the validation pass using the same model as the generation pass? If yes: use structured gate scoring (not freeform critique) + "Wait" prefix + recency reminder at end.
9. **Original task in validation** — Does the validation prompt include the original task at the top AND a reminder at the end?
10. **One criterion per validation call** — Is each evaluation criterion assessed separately, not bundled with other criteria in a single holistic score?
