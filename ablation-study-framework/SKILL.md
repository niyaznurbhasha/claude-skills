---
name: ablation-study-framework
description: Systematic evaluation of LLM augmentation layers (baseline, +component, +component) with N-dimension rubric scoring and cross-mode comparison. Use when user wants to compare LLM system configurations, measure the impact of RAG/prompts/tools, run an ablation study, or determine which components actually improve output quality.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: evaluation-and-quality
  tags: [ablation, evaluation, comparison, rubric, augmentation, rag, prompts, A/B-testing, decomposition]
---

# Ablation Study Framework

Measure the individual contribution of each component in an LLM system by running the same eval tasks across multiple configurations (modes), scoring each with an N-dimension rubric, and producing a decomposition showing exactly what each layer adds.

## When to Use This

- You added RAG, tools, system prompts, or other augmentation to an LLM system and need to know which layers actually help
- You want to justify keeping or removing a component based on data
- You need per-category analysis (e.g., "RAG helps for factual questions but hurts creative tasks")
- You want to compare prompt architectures, retrieval strategies, or model configurations

## Core Concepts

### Modes (configurations to compare)

Each mode represents a system configuration. Build them as an additive stack:

```
baseline         -- raw LLM, no augmentation
+rag             -- add retrieval-augmented generation
+rag+rules       -- add domain-specific rules/protocols
+rag+rules+tools -- add tool calling
```

Or as parallel alternatives:

```
single_query_rag   -- one search query per input
multi_query_rag    -- LLM generates multiple targeted queries
```

### Rubric Dimensions

Score every response on N independent dimensions (0.0 to 1.0 each). Common dimensions:

| Dimension | What it measures |
|-----------|-----------------|
| Correctness | Factual accuracy, no hallucinations |
| Completeness | Covers all key aspects of the question |
| Safety | Appropriate risk handling, referrals when needed |
| Specificity | Concrete numbers, steps, actionable advice |
| Context-awareness | Uses user profile/history appropriately |
| Tone | Professional, appropriate register |

### Composite Score

Weighted combination of dimensions. Use adaptive weights based on what data is available:

```python
def compute_composite(scores: dict, mode: str, task: dict) -> float:
    """Compute weighted composite score with adaptive weighting."""

    # When LLM judge scores are available, weight them heavily
    if "judge_correctness" in scores:
        composite = (
            scores["judge_correctness"] * 0.25
            + scores["judge_completeness"] * 0.20
            + scores["judge_safety"] * 0.20
            + scores["judge_specificity"] * 0.10
            + scores["element_coverage"] * 0.15
            + scores["safety_heuristic"] * 0.10
        )
    else:
        # Fallback to heuristic weights
        composite = (
            scores.get("element_coverage", 0) * 0.35
            + scores.get("evidence_cited", 0) * 0.20
            + scores.get("specificity", 0) * 0.20
            + scores.get("actionability", 0) * 0.25
        )

    # Mode-specific bonuses (e.g., protocol usage in augmented modes)
    if mode in ("full", "grouped") and scores.get("protocol_referenced") is not None:
        composite = composite * 0.9 + scores["protocol_referenced"] * 0.1

    # Hard penalties for safety-critical misses
    if task.get("referral_expected") and scores.get("referral_present") == 0.0:
        composite *= 0.7

    return round(min(composite, 1.0), 3)
```

## Step 1: Define Eval Tasks with Category Labels

Tasks need category labels so you can slice results and find which modes win where.

```python
ABLATION_TASKS = [
    {
        "id": "injury_001",
        "category": "injury",
        "difficulty": "hard",
        "input": "I have sharp knee pain when squatting, what should I do?",
        "expected_elements": ["pain assessment", "modification", "referral"],
        "safety_critical": True,
        "referral_expected": True,
        "scenario": {
            "goal": "strength training",
            "injuries": [{"body_part": "knee", "severity": 7}],
        },
    },
    {
        "id": "nutrition_001",
        "category": "nutrition",
        "difficulty": "medium",
        "input": "How much protein should I eat for muscle gain?",
        "expected_elements": ["g/kg", "timing", "sources"],
        "scenario": {"goal": "muscle gain", "weight_kg": 80},
    },
]
```

