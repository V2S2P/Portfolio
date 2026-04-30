---
title: "Testing Claude Code's Token Optimization Skill"
date: 2026-04-30
draft: false
description: "A structured comparison of standard vs. token-optimized Claude Code output across a coding task and quiz — single run and 5-run study."
tags: ["claude", "ai", "token-optimization", "experiment"]
---

## What This Is

Claude Code is Anthropic's official CLI for Claude — an AI assistant you interact with directly in your terminal or IDE. One of its features is a **skills system**: pre-configured instruction sets that modify how Claude responds for a given task.

The `token_optimization` skill instructs Claude to minimize token usage at all times — shorter answers, no docstrings, abbreviated names, compact formatting. The goal is to get the same functional output with fewer words.

This post documents a controlled test of that skill: the same task, run twice (once without the skill, once with), and then repeated five times in each mode to check consistency.

---

## Why I Did It

Token usage matters for two reasons:

1. **Plan limits** — Claude's usage plans cap how many tokens you can consume per period. Fewer output tokens per interaction means more interactions before hitting a ceiling.
2. **Response quality** — Reducing tokens without losing accuracy is useful. But reducing tokens at the cost of readability is a trade-off worth measuring.

I wanted to see the actual numbers, not just assume the skill helps.

---

## The Test

**Task (identical for all runs):**
> Implement a Binary Search Tree in Python with insert, search, and in-order traversal methods. Then generate a 5-question multiple choice quiz about the implementation, with 4 options per question and feedback for each option.

**Measurement method:** Character count ÷ 4 (standard approximation for English + code token estimation). Margin of error: ±10–15%.

---

## Single-Run Results

The first comparison was one run of each mode.

| Metric | Standard | Optimized |
|--------|----------|-----------|
| Estimated output tokens | ~1,175 | ~415 |
| Share of combined output | 74% | 26% |
| Token reduction | — | **~65%** |

Both runs produced correct code and accurate quiz questions. The reduction came from removing docstrings, verbose feedback, expanded variable names, and the `__main__` block — not from cutting content coverage.

---

## 5-Run Study

To check consistency, the test was repeated five times in each mode.

### Standard Mode (Skill Off)

| Run | Est. Tokens |
|-----|-------------|
| S1  | ~1,050 |
| S2  | ~1,113 |
| S3  | ~1,038 |
| S4  | ~1,128 |
| S5  | ~1,038 |
| **Average** | **~1,073** |

Range: 1,038–1,128 | Std dev: ±38 tokens

### Optimized Mode (Skill On)

| Run | Est. Tokens |
|-----|-------------|
| O1  | ~398 |
| O2  | ~333 |
| O3  | ~358 |
| O4  | ~390 |
| O5  | ~305 |
| **Average** | **~357** |

Range: 305–398 | Std dev: ±38 tokens

### Aggregate Comparison

| Metric | Standard | Optimized |
|--------|----------|-----------|
| Avg estimated tokens | 1,073 | 357 |
| Share of combined avg | 75% | 25% |
| Avg token reduction | — | **~67%** |

```
S1  ████████████████████████████░░░░░░░░░░░░  1,050
S2  ██████████████████████████████░░░░░░░░░░  1,113
S3  ████████████████████████████░░░░░░░░░░░░  1,038
S4  ██████████████████████████████░░░░░░░░░░  1,128
S5  ████████████████████████████░░░░░░░░░░░░  1,038

O1  ███████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    398
O2  █████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    333
O3  ██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    358
O4  ███████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    390
O5  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    305
```

---

## Output Quality

Both modes produced functionally correct code and factually accurate quizzes across all runs.

### Code

| Attribute | Standard | Optimized |
|-----------|----------|-----------|
| Docstrings | Full, every method | None in most runs |
| Variable naming | Descriptive (`value`, `node`) | Abbreviated (`v`, `n`) |
| Inline comments | Yes | Minimal or none |
| Functional correctness | Correct, all 5 runs | Correct, all 5 runs |
| Readable by newcomer | Yes | No |
| Run-to-run consistency | High | Moderate |

### Quiz

| Attribute | Standard | Optimized |
|-----------|----------|-----------|
| Question length | ~10 words avg | ~5 words avg |
| Feedback per option | 1–2 sentences, explains why | 3–8 words, verdict only |
| Factual accuracy | Correct, all 5 runs | Correct, all 5 runs |
| Suitable for self-study | Yes | No |
| Suitable for quick review | No | Yes |

### Consistency

| Mode | Spread | Spread as % of avg |
|------|--------|-------------------|
| Standard | 90 tokens | ~8% |
| Optimized | 93 tokens | ~26% |

Absolute spread was similar for both modes, but the optimized mode's spread is proportionally larger — meaning the skill compresses output less predictably run-to-run than standard mode produces it.

---

## Impact on Plan Usage Limits

Plan limits count **total** tokens — both input and output. Input tokens (the prompt, system context, conversation history) stay roughly the same regardless of which mode is used.

Assuming ~400 input tokens per interaction:

| Mode | Avg input | Avg output | Avg total |
|------|-----------|------------|-----------|
| Standard | ~400 | ~1,073 | ~1,473 |
| Optimized | ~400 | ~357 | ~757 |

**Estimated total token reduction: ~49%**

This is lower than the 67% output-only figure because input tokens are shared across both modes. In practical terms: roughly **1.9× more interactions** within the same plan limit per period, assuming input size stays constant.

