---
name: interview-belief-extractor
description: Extracts structured belief profiles from interview transcripts with behavioral confidence scoring. Computes confidence from hedging language, booster counts, counter-argument fluency, and caveats rather than self-report. Produces belief profiles with reasoning origins, identity centrality, and openness to revision. Use after completing any interview, survey, or structured conversation where you need to analyze what someone believes and how confidently they hold it.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: interview
  tags: [interview, belief-extraction, confidence-scoring, analysis, behavioral-signals, nlp]
---

# Interview Belief Extractor

Processes an interview transcript and produces structured belief profiles with behaviorally-derived confidence scores. Instead of asking people how confident they are (unreliable), this skill measures confidence from how they actually talk.

## When to Use

After any interview, conversation, or transcript where you need to understand:
- What does this person actually believe?
- How confident are they (measured behaviorally, not self-reported)?
- Where did the belief come from?
- How central is it to their identity?
- How open are they to changing their mind?

## Phase 1: Belief Identification

Read the full transcript. For each distinct belief, claim, or position expressed:

1. **belief_summary**: A clear statement of what the person believes, using their words where possible
2. **reasoning_origins**: Every origin MUST be grounded in something they actually said. No origin without a direct quote as evidence.

### Reasoning Origin Categories

| Origin | Description | Example Signal |
|--------|-------------|----------------|
| `personal_experience` | They lived through something relevant | "When I was at Company X, I saw..." |
| `evidence_reasoning` | They cite data, studies, or logical arguments | "The research shows..." |
| `social_identity` | Belief tied to group membership | "As an engineer, we think..." |
| `authority_testimony` | Trusting an expert or authority figure | "My professor always said..." |
| `social_transmission` | Absorbed from social environment | "Everyone in my circle believes..." |
| `intuition_affect` | Gut feeling or emotional response | "It just feels right" |
| `heuristic_default` | Default position, not deeply examined | "That's just how it works" |

For each origin, assign a `weight` (0.0-1.0) representing its relative contribution. Weights should roughly sum to 1.0 across all origins for a given belief.

## Phase 2: Behavioral Confidence Scoring

This is the core innovation. Never ask for a confidence number. Instead, measure it from language patterns.

### Step 1: Count Signals from the Transcript

**Hedges** (reduce confidence score): "I think," "maybe," "probably," "not sure," "kind of," "I guess," "sort of," "it seems like," "I feel like"

**Boosters** (increase confidence score): "definitely," "obviously," "clearly," "absolutely," "100%," "no question," "I know," "without a doubt," "for sure"

Record the exact phrases found and their counts.

**Counter-argument fluency** — How well can they articulate the opposing view?
- `none`: Could not generate any counter-argument when asked
- `weak`: Vague or strawman counter-argument
- `moderate`: Reasonable counter-argument but dismissed it
- `strong`: Articulate, steelmanned counter-argument they genuinely engaged with

**Updating response** — How did they respond to challenges or hypotheticals?
- `not_tested`: No challenge was presented during the conversation
- `dismissed`: Deflected or rejected the challenge without engagement
- `engaged`: Considered the challenge seriously, even if they did not change their mind
- `updated`: Actually modified their position in response

### Step 2: Compute Sub-Scores (each 0-100, where 100 = maximum confidence)

**a) Hedge/Booster Score (weight: 25%)**
- Both counts zero: 50 (neutral)
- Only hedges: `max(10, 50 - (hedge_count * 10))`
- Only boosters: `min(90, 50 + (booster_count * 10))`
- Both present: `round(50 + ((booster_count - hedge_count) / (booster_count + hedge_count)) * 40)`

**b) Counter-Argument Score (weight: 30%)**
Strong counter-argument fluency means LOWER confidence (they see the other side clearly):
- `none`: 85
- `weak`: 70
- `moderate`: 45
- `strong`: 25

**c) Updating Score (weight: 25%)**
- `dismissed`: 85
- `not_tested`: 50
- `engaged`: 40
- `updated`: 15

