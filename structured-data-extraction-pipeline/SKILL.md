---
name: structured-data-extraction-pipeline
description: Build an LLM-based extraction system that converts free-form user input into validated, structured records with alias resolution and domain-specific rules. Use when building any system that needs to parse natural language into database-ready structured data — food logs, exercise logs, expense tracking, medical intake, etc.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: data-pipelines
  tags: [extraction, llm, validation, canonicalization, structured-data, parsing, nlp]
---

# Structured Data Extraction Pipeline

Converts unstructured natural-language input into validated, structured records using LLM extraction with prompt templates, alias resolution (canonicalization), domain-specific validation rules, and follow-up handling for missing fields.

## Architecture Overview

The pipeline has 4 layers:

```
User Input (free text)
    |
    v
[1. Pre-validation] — reject ambiguous/garbage input early
    |
    v
[2. LLM Extraction] — prompt template extracts structured JSON
    |
    v
[3. Canonicalization] — resolve aliases to canonical names
    |
    v
[4. Post-validation] — detect missing fields, trigger follow-ups
    |
    v
Structured Record → DB
```

## Step 1: Define Your Domain Schema

Every extraction domain needs a clear output schema. Define every field, its type, and whether it's required or optional.

```python
# Example: a generic "event log" extraction schema
EVENT_SCHEMA = {
    "event_name": str,       # required — what happened
    "category": str,         # required — classification
    "quantity": float,       # optional — numeric value if applicable
    "unit": str,             # optional — unit of measurement
    "intensity": str,        # optional — "low", "medium", "high"
    "timestamp": datetime,   # derived — parsed from text or defaulted to now
    "notes": str,            # optional — any extra context
}
```

## Step 2: Build the LLM Extraction Prompt

The prompt is the core of the pipeline. It must include:
- Clear instructions on what to extract
- The exact JSON output format with field names and types
- Critical rules for edge cases (quantities, multiples, ambiguity)
- 2-3 concrete examples showing input-to-output mapping

```python
from langchain.prompts import ChatPromptTemplate

EXTRACTION_PROMPT = ChatPromptTemplate.from_template(
    """
Extract ALL items from the user's input as a JSON array.

User input: {input}

CRITICAL RULES:
- Multiple items = separate objects for EACH
- Always provide realistic estimates (never use 0.0 for known quantities)
- If quantity isn't stated, infer a standard default
- QUANTITY MATTERS: "3 cups" = triple the values of "1 cup"

Output format (JSON array only, no markdown):
[
  {{"name": "item name with quantity", "value": 100.0, "category": "type", "unit": "kg"}}
]

Example - "2 meetings and a phone call":
[
  {{"name": "2 meetings", "value": 2.0, "category": "meeting", "unit": "count"}},
  {{"name": "phone call", "value": 1.0, "category": "call", "unit": "count"}}
]
"""
)
```

Key prompt design principles:
- **Explicitly handle multiples** — LLMs often ignore quantity multipliers unless told
- **Provide defaults** — tell the LLM what to do when information is missing
- **Show edge cases in examples** — the examples are the strongest steering mechanism
- **Output JSON only** — explicitly say "no markdown" to prevent ```json``` wrapping

## Step 3: Pre-Validation (Reject Ambiguous Input)

Before calling the LLM, reject inputs that are too vague to extract:

```python
MAX_INPUT_LEN = 5000

def extract_items(llm, input_text: str, user_id: str):
    input_text = input_text[:MAX_INPUT_LEN]
    input_lower = input_text.lower()

    # Reject ambiguous input early — saves an LLM call
    ambiguous_patterns = [
        "something from", "forgot what", "don't remember", "not sure what"
    ]
    if any(p in input_lower for p in ambiguous_patterns):
        return {
            "status": "ambiguous",
            "message": "Please provide more details before logging."
        }

    # Proceed with LLM extraction
    resp = (EXTRACTION_PROMPT | llm).invoke({"input": input_text}).content
    items = extract_json(resp, is_array=True)
    ...
```

