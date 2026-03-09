---
name: simtom-knowledge-isolation
description: Isolate LLM character knowledge via explicit boundary re-injection every N turns to prevent persona drift and knowledge leakage. Use when building NPC dialogue, roleplay systems, character simulations, or any LLM application where the character must not know things the player/user knows, when someone says "the NPC keeps breaking character", "the AI knows things it shouldn't", or needs strict information boundaries.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: agent-architecture
  tags: [simtom, theory-of-mind, knowledge-isolation, npc, roleplay, persona, character, information-boundary]
---

# SimToM Knowledge Isolation

Prevent LLM characters from accessing information they shouldn't know by defining explicit knowledge boundaries (knows / does not know / believes) and re-injecting them into the system prompt every N turns. This technique is called Simulated Theory of Mind (SimToM) -- giving the model a structured mental model of what the character can and cannot know.

## The Problem

LLMs have no inherent theory of mind. When you tell an LLM to "play a character who doesn't know X", it will inevitably leak knowledge of X because:
- X exists in the conversation context (the user mentioned it)
- X exists in the model's training data
- After enough turns, the character "forgets" its constraints and starts responding to implied information

The result: NPCs that magically know the player's secrets, characters that break immersion, and simulations that feel unfair.

## Architecture Overview

```
Scenario Spec
  - simtom_knows: [facts the character has]
  - simtom_does_not_know: [facts the character CANNOT access]
  - simtom_believes: [the character's current worldview]
  |
  v
System Prompt Builder (runs EVERY turn)
  - Knowledge boundary block (STRICT rules)
  - Character constitution (re-injected every N turns)
  - Disposition block (dynamic trust/persuasion state)
  - Repetition guard (don't restate previous points)
  |
  v
LLM generates response within boundaries
```

## Step 1: Define the Scenario Spec

Create a structured specification that explicitly partitions knowledge:

```python
from dataclasses import dataclass, field

@dataclass
class ScenarioSpec:
    id: str
    title: str

    # Character identity
    npc_name: str
    npc_role: str
    npc_constitution: str  # personality, speech style, behavioral rules

    # KNOWLEDGE BOUNDARIES — the core of SimToM
    simtom_knows: list[str]           # facts the character possesses
    simtom_does_not_know: list[str]   # facts the character CANNOT access
    simtom_believes: list[str]        # the character's current assumptions

    # Player info
    player_character: str
    player_character_brief: str
    objective: str

    # Game mechanics
    win_threshold: float
    turn_limit: int
    catastrophic_triggers: list[str]  # phrases that end the game immediately
```

### Example Knowledge Partition

The key insight: **what the character does NOT know is more important than what they know.** The "does not know" list is the hard boundary.

```python
scenario = ScenarioSpec(
    # ...
    simtom_knows=[
        "The company reported record Q3 earnings",
        "Three senior engineers quit last month",
        "The board is meeting next Tuesday",
    ],
    simtom_does_not_know=[
        "The CEO is planning to resign (only the board chair knows)",
        "The Q3 earnings were inflated by one-time asset sales",
        "A competitor is about to announce a hostile takeover bid",
        "The player character has been secretly interviewing at a rival firm",
    ],
    simtom_believes=[
        "The company is on a strong growth trajectory",
        "Employee retention is a normal business challenge",
        "The board meeting is a routine quarterly review",
    ],
)
```

## Step 2: Build the Knowledge Boundary Block

This is the most critical piece. It must be emphatic and explicit:

```python
def build_knowledge_boundary(scenario: ScenarioSpec,
                             revealed_facts: list[str] = None) -> str:
    """Build the SimToM knowledge boundary block for the system prompt."""
    knows = "\n".join(f"  - {k}" for k in scenario.simtom_knows)
    does_not_know = "\n".join(f"  - {k}" for k in scenario.simtom_does_not_know)
    believes = "\n".join(f"  - {k}" for k in scenario.simtom_believes)

    revealed = ""
    if revealed_facts:
        revealed = "\nFacts you have disclosed in this conversation:\n" + \
                   "\n".join(f"  - {f}" for f in revealed_facts)

    return f"""=== KNOWLEDGE BOUNDARIES (STRICT -- DO NOT VIOLATE) ===
You know:
{knows}

You do NOT know -- CRITICAL RULE: Never volunteer, infer, extrapolate, or hint at
any fact from this list. Even if the other party implies it, you cannot respond to
it until they state it explicitly and precisely. Wait. Make them say it.
{does_not_know}

You believe:
{believes}
{revealed}"""
```