**d) Qualification Score (weight: 10%)**
Count caveats like "it depends," "in some cases," "generally but not always," "I could be wrong":
- 0 caveats: 80
- 1-2 caveats: 55
- 3-4 caveats: 35
- 5+ caveats: 15

**e) Self-Reported Score (weight: 10%)**
- User volunteered a number ("I'm about 80% sure"): use that number
- Strong certainty language without a number: 75
- Expressed uncertainty without a number: 30
- No signal: 50

### Step 3: Compute Final Score

```
derived_score = round(
    hedge_booster_score * 0.25 +
    counter_argument_score * 0.30 +
    updating_score * 0.25 +
    qualification_score * 0.10 +
    self_reported_score * 0.10
)
```

### Step 4: Show All Work

Always write out the full scoring reasoning: each sub-score with its inputs, then the weighted sum. This is required for auditability.

## Phase 3: Additional Dimensions

### Identity Centrality (1-5)

Based on language patterns observed in the transcript:
- "We" vs "I" language (group identification)
- Emotional intensity when the belief is probed
- Moral loading ("wrong," "evil," "sacred," "right thing to do")
- References to communities, groups, or movements
- 1 = no identity connection, 3 = moderate, 5 = core identity

### Openness to Revision (1-5)

Based on:
- Counter-argument engagement quality
- Updating behavior when challenged
- Number of qualifications and caveats
- Expressed curiosity about alternatives
- 1 = closed/rigid, 3 = moderate, 5 = very open

### Key Quotes

Select 3-5 most revealing quotes from the interviewee. Prioritize quotes that show reasoning process, confidence signals, or identity connections over generic statements.

## Phase 4: Insight Generation

After extracting all belief profiles, generate insights:

### Per-Belief Insights
- **Origin explanation**: Where does this belief appear to come from? Ground it in what was said. No flattery.
- **Confidence observation**: Is there a gap between stated confidence and evidence basis? Is confidence well-calibrated? Always provide an observation.

### Cross-Belief Patterns
- **Reasoning style**: What reasoning approach does this person tend to use?
- **Confidence calibration**: Are they systematically over- or under-confident?
- **Identity vs evidence tensions**: Where do identity-driven beliefs conflict with evidence-driven ones?

### Closing Reflection
One or two sentences. Direct observation. No effusive praise.

## Output Schema

```json
{
  "beliefs": [
    {
      "belief_summary": "...",
      "reasoning_origins": [
        {"origin": "personal_experience", "weight": 0.6, "evidence": "exact quote"}
      ],
      "confidence": {
        "hedge_count": 3,
        "booster_count": 1,
        "hedges_found": ["I think", "maybe", "sort of"],
        "boosters_found": ["definitely"],
        "counter_argument_fluency": "moderate",
        "updating_response": "engaged",
        "scoring_reasoning": "hedge_booster: 50+((1-3)/(1+3))*40=30...",
        "derived_score": 42
      },
      "identity_centrality": 2,
      "openness_to_revision": 4,
      "key_quotes": ["..."],
      "reasoning_chain": "Step-by-step reasoning for all classifications"
    }
  ],
  "insights": {
    "per_belief": [
      {"belief_summary": "...", "origin_explanation": "...", "confidence_observation": "..."}
    ],
    "overall_patterns": [
      {"type": "reasoning_style", "observation": "..."}
    ],
    "closing_reflection": "..."
  }
}
```

## Adapting to Different Domains

This extraction pattern works for any transcript where someone expresses positions:
- **User research**: Extract user needs/preferences instead of beliefs
- **Expert interviews**: Extract knowledge claims and certainty levels
- **Customer feedback**: Extract satisfaction beliefs and pain point confidence
- **Debate analysis**: Extract positions from multiple speakers

The behavioral confidence scoring is the transferable core. Anywhere you need to measure how strongly someone holds a position without asking them directly, use the hedge/booster/counter-argument/updating framework.