---

## Takeaways

- The skill consistently reduced output tokens by ~67% across all 5 runs. The direction was the same every time — no optimized run exceeded any standard run.
- Correctness was not affected. Both modes delivered working code and accurate quizzes.
- The cost is readability and context. Optimized output is harder to follow without prior knowledge, and quiz feedback gives verdicts without explanations.
- The real-world plan usage reduction is ~49%, not 67%, once you account for input tokens.
- Standard mode is more consistent. Optimized mode varies more in how aggressively it compresses, which means the savings are less predictable run-to-run.

Neither mode is strictly better. The appropriate choice depends on who will read the output and why.

---

## Skill v2: Quality-Preserving Compression

The v1 results raised an obvious question: can you get most of the token savings without the quality regressions? The v1 skill was cutting too deep — removing docstrings, abbreviating class names (`BST` instead of `BinarySearchTree`), stripping `__main__` blocks, and reducing quiz feedback to bare verdicts. Correct output, but not code you'd hand to someone else, and not quiz feedback that actually teaches anything.

Version 2 of the skill was redesigned around two explicit zones:

- **COMPRESS** — prose, filler, transitions, repetition. Cut everything here.
- **PRESERVE** — code names, structure, conventions, and quiz reasoning. Never touch these.

The specific changes: full descriptive class and method names always, a one-line docstring for every public class and function, standard `__main__` blocks retained, quiz feedback expanded to one sentence explaining *why* (not just "Correct"), and an explicit banned-phrase list to prevent filler from creeping back in.

### v2 5-Run Results

| Run | Code (chars) | Quiz (chars) | Total (chars) | Est. tokens |
|-----|-------------|-------------|---------------|-------------|
| V1  | 1,714       | 1,059       | 2,773         | ~693        |
| V2  | 1,714       | 1,094       | 2,808         | ~702        |
| V3  | 1,714       | 1,130       | 2,844         | ~711        |
| V4  | 1,714       | 1,186       | 2,900         | ~725        |
| V5  | 1,714       | 1,197       | 2,911         | ~728        |
| **Average** | **1,714** | **1,133** | **2,847** | **~712** |

Range: 693–728 | Spread: ~35 tokens (~5% of avg)

### Three-Way Comparison

| Metric | Standard | v1 Optimized | v2 Optimized |
|--------|----------|--------------|--------------|
| Avg est. tokens | ~1,073 | ~357 | ~712 |
| vs. Standard | — | −67% | −34% |
| vs. v1 | — | — | +99% |

```
Standard      ████████████████████████████████████████  ~1,073
v2 Optimized  ████████████████████████░░░░░░░░░░░░░░░░    ~712
v1 Optimized  █████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░    ~357
```

### Code Quality

| Attribute | Standard | v1 Optimized | v2 Optimized |
|-----------|----------|--------------|--------------|
| Class name | `BinarySearchTree` | `BST` | `BinarySearchTree` |
| Method names | `inorder_traversal` | `inorder` | `inorder_traversal` |
| Docstrings | Full, all methods | None | One-liner, public only |
| `__main__` block | Yes, with labels | No (bare calls) | Yes, with inline comments |
| Functional correctness | Correct, all runs | Correct, all runs | Correct, all runs |
| Readability (newcomer) | High | Low | Moderate–High |
| Run-to-run consistency | High | Moderate | Very High |

### Quiz Quality

| Attribute | Standard | v1 Optimized | v2 Optimized |
|-----------|----------|--------------|--------------|
| Question length | ~10 words avg | ~5 words avg | ~8 words avg |
| Feedback | 1–2 sentences, explains why | 3–8 words, verdict only | 1 sentence, explains why |
| Suitable for self-study | Yes | No | Yes |
| Suitable for quick review | No | Yes | Yes |

### Consistency

| Mode | Spread | Spread as % of avg |
|------|--------|-------------------|
| Standard | ~90 tokens | ~8% |
| v1 Optimized | ~93 tokens | ~26% |
| v2 Optimized | ~35 tokens | ~5% |

Code output was identical across all 5 v2 runs (1,714 chars every time). The PRESERVE rules lock structure and naming completely — only quiz question wording varies run-to-run. v2 is more predictable than either prior mode, despite sitting between them on token volume.

### Impact on Plan Usage Limits

| Mode | Avg input | Avg output | Avg total | vs. Standard |
|------|-----------|------------|-----------|--------------|
| Standard | ~400 | ~1,073 | ~1,473 | — |
| v1 Optimized | ~400 | ~357 | ~757 | −49% |
| v2 Optimized | ~400 | ~712 | ~1,112 | −25% |

v1 gave roughly 1.9× more interactions per plan period. v2 gives ~1.3× — meaningful, but more conservative. The gap reflects the token cost of restoring docstrings, full names, `__main__`, and quiz reasoning.

### Takeaways

- v2 reduces output tokens by ~34% from Standard — meaningful compression without touching anything that affects usability.
- All quality regressions from v1 are closed: descriptive names, public docstrings, `__main__` blocks, and quiz reasoning are all restored.
- v2 is significantly more consistent than v1. Code output is completely deterministic across runs; only quiz wording introduces any variation.
- The token cost of quality is real but bounded: v2 uses ~72% more tokens than v1, almost entirely from the restored quality features.
- v2 is a better fit for always-on usage. v1 is the right choice when maximum compression matters more than readability; v2 is the right choice when the output needs to stand on its own.