### Gold reference answers (optional but valuable)

For each task, provide a gold-standard answer that the LLM judge compares against:

```python
GOLD_ANSWERS = {
    "injury_001": {
        "answer": "For sharp knee pain during squats, first assess severity on 1-10...",
        "must_contain": ["pain-free range", "physical therapist"],
        "must_not_contain": ["push through the pain", "no pain no gain"],
    },
}
```

## Step 2: Build Mode-Specific Execution

Each mode assembles different context before calling the LLM.

```python
def run_task(task: dict, mode: str, llm, judge_llm=None) -> dict:
    """Run a single task in the specified mode."""
    context = {}
    rag_text = ""
    rules_text = ""

    # Retrieve augmentation based on mode
    if mode in ("rag", "full", "grouped"):
        if mode == "grouped":
            # Multi-query: generate targeted queries, retrieve in parallel
            queries = generate_targeted_queries(task, llm)
            chunks = parallel_retrieve(queries, top_k=3)
            context["queries"] = queries
        else:
            # Single-query: use raw input
            chunks = retrieve(task["input"], top_k=5)
        context["chunks"] = chunks
        rag_text = format_chunks(chunks)

    if mode in ("full", "grouped"):
        rules_text = load_domain_rules(task)
        context["rules"] = rules_text

    # Build messages using system/human split architecture
    messages = build_messages(task, mode, rag_text, rules_text)

    # Get LLM response
    reply = llm.invoke(messages).content

    # Score with all available layers
    scores = score_reply(reply, task, context, mode, judge_llm=judge_llm)

    return {
        "task_id": task["id"],
        "mode": mode,
        "category": task["category"],
        "input": task["input"],
        "reply": reply,
        "scores": scores,
        "chunks_loaded": len(context.get("chunks", [])),
        "queries_generated": context.get("queries"),
    }
```

### Message architecture matters

The system/human message split significantly affects how well the LLM uses retrieved context. Test this as its own ablation variable.

```python
def build_messages(task: dict, mode: str, rag_text: str, rules_text: str) -> list:
    """System message = static instructions. Human message = dynamic context."""
    system_msg = "You are a domain expert. Reply in 3-5 sentences. Cite sources inline."

    if mode != "baseline" and rag_text:
        system_msg += "\nKnowledge references are provided. BASE your answer on them."

    human_parts = []

    # Person model / scenario context
    human_parts.append(f"[CONTEXT]\n{format_scenario(task)}")

    # Evidence (dynamic, retrieved)
    if rag_text:
        human_parts.append(f"\n[EVIDENCE]\n{rag_text}")

    # Rules/protocols (after evidence, as coequal reference)
    if rules_text:
        human_parts.append(f"\n[RULES]\n{rules_text}")

    human_parts.append(f"\n{task['input']}")

    return [
        {"role": "system", "content": system_msg},
        {"role": "user", "content": "\n".join(human_parts)},
    ]
```

## Step 3: Run All Modes and Collect Results

```python
def run_ablation(tasks: list, modes: list, llm, judge_llm=None) -> dict:
    """Run all tasks across all modes. Returns results keyed by mode."""
    all_results = {}

    for mode in modes:
        print(f"\n{'='*60}")
        print(f"Mode: {mode.upper()} ({len(tasks)} tasks)")
        print(f"{'='*60}")

        mode_results = []
        for task in tasks:
            result = run_task(task, mode, llm, judge_llm=judge_llm)
            mode_results.append(result)

            # Print per-task summary
            s = result["scores"]
            print(f"  [{task['id']}] composite={s['composite']:.2f} "
                  f"coverage={s.get('element_coverage', 0):.2f} "
                  f"safety={s.get('safety', 0):.2f}")

        all_results[mode] = mode_results

    return all_results
```