## Step 4: Natural Language Date/Time Parsing

Many extraction domains need date parsing from phrases like "yesterday", "last Tuesday", "2 weeks ago":

```python
import re
from datetime import datetime, timedelta
from dateutil import parser as dateutil_parser

WEEKDAYS = ['monday', 'tuesday', 'wednesday', 'thursday', 'friday',
            'saturday', 'sunday']

def parse_date_reference(text: str) -> datetime:
    """Parse natural language date references. Falls back to now."""
    text_lower = text.lower()
    now = datetime.utcnow()

    if "yesterday" in text_lower:
        return now - timedelta(days=1)

    days_match = re.search(r'(\d+)\s*days?\s*ago', text_lower)
    if days_match:
        return now - timedelta(days=int(days_match.group(1)))

    # Try explicit date patterns
    date_patterns = [
        r'(\d{1,2}[/-]\d{1,2}[/-]\d{2,4})',
        r'(\d{4}[/-]\d{1,2}[/-]\d{1,2})',
    ]
    for pattern in date_patterns:
        match = re.search(pattern, text_lower)
        if match:
            try:
                parsed = dateutil_parser.parse(match.group(1), fuzzy=True)
                if parsed > now:
                    parsed = parsed.replace(year=parsed.year - 1)
                return parsed
            except (ValueError, TypeError):
                continue

    return now
```

## Step 5: Canonical Name Resolution (Alias Lookup)

When users refer to the same entity by different names, resolve to a canonical form. Use a priority-based lookup:

```python
def resolve_canonical_name(raw_name: str, db_conn) -> str | None:
    """Resolve a raw name to its canonical entry.

    Priority: exact alias match > exact name match > display name > substring.
    Returns None if no match found.
    """
    if not raw_name:
        return None

    normalized = raw_name.strip().lower()

    with db_conn.cursor() as c:
        c.execute(
            """SELECT name, priority FROM (
                 SELECT canonical_name AS name, 1 AS priority
                   FROM aliases WHERE LOWER(alias) = %s
                 UNION ALL
                 SELECT name, 2 AS priority
                   FROM entities WHERE LOWER(name) = %s
                 UNION ALL
                 SELECT name, 3 AS priority
                   FROM entities WHERE LOWER(display_name) = %s
                 UNION ALL
                 SELECT name, 4 AS priority
                   FROM entities WHERE LOWER(display_name) LIKE %s
               ) sub
               ORDER BY priority, LENGTH(name) ASC
               LIMIT 1""",
            (normalized, normalized, normalized, f"%{normalized}%"),
        )
        row = c.fetchone()
        return row[0] if row else None
```

This pattern requires two tables:
- **entities** — the canonical list (`name`, `display_name`, metadata)
- **aliases** — maps variant names to canonical (`alias` -> `canonical_name`)

## Step 6: Post-Validation and Follow-Up Detection

After extraction, detect what's missing and queue follow-up questions:

```python
def detect_missing_fields(items: list[dict]) -> list[str]:
    """Check extracted items for missing important fields."""
    follow_ups = []

    missing_quantity = [i for i in items if not i.get("quantity")]
    if missing_quantity:
        names = [i["name"] for i in missing_quantity][:3]
        follow_ups.append(f"quantity for {', '.join(names)}")

    missing_category = [i for i in items if not i.get("category")]
    if missing_category:
        names = [i["name"] for i in missing_category][:3]
        follow_ups.append(f"category for {', '.join(names)}")

    return follow_ups

# In the main extraction function:
result = {"status": "ok", "items": items, "logged_ids": logged_ids}
follow_ups = detect_missing_fields(items)
if follow_ups:
    result["follow_up"] = follow_ups
return result
```

## Step 7: Follow-Up Update Handler

When the user responds to follow-up questions, extract ONLY the missing fields and update existing records:

