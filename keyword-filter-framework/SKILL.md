---
name: keyword-filter-framework
description: High-performance keyword-based content filtering with compiled regex, bot/spam exclusion, configurable thresholds, and quality scoring. Use when building any text classification or filtering pipeline — content moderation, comment filtering, relevance scoring, spam detection, domain-specific text selection.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: miscellaneous
  tags: [filtering, keyword-matching, regex, text-classification, quality-scoring, content-moderation]
---

# Keyword Filter Framework

A pattern for building fast, maintainable keyword-based content filters. Combines compiled regex for speed, layered rejection rules, and a scoring function for ranking passing items.

## When to Use

Any time you need to filter a stream of text items (comments, articles, messages, reviews, documents) based on domain relevance, quality, and source trustworthiness. This is the right pattern when:
- You have a known vocabulary for your domain
- You need to exclude known-bad sources (bots, spam accounts)
- You want both a binary pass/fail AND a quality ranking
- Performance matters (thousands+ items)

## Architecture: Three-Layer Filter

Every filter follows the same structure: **exclude bad sources, reject low-quality items, require domain signal**.

### Layer 1: Source Exclusion

Maintain a set of known-bad sources to skip entirely. These never change per-item — check them first for early exit.

```python
EXCLUDED_SOURCES = {
    "AutoModerator",
    "SpamBot",
    "[deleted]",
    "[removed]",
    # Add domain-specific exclusions
}

def _is_excluded_source(item):
    author = item.get("author", "")
    return author in EXCLUDED_SOURCES or author == ""
```

Using a `set` gives O(1) lookup. Always check this first — it is the cheapest rejection.

### Layer 2: Low-Effort Rejection

Reject items that are too short, match known low-effort patterns, or are obviously off-topic. Use pre-compiled regex patterns for speed.

```python
import re

MIN_LENGTH = 50  # Tune per domain

LOW_EFFORT_PATTERNS = [
    re.compile(r"^(nice|cool|awesome|love it|wow|lol|same|thanks)[\.\!\?\s]*$", re.IGNORECASE),
    re.compile(r"^(where did you (get|buy|find)|what is that|source|link)\b", re.IGNORECASE),
    re.compile(r"^https?://", re.IGNORECASE),  # bare links
]

def _is_low_effort(text):
    if len(text) < MIN_LENGTH:
        return True
    for pattern in LOW_EFFORT_PATTERNS:
        if pattern.match(text):
            return True
    return False
```

Key decisions:
- `MIN_LENGTH`: Start at 50, adjust based on your domain. Shorter for tweets, longer for articles.
- Use `pattern.match()` (anchored to start) for patterns that describe the entire message.
- Use `pattern.search()` for patterns that can appear anywhere.

### Layer 3: Domain Keyword Requirement

The item must contain at least one domain-relevant keyword. Compile all keywords into a single regex for performance.

```python
DOMAIN_KEYWORDS = [
    # Group by sub-category for maintainability
    # Category A
    "keyword1", "keyword2", "multi-word phrase",
    # Category B
    "keyword3", "keyword4",
]

# Compile once at module load — not per call
_KEYWORD_PATTERN = re.compile(
    "|".join(re.escape(kw) for kw in DOMAIN_KEYWORDS),
    re.IGNORECASE,
)

def _has_domain_signal(text):
    return bool(_KEYWORD_PATTERN.search(text))
```

Why one compiled regex instead of looping:
- Single pass through the text instead of N passes
- `re.compile` with alternation is optimized by Python's regex engine
- `re.escape` handles special characters in keywords safely

### Putting It Together

```python
def passes_filter(item):
    """Returns True if item passes all filter layers."""
    if _is_excluded_source(item):
        return False

    text = item.get("body", "") or item.get("text", "") or item.get("content", "")

    if text in ("[deleted]", "[removed]", ""):
        return False

    if _is_low_effort(text):
        return False

    if not _has_domain_signal(text):
        return False

    return True
```

The order matters: cheapest checks first, most expensive last.

## Quality Scoring

Items that pass the filter can be ranked. A good scoring function combines:

1. **Keyword density** — More domain keywords = more relevant
2. **Length bonus** — Longer items carry more signal (with diminishing returns)
3. **External signal** — Upvotes, likes, citations, or any platform-provided quality metric

```python
def score_item(item):
    """Score a passing item for ranking. Higher = better."""
    text = item.get("body", "")
    external_score = max(item.get("score", 1), 1)  # Floor at 1

    # Count keyword matches
    keyword_hits = len(_KEYWORD_PATTERN.findall(text))

    # Length bonus with diminishing returns (cap at 2x)
    length_score = min(len(text) / 300, 2.0)

    # Weighted combination
    return keyword_hits * 2 + length_score + (external_score ** 0.3)
```

Tuning notes:
- The `** 0.3` on external score compresses outliers (a 1000-upvote comment is not 1000x better than a 1-upvote comment)
- Adjust the `300` divisor based on typical item length in your domain
- Keyword hits get the highest weight because domain relevance is the primary signal

## Customization Guide

### Adapting to a New Domain

1. **Define your keyword list.** Group keywords by sub-category for readability. Aim for 30-100 keywords. Too few = low recall, too many = low precision.
2. **Define your excluded sources.** Known bots, spam accounts, system accounts.
3. **Define your low-effort patterns.** What does a useless item look like in your domain?
4. **Set MIN_LENGTH.** Look at 20 good items and 20 bad items — find the length threshold that separates them.
5. **Tune scoring weights.** Run on 100 items, sort by score, check if the top 10 are actually the best. Adjust weights.

### Example Domains

**Code review comments:**
- Keywords: "refactor", "bug", "performance", "readable", "test", "edge case", "null check"
- Exclude: CI bots, auto-generated comments
- Low effort: "LGTM", "+1", "nit"

**Product reviews:**
- Keywords: "battery", "screen", "camera", "build quality", "price", "worth", "compared to"
- Exclude: incentivized reviewers, duplicate accounts
- Low effort: "Great product!", "5 stars", single emoji

**Research paper relevance:**
- Keywords: domain-specific terms from your research area
- Exclude: retracted papers, predatory publishers
- Low effort: abstracts under 100 words

## Self-Test Pattern

Always include a self-test at the bottom of your filter module. Define items that should pass and items that should fail, then assert.

```python
if __name__ == "__main__":
    test_items = [
        {"body": "Nice!", "author": "user1", "score": 5},         # too short
        {"body": "Bot output.", "author": "SpamBot", "score": 0},  # excluded source
        {"body": "A substantive comment with keyword1 and detailed reasoning about the topic at hand.", "author": "real_user", "score": 15},  # PASS
    ]

    for item in test_items:
        result = passes_filter(item)
        score = score_item(item) if result else 0
        preview = item["body"][:60]
        print(f"  [{'PASS' if result else 'SKIP'}] (score={score:.1f}) {preview}")

    # Assert expected results
    assert passes_filter(test_items[2]), "Good item should pass"
    assert not passes_filter(test_items[0]), "Short item should fail"
    assert not passes_filter(test_items[1]), "Bot should fail"
```

This catches regressions when you add or modify keywords. Run it every time you change the filter.