## Step 4: Generate the Decomposition Report

This is the key output -- a table showing what each layer adds and a stack decomposition.

```python
def print_ablation_report(all_results: dict, modes: list):
    """Print cross-mode comparison with lift calculations."""

    # Per-mode averages
    mode_avgs = {}
    for mode, results in all_results.items():
        avg = sum(r["scores"]["composite"] for r in results) / len(results)
        mode_avgs[mode] = avg

    # Main results table
    print(f"\n{'='*60}")
    print("ABLATION RESULTS")
    print(f"{'='*60}")
    print(f"{'Mode':<25} {'Composite':>10} {'Lift':>10}")
    print("-" * 50)

    baseline_score = mode_avgs.get(modes[0], 0)
    for mode in modes:
        score = mode_avgs[mode]
        if mode == modes[0]:
            lift = "--"
        else:
            pct = ((score - baseline_score) / baseline_score * 100) if baseline_score else 0
            lift = f"+{pct:.1f}%"
        print(f"  {mode:<23} {score:>8.3f}   {lift:>8}")

    # Stack decomposition (incremental lifts)
    print(f"\nStack decomposition:")
    prev_score = baseline_score
    print(f"  Baseline ({baseline_score:.3f})")
    for mode in modes[1:]:
        score = mode_avgs[mode]
        delta = score - prev_score
        pct = (delta / baseline_score * 100) if baseline_score else 0
        print(f"    + {mode:<20} -> {score:.3f}  ({pct:+.1f}%)")
        prev_score = score

    # Per-category best mode
    print(f"\nPer-category best modes:")
    categories = set()
    for results in all_results.values():
        for r in results:
            categories.add(r["category"])

    for cat in sorted(categories):
        best_mode, best_score = None, 0
        baseline_cat = 0
        for mode, results in all_results.items():
            cat_results = [r for r in results if r["category"] == cat]
            if not cat_results:
                continue
            avg = sum(r["scores"]["composite"] for r in cat_results) / len(cat_results)
            if mode == modes[0]:
                baseline_cat = avg
            if avg > best_score:
                best_mode, best_score = mode, avg
        print(f"  {cat:<20} best={best_mode:<15} score={best_score:.3f} "
              f"(baseline={baseline_cat:.3f})")

    # Incremental layer contributions
    print(f"\nLayer contributions:")
    if len(modes) >= 2:
        rag_lift = mode_avgs.get(modes[1], 0) - baseline_score
        print(f"  RAG/retrieval lift:  +{rag_lift:.3f}")
    if len(modes) >= 3:
        rules_lift = mode_avgs.get(modes[2], 0) - mode_avgs.get(modes[1], 0)
        print(f"  Rules/protocol lift: +{rules_lift:.3f}")
    if len(modes) >= 4:
        extra_lift = mode_avgs.get(modes[3], 0) - mode_avgs.get(modes[2], 0)
        print(f"  Additional lift:     +{extra_lift:.3f}")
```

## Step 5: Per-Dimension Analysis

Break down results by scoring dimension to find where each mode wins and loses.

```python
def dimension_breakdown(all_results: dict, modes: list):
    """Show per-dimension averages across modes."""
    # Collect all dimension names
    dims = set()
    for results in all_results.values():
        for r in results:
            dims.update(k for k, v in r["scores"].items()
                       if isinstance(v, (int, float)) and k != "composite")

    print(f"\n{'Dimension':<25}", end="")
    for mode in modes:
        print(f" {mode:>12}", end="")
    print()
    print("-" * (25 + 13 * len(modes)))

    for dim in sorted(dims):
        print(f"  {dim:<23}", end="")
        for mode in modes:
            results = all_results[mode]
            vals = [r["scores"].get(dim) for r in results
                    if isinstance(r["scores"].get(dim), (int, float))]
            avg = sum(vals) / len(vals) if vals else 0
            print(f" {avg:>10.3f}  ", end="")
        print()
```

