---
name: project-insights
description: Deep analysis of any project using journal entries, repo files, and strategy docs. Use when user says "give me insights on", "analyze project", "how is X going", "project status", "what's stalled", or asks about a specific project's health.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: analysis
  tags: [projects, insights, analysis, strategy]
---

# Project Insights

Delivers deep analysis of any project by cross-referencing journal entries, repo files, and strategy docs.

## Instructions

### Step 1: Identify the project

Match the user's request to a project. If ambiguous, ask.

### Step 2: Gather all data

1. **Journal entries**: Read ALL entries for the project from your journal file (filter by `project:<slug>` tag)
2. **Repo files**: Read the project's repo — strategy docs, PLAN.md, FEATURES.md, README, CLAUDE.md
3. **Principles**: Check any team/personal principles docs for anti-pattern checks

### Step 3: Analyze across 5 dimensions

#### 1. Momentum
- Frequency and recency of entries
- Regular work vs sporadic bursts
- Shipped features with real user impact

#### 2. Stalls
- No entries in 14+ days
- Open decisions older than 14 days without resolution
- Action items from brainstorms that never got started

#### 3. Repeated Patterns
- Same anti-pattern tags appearing multiple times
- Similar decisions being re-made (indecision loop)
- Brainstorms without follow-through (idea accumulation without execution)

#### 4. Strategy Contradictions
- What journal says vs what strategy docs say
- Stated priorities vs actual work logged
- Common anti-patterns: infrastructure trap, feature overload, validation avoidance, perfection loop

#### 5. Gaps
- No validation entries (who have you talked to?)
- No user feedback logged
- No marketing or distribution work
- No revenue model discussion

### Step 4: Deliver the analysis

Be specific — cite journal entries by date and content. No vague observations.

```
## [Project Name] — Deep Analysis

### Momentum: [Good/Slowing/Stalled]
[Specific evidence with dates]

### Key Wins
- [Dated, specific wins]

### Concerns
- [Dated, specific concerns with evidence]

### Anti-Pattern Check
- [Any violations detected]

### Recommended Actions
1. [Most impactful action]
2. [Second action]
3. [Third action]
```

## Honesty Rules

- Challenge bad reasoning, don't just record it
- Flag when the user is repeating a known anti-pattern
- Push back when something contradicts their own strategy docs
- "What are you missing?" is always a valid question