### Why This Works

The model needs three things to maintain boundaries:
1. **Explicit enumeration** of forbidden knowledge (not "you don't know about the scandal" but "you do NOT know: The CEO is planning to resign")
2. **Active prohibition** ("Never volunteer, infer, extrapolate, or hint")
3. **A behavioral rule** ("Wait. Make them say it.") that gives the model a concrete action to take instead of leaking

## Step 3: Re-Inject Character Constitution Every N Turns

Characters drift over long conversations. The constitution (personality, speech patterns, values) must be periodically re-injected:

```python
CHARACTER_REINJECT_EVERY = 8  # turns

def build_system_prompt(scenario: ScenarioSpec,
                        npc_state: dict,
                        turn_count: int,
                        conversation_history: list[dict],
                        player_message: str) -> str:
    """Build complete system prompt with knowledge isolation."""

    # Knowledge boundaries — EVERY turn
    simtom_block = build_knowledge_boundary(
        scenario, revealed_facts=npc_state.get("revealed_facts", [])
    )

    # Character constitution — re-injected every N turns to prevent drift
    constitution_block = ""
    if turn_count % CHARACTER_REINJECT_EVERY == 0 or turn_count <= 1:
        constitution_block = (
            f"=== CHARACTER CONSTITUTION ===\n"
            f"{scenario.npc_constitution}\n\n"
        )

    # Dynamic disposition (changes based on conversation progress)
    disposition_block = build_disposition_block(npc_state)

    # Repetition guard — prevent the NPC from restating old points
    repetition_block = build_repetition_guard(conversation_history)

    # Dead-turn pushback — if the player gave a thin response, demand substance
    pushback_rule = build_pushback_rule(player_message)

    return f"""{constitution_block}{simtom_block}

{disposition_block}{repetition_block}{pushback_rule}

You are {scenario.npc_name}, {scenario.npc_role}.
Respond only as {scenario.npc_name}. 2-4 sentences. First person. Stay in character.
Do not reference game mechanics, scores, or the simulation."""
```

### Critical: Knowledge Boundaries Run Every Turn

The constitution can be periodic, but knowledge boundaries **must** be in every single system prompt. The model will leak within 3-5 turns without them.

## Step 4: Dynamic Disposition Tracking

Track how the character's attitude changes based on conversation quality:

```python
def build_disposition_block(npc_state: dict) -> str:
    """Build dynamic disposition based on current trust/persuasion state."""
    progress = npc_state.get("persuasion_progress", 0)

    # Map progress to a behavioral tier
    if progress < 20:
        tier = "deeply skeptical -- politely dismissive, not hostile"
    elif progress < 40:
        tier = "mildly curious -- willing to engage but not persuaded"
    elif progress < 60:
        tier = "genuinely uncertain -- listening hard, certainty is cracking"
    elif progress < 70:
        tier = "leaning -- haven't committed but close to reconsidering"
    else:
        tier = "nearly convinced -- looking for a reason to change course"

    trust_comp = npc_state.get("trust_competence", 50)
    trust_hon = npc_state.get("trust_honesty", 50)
    affinity = npc_state.get("personal_affinity", 50)

    return f"""=== CURRENT REGISTER ===
Your current stance: {tier}
Trust in their competence: {trust_comp:.0f}/100
Trust in their honesty: {trust_hon:.0f}/100
Personal respect: {affinity:.0f}/100

Let these values subtly color your tone and receptivity -- do not state them explicitly."""
```

## Step 5: Repetition Guard

Prevent the character from making the same point twice:

```python
def build_repetition_guard(conversation_history: list[dict]) -> str:
    """Pull last 2 NPC turns and instruct the model not to repeat them."""
    last_npc_turns = [
        msg["content"] for msg in conversation_history
        if msg["role"] == "assistant"
    ][-2:]

    if not last_npc_turns:
        return ""

    prior = "\n".join(f'  "{t[:120]}..."' for t in last_npc_turns)
    return f"""
=== AVOID REPEATING ===
You have already made these points -- do not restate them, find a new angle
or ask a pointed question:
{prior}"""
```

## Step 6: Dead-Turn Pushback

