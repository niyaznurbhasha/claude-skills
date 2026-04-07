---
name: chaos-controlled-behavior-system
description: Maps a single global parameter (0-1) to many derived behavior metrics for cohesive interactive response. Entities respond to one control value instead of tuning each subsystem separately. Use when building games, visualizations, interactive UIs, simulations, or any system where multiple behaviors need to scale together from a single intensity dial.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: miscellaneous
  tags: [chaos, behavior-system, game-design, interactive, animation, parameter-mapping, ui]
---

# Chaos-Controlled Behavior System

A pattern where a single global parameter (0 to 1) drives all dynamic behavior in a system. Instead of tuning 10+ knobs independently, every subsystem reads from one value and derives its own range. The result: cohesive, predictable intensity scaling from a single control point.

## Core Concept

Define one global value (call it `chaos`, `intensity`, `energy`, or whatever fits your domain). Every behavior in the system derives its parameters from this value using linear interpolation between a minimum and maximum.

```
chaos = 0.0 → everything calm, slow, minimal
chaos = 0.5 → normal baseline behavior
chaos = 1.0 → maximum intensity, fast, dramatic
```

The key insight: you never tune individual behaviors directly. You define each behavior's min/max range once, then the single global value controls everything in lockstep.

## The Global Parameter

```javascript
// JavaScript example
let _chaos = 0.5;

export const chaos = {
  get value() { return _chaos; },
  set value(v) { _chaos = Math.max(0, Math.min(1, v)); },

  // Derived values — each maps chaos 0-1 to its own range
  get speedMultiplier()     { return 0.2 + _chaos * 2.8; },        // 0.2x – 3.0x
  get idleTimerMin()        { return 8 - _chaos * 7; },            // 8s – 1s
  get idleTimerMax()        { return 12 - _chaos * 10; },          // 12s – 2s
  get actionChance()        { return 0.1 + _chaos * 0.7; },        // 10% – 80%
  get wanderSpeed()         { return 0.3 + _chaos * 3.5; },        // 0.3 – 3.8
  get bobIntensity()        { return 0.3 + _chaos * 2.0; },        // 0.3x – 2.3x
  get interactionInterval() { return 6 - _chaos * 5; },            // 6s – 1s
  get interactionChance()   { return 0.1 + _chaos * 0.5; },        // 10% – 60%
  get reactRadius()         { return 2 + _chaos * 8; },            // 2 – 10 units
  get reactChance()         { return 0.2 + _chaos * 0.7; },        // 20% – 90%
  get squashIntensity()     { return 0.1 + _chaos * 0.6; },        // 0.1 – 0.7
  get behaviorIntensity()   { return 0.3 + _chaos * 1.7; },        // 0.3x – 2.0x
};
```

```python
# Python equivalent
class ChaosParameter:
    def __init__(self, value=0.5):
        self._value = max(0.0, min(1.0, value))

    @property
    def value(self):
        return self._value

    @value.setter
    def value(self, v):
        self._value = max(0.0, min(1.0, v))

    @property
    def speed_multiplier(self):
        return 0.2 + self._value * 2.8

    @property
    def action_chance(self):
        return 0.1 + self._value * 0.7

    # ... add more derived values as needed

chaos = ChaosParameter()
```

## The Mapping Formula

Every derived value follows this pattern:

```
derived = min_value + chaos * (max_value - min_value)
```

Where:
- `min_value` is the output when chaos = 0
- `max_value` is the output when chaos = 1
- The relationship is linear by default

For shorthand in code: `min + chaos * range` where `range = max - min`.

### Choosing Ranges

| Metric Type | Low Chaos (0) | High Chaos (1) | Notes |
|-------------|---------------|-----------------|-------|
| Speed multiplier | 0.2x | 3.0x | Never zero — things should still move |
| Event chance | 5-10% | 70-90% | Cap below 100% to keep randomness |
| Timer interval | Long (6-12s) | Short (1-2s) | Inverts: high chaos = short intervals |
| Amplitude/intensity | 0.1-0.3 | 1.5-2.5 | Scale of motion/effect |
| Radius/range | Small (2) | Large (10) | Area of influence |

Note that timer intervals invert: `interval = max_interval - chaos * range`. Higher chaos means shorter wait times.