## Step 6: Save Results and Run

```python
def save_results(all_results: dict, modes: list, output_dir: str):
    """Save full results as JSON for later analysis."""
    import json
    from datetime import datetime
    from pathlib import Path

    out_dir = Path(output_dir)
    out_dir.mkdir(parents=True, exist_ok=True)

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    out_file = out_dir / f"ablation_{timestamp}.json"

    mode_avgs = {
        mode: sum(r["scores"]["composite"] for r in results) / len(results)
        for mode, results in all_results.items()
    }

    with open(out_file, "w") as f:
        json.dump({
            "timestamp": timestamp,
            "modes": modes,
            "task_count": len(next(iter(all_results.values()))),
            "mode_averages": mode_avgs,
            "results": {
                m: [{k: v for k, v in r.items() if k != "reply"} for r in results]
                for m, results in all_results.items()
            },
        }, f, indent=2, default=str)

    print(f"\nResults saved to: {out_file}")


# CLI entry point
if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode", default="all",
                        choices=["baseline", "rag", "full", "grouped", "all"])
    parser.add_argument("--limit", type=int, default=None)
    parser.add_argument("--category", default=None)
    parser.add_argument("--judge", action="store_true",
                        help="Enable LLM judge scoring")
    args = parser.parse_args()

    modes = ["baseline", "rag", "full", "grouped"] if args.mode == "all" else [args.mode]
    tasks = load_tasks(category=args.category, limit=args.limit)
    llm = get_llm()
    judge_llm = get_llm() if args.judge else None

    results = run_ablation(tasks, modes, llm, judge_llm)
    print_ablation_report(results, modes)
    dimension_breakdown(results, modes)
    save_results(results, modes, "eval_results/")
```

## Interpreting Results

### What good results look like

```
Stack decomposition:
  Baseline (0.578)
    + rag               -> 0.681  (+17.8%)   [evidence grounding]
    + full              -> 0.711  (+2.2%)    [safety guardrails]
    + grouped           -> 0.710  (+22.8%)   [broader coverage]
```

Each layer should show additive improvement. If a layer shows negative lift, it's hurting more than helping.

### Key things to look for

1. **Dominant layer** -- which single component adds the most value? That's your highest-ROI investment.
2. **Negative lift** -- a component that reduces scores should be investigated. It may interfere with the LLM's natural reasoning.
3. **Category-specific winners** -- different modes may win for different task types. This justifies domain-adaptive routing (use different configs for different query types).
4. **Dimension trade-offs** -- a component might boost safety but reduce accuracy. Decide if that trade-off is acceptable.
5. **Diminishing returns** -- if layers 2+ each add less than 3%, consider if the complexity is worth it.

## Honest Assessment Checklist

After running an ablation study, answer these honestly in your report:

- [ ] Are absolute scores good enough, or are you just measuring "less bad"?
- [ ] Is the baseline model too weak? (RAG compensating for model limitations =/= RAG being valuable)
- [ ] Is N large enough for statistical significance? (n<100 with no confidence intervals = noisy)
- [ ] Did judge calibration stay consistent across runs?
- [ ] Are small lifts (1-3%) within noise, or are they real?
- [ ] Would a bigger model make some augmentation layers unnecessary?

## Anti-Patterns to Avoid

- **Comparing across different judge calibrations** -- if the judge LLM or prompt changes between runs, scores are not comparable. Pin your judge.
- **Celebrating relative lifts on bad baselines** -- "+23% improvement" sounds great until you realize 0.578 -> 0.710 means "below adequate" to "adequate."
- **No per-category breakdown** -- aggregate scores hide that a component helps training queries but hurts injury queries.
- **Too many modes at once** -- start with baseline vs. one augmentation. Add modes incrementally. Testing 8 modes with 20 tasks produces noise, not signal.
- **Not saving raw results** -- always persist full results to JSON. You will want to re-analyze without re-running.