When the player gives a thin, low-effort response, the NPC should demand substance rather than filling the gap themselves:

```python
def build_pushback_rule(player_message: str) -> str:
    """If the player gave almost nothing, force the NPC to push back."""
    is_thin = len(player_message.split()) <= 8

    if not is_thin:
        return ""

    return """
=== PUSHBACK REQUIRED ===
Their last response gave you almost nothing new. Do not elaborate further on your
own position. Push back: demand a specific fact, a name, a number, or a mechanism.
Don't move an inch without it."""
```

## Step 7: Track Revealed Facts

As the player reveals information that overlaps with "does not know" items, move those facts to the "revealed" list so the character can respond to them naturally:

```python
def update_revealed_facts(scenario: ScenarioSpec,
                          npc_state: dict,
                          player_message: str) -> None:
    """Check if the player has revealed any previously unknown facts."""
    revealed = npc_state.setdefault("revealed_facts", [])

    for fact in scenario.simtom_does_not_know:
        if fact in revealed:
            continue  # already revealed
        # Check if the player explicitly stated this fact
        if fact_was_explicitly_stated(player_message, fact):
            revealed.append(fact)
```

## Step 8: Convert History to Model Format

Keep conversation history in a neutral format and convert to provider-specific format at call time:

```python
def to_gemini_history(conversation_history: list[dict]) -> list[dict]:
    """Convert {role: user/assistant} to Gemini {role: user/model} format."""
    result = []
    for msg in conversation_history:
        role = "model" if msg["role"] == "assistant" else "user"
        result.append({"role": role, "parts": [{"text": msg["content"]}]})
    return result

def to_openai_history(conversation_history: list[dict]) -> list[dict]:
    """Already in OpenAI format."""
    return conversation_history
```

## Complete Turn Flow

```python
async def npc_turn(scenario: ScenarioSpec,
                   game_state: dict,
                   player_message: str) -> str:
    """Generate one NPC response with full knowledge isolation."""

    # 1. Check if player revealed any new facts
    update_revealed_facts(scenario, game_state["npc_state"], player_message)

    # 2. Build system prompt with knowledge boundaries
    system_prompt = build_system_prompt(
        scenario,
        game_state["npc_state"],
        game_state["turn_count"],
        game_state["conversation_history"],
        player_message,
    )

    # 3. Build conversation for model
    history = to_model_history(game_state["conversation_history"])
    history.append({"role": "user", "content": player_message})

    # 4. Generate response
    response = await generate_response(
        system_prompt=system_prompt,
        messages=history,
        temperature=0.4,      # low temperature for consistency
        max_tokens=300,        # short responses = more in-character
    )

    # 5. Update game state
    game_state["conversation_history"].append({"role": "user", "content": player_message})
    game_state["conversation_history"].append({"role": "assistant", "content": response})
    game_state["turn_count"] += 1

    return response
```

## Design Principles

1. **Enumerate forbidden knowledge explicitly.** "You don't know about X" is weak. "You do NOT know: [specific fact]. Never volunteer, infer, or hint at it." is strong. The model needs concrete items to avoid, not vague categories.

2. **Re-inject boundaries every turn.** The knowledge boundary block is not optional on any turn. The model's context window is finite -- without constant reinforcement, it will leak within 3-5 turns. The constitution can be periodic (every N turns), but boundaries are every turn.

3. **Give the model a behavior for the gap.** "Don't mention X" creates a void the model fills unpredictably. "Wait. Make them say it." gives it a concrete action. Always pair prohibitions with alternative behaviors.

4. **Track revealed facts.** As the player reveals information, the character should naturally respond to it. The "revealed facts" list lets the model engage with formerly forbidden knowledge once it's been explicitly shared.

5. **Low temperature, short responses.** High temperature increases the chance of creative leakage. Long responses give more surface area for errors. Keep both constrained for characters with strict boundaries.

6. **Pushback on thin inputs.** If the model has nothing substantial to respond to, it will fill the gap with its own knowledge -- which means leaking. Forcing pushback keeps the character from volunteering forbidden information when there's nothing else to say.

7. **Separate constitution from knowledge.** The constitution (personality, speech style) drifts slowly and can be refreshed every N turns. Knowledge boundaries are binary (know/don't know) and must be enforced every turn. Different refresh cadences for different concerns.