## Behavior Functions

Each entity type has its own behavior function that reads from the global chaos parameter. The behavior function uses two common accessors:

```javascript
const I = () => chaos.behaviorIntensity;  // amplitude/scale
const S = () => chaos.speedMultiplier;     // time/speed

function workout(entity, dt) {
  entity.phase += dt * 8 * S();
  const pump = Math.sin(entity.phase);
  entity.y = Math.abs(pump) * 0.2 * I();
  const squash = 1 + pump * 0.12 * I();
  entity.scaleY = squash;
  entity.scaleX = 1 / squash;
}

function meditate(entity, dt) {
  // Some behaviors can RESIST chaos — stay calm even at high values
  const calm = Math.max(0.3, 1 - chaos.value * 0.5);
  entity.phase += dt * 0.8 * (0.5 + S() * 0.5);
  entity.y = Math.sin(entity.phase) * 0.05 * calm;
}
```

Key patterns:
- **Speed scaling**: Multiply `dt` (delta time) by `S()` to speed up/slow down animations
- **Intensity scaling**: Multiply amplitudes by `I()` to scale motion magnitude
- **Chaos-resistant behaviors**: Some behaviors should stay calm regardless. Use `max(floor, 1 - chaos * factor)` to dampen the chaos influence.

## Behavior Dispatch

Use a switch/dispatch pattern to route entities to their behavior function:

```javascript
function applyBehavior(entity, dt) {
  switch (entity.behaviorType) {
    case 'workout':
    case 'pushup':
      workout(entity, dt);
      break;
    case 'dance':
    case 'groove':
      dance(entity, dt);
      break;
    case 'meditate':
    case 'breathe':
      meditate(entity, dt);
      break;
    default:
      genericIdle(entity, dt);
  }
}
```

Grouping multiple type strings to the same function (e.g., `workout` and `pushup` both route to `workout()`) lets you have rich type vocabularies without duplicating logic.

## Connecting the Slider

The chaos value can be driven by:

### User Input (slider, scroll, mouse position)
```javascript
slider.addEventListener('input', (e) => {
  chaos.value = parseFloat(e.target.value);
});
```

### Game State (health, score, difficulty)
```javascript
function updateChaos(gameState) {
  // Low health = high chaos
  chaos.value = 1 - (gameState.health / gameState.maxHealth);
}
```

### Time or Events
```javascript
// Gradually increase over time
chaos.value = Math.min(1, chaos.value + 0.001 * dt);

// Spike on events, decay back
function onExplosion() {
  chaos.value = Math.min(1, chaos.value + 0.3);
}
function decay(dt) {
  chaos.value = Math.max(0.5, chaos.value - 0.05 * dt);
}
```

### Audio Amplitude
```javascript
// Drive chaos from music volume
function onAudioFrame(amplitude) {
  chaos.value = amplitude;  // 0-1 normalized
}
```

## Design Principles

1. **Never zero out a behavior.** Minimum values should be small but positive (0.1, 0.2). An entity that stops completely looks broken, not calm.
2. **One value to rule them all.** If you find yourself wanting a second global parameter, resist. Instead, make certain behaviors resist or amplify the single parameter.
3. **Ranges are the tuning interface.** Once the system is built, all tuning happens by adjusting the min/max of each derived value. This is far easier than tuning independent parameters.
4. **Chaos-resistant behaviors add character.** A meditating entity that barely reacts to chaos = 1.0 is more interesting than one that goes wild like everything else.
5. **Use getters/properties.** Every derived value should be computed on access, not cached. The chaos value can change every frame.

## Adapting to Different Systems

- **Particle systems**: chaos controls emission rate, speed, size, lifetime, spread angle
- **Music/audio**: chaos controls tempo, reverb, distortion, filter cutoff
- **UI animations**: chaos controls transition speed, bounce, overshoot, parallax depth
- **NPC behavior**: chaos controls aggression, patrol speed, detection radius, response time
- **Data visualization**: chaos controls jitter, transition duration, label density, color saturation
- **Procedural generation**: chaos controls noise frequency, feature density, irregularity

The pattern is domain-agnostic. Any system with multiple tunable parameters can benefit from collapsing them into derived values from a single control point.