```python
FOLLOW_UP_PROMPT = ChatPromptTemplate.from_template(
    """The user was asked follow-up questions about a previous entry.
Extract ONLY the answers to the missing fields.

Missing fields: {missing}
User's answer: {answer}

Map answers to these EXACT column names (no other keys allowed):
{allowed_columns}

Output JSON only:
{{"field_name": value}}
"""
)

def update_from_follow_up(llm, answer: str, context: dict) -> dict:
    ids = context.get("ids", [])
    missing = context.get("missing", [])

    if not ids or not missing:
        return {"status": "no_context"}

    resp = (FOLLOW_UP_PROMPT | llm).invoke({
        "missing": ", ".join(missing),
        "answer": answer,
        "allowed_columns": ", ".join(ALLOWED_COLUMNS),
    }).content

    extracted = extract_json(resp, is_array=False)

    # Filter to allowed columns only
    updates = {k: v for k, v in extracted.items()
               if k in ALLOWED_COLUMNS and v is not None}

    # Coerce numeric columns
    for col in list(updates):
        if col in NUMERIC_COLUMNS and not isinstance(updates[col], (int, float)):
            nums = re.findall(r"[\d.]+", str(updates[col]))
            updates[col] = float(nums[0]) if nums else None

    # Apply updates to DB
    with db_conn.cursor() as c:
        for entry_id in ids:
            set_parts = [f"{col} = %s" for col in updates]
            c.execute(
                f"UPDATE {table} SET {', '.join(set_parts)} WHERE id = %s",
                list(updates.values()) + [entry_id],
            )

    return {"status": "ok", "fields_updated": list(updates.keys())}
```

## Step 8: Keyword-Based Routing (Multi-Domain)

When your system handles multiple extraction types, route based on keywords:

```python
def dispatch_extraction(llm, input_text: str, user_id: str):
    """Route to the correct extraction tool based on input keywords."""
    text_lower = input_text.lower()

    # Define keyword sets per domain
    domain_keywords = {
        "cardio": ["ran", "run", "mile", "km", "cycling", "swim"],
        "nutrition": ["ate", "breakfast", "lunch", "dinner", "snack"],
        "workout": [],  # default fallback
    }

    for domain, keywords in domain_keywords.items():
        if any(kw in text_lower for kw in keywords):
            return EXTRACTORS[domain](llm, input_text, user_id)

    return EXTRACTORS["default"](llm, input_text, user_id)
```

## JSON Extraction Helper

Always use a robust JSON extractor that handles markdown wrapping and partial responses:

```python
import json
import re

def extract_json(text: str, is_array: bool = False):
    """Extract JSON from LLM response, handling markdown code blocks."""
    # Strip markdown code blocks
    text = re.sub(r'```(?:json)?\s*', '', text).strip()
    text = text.rstrip('`').strip()

    try:
        result = json.loads(text)
        if is_array and not isinstance(result, list):
            result = [result]
        return result
    except json.JSONDecodeError:
        # Try to find JSON within the text
        pattern = r'\[.*\]' if is_array else r'\{.*\}'
        match = re.search(pattern, text, re.DOTALL)
        if match:
            return json.loads(match.group())
        return [] if is_array else {}
```

## Common Pitfalls

1. **LLMs ignore quantities** — Always include explicit quantity multiplication rules and examples in the prompt
2. **0.0 defaults** — LLMs return zeros when unsure; instruct them to estimate or use null instead
3. **Markdown wrapping** — LLMs wrap JSON in ```json blocks; always strip these
4. **Ambiguous input wastes tokens** — Pre-filter vague inputs before calling the LLM
5. **Follow-up field mapping** — LLMs invent descriptive key names like `reps_for_bench_press` instead of `reps`; always filter to allowed columns
6. **Numeric coercion** — LLMs return `"8 ounces"` instead of `226.8`; extract the number with regex
