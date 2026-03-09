---
name: langgraph-agentic-pipeline
description: Build multi-node LangGraph StateGraph pipelines with interactive, background, and full execution modes, plus timing instrumentation. Use when building a multi-step agent pipeline, when someone says "build an agent with LangGraph", "add a node to the pipeline", "I need different execution modes", or needs a structured agent graph.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: agent-architecture
  tags: [langgraph, pipeline, stategraph, agent, execution-modes, instrumentation]
---

# LangGraph Agentic Pipeline

Build structured multi-node agent pipelines using LangGraph's StateGraph with three execution modes (interactive, background, full), timing instrumentation, and clean node composition.

## Architecture Overview

```
Full Mode (default):
  __start__ -> intent -> policy -> planner -> tools -> reply -> insights -> feedback -> memwrite -> __end__
                                                        |
Interactive Mode (low-latency):                         |
  __start__ -> intent -> policy -> planner -> tools -> reply -> __end__

Background Mode (post-reply processing):
  __start__ -> insights -> feedback -> memwrite -> __end__
```

The key insight: split your pipeline into a **core path** (user-facing, latency-sensitive) and a **background path** (post-reply processing that can run async). Offer three modes so callers can choose the right tradeoff.

## Step 1: Define the State Shape

The state dict flows through every node. Define it once with all fields any node might need:

```python
from typing import Any, Dict

def new_state(thread_id: str, user_input: str) -> Dict[str, Any]:
    return {
        "thread_id": thread_id,
        "messages": [{"role": "user", "content": user_input}],

        # Planning
        "goal": "",
        "subgoals": [],
        "plan": [],
        "next_action": None,

        # Execution
        "scratchpad": {},
        "tool_io_log": [],        # [{"tool": str, "input": Any, "output": Any}]
        "retrieved_refs": [],

        # Quality
        "confidence": 0.0,
        "pending_checks": [],
        "tags": [],
        "meta_memory": [],

        # Metrics
        "metrics": {
            "tokens_used": 0,
            "tool_calls": 0,
            "turn_index": 0,
            "needs_planning": False,
            "insights_trigger": None,
            "last_insights_turn": -1,
        },
    }
```

### State Design Rules

- **Messages list is append-only** — nodes add to it, never replace. Newest last.
- **Scratchpad is freeform** — intermediate node outputs that don't fit structured fields go here.
- **Tool IO log tracks every tool call** — essential for debugging and evaluation.
- **Metrics accumulate** — every node can increment counters.

## Step 2: Build Node Functions

Each node is a function that takes state and returns (possibly modified) state. Wrap nodes with a factory pattern to inject dependencies (LLM, domain config, user_id):

```python
from typing import Callable

NodeFn = Callable[[dict], dict]

def intent_node_fn(llm, domain: str) -> NodeFn:
    """Create an intent classification node bound to a specific LLM and domain."""
    def run(state: dict) -> dict:
        user_msg = state["messages"][-1]["content"]
        # Classify intent using the LLM
        intent = classify_intent(llm, user_msg, domain)
        state["scratchpad"]["intent"] = intent
        return state
    return run

def policy_node_fn(domain: str) -> NodeFn:
    """Create a policy guard node that checks safety/policy constraints."""
    def run(state: dict) -> dict:
        violation = check_policy(state, domain)
        if violation:
            state["messages"].append({
                "role": "assistant",
                "content": violation["response"]
            })
            state["scratchpad"]["policy_blocked"] = True
        return state
    return run

def planner_node_fn(llm, domain: str) -> NodeFn:
    """Create a planner that decomposes the goal into steps."""
    def run(state: dict) -> dict:
        if not state["metrics"]["needs_planning"]:
            return state
        plan = generate_plan(llm, state)
        state["plan"] = plan
        return state
    return run

def tools_node_fn(llm, domain: str, user_id: str) -> NodeFn:
    """Create a tool router that executes planned tool calls."""
    def run(state: dict) -> dict:
        results = execute_tools(llm, state, domain, user_id)
        state["tool_io_log"].extend(results)
        return state
    return run

def reply_node_fn(llm, domain: str, user_id: str) -> NodeFn:
    """Create the reply generation node."""
    def run(state: dict) -> dict:
        reply = generate_reply(llm, state, domain, user_id)
        state["messages"].append({"role": "assistant", "content": reply})
        return state
    return run
```

## Step 3: Add Timing Instrumentation

Wrap every node with timing so you can identify bottlenecks:

```python
import time
import logging

logger = logging.getLogger(__name__)

def timed_node(name: str, fn: NodeFn) -> NodeFn:
    """Wrap a node function with timing instrumentation."""
    def wrapper(state: dict) -> dict:
        start = time.perf_counter()
        result = fn(state)
        elapsed = time.perf_counter() - start
        logger.info(f"Node '{name}' completed in {elapsed:.3f}s")
        # Optionally store timing in state metrics
        state.setdefault("_node_timings", {})[name] = elapsed
        return result
    return wrapper
```

## Step 4: Build the Graph with Execution Modes

The core pattern: one `build_graph` function that accepts a `mode` parameter and returns different compiled graphs.

```python
from langgraph.graph import StateGraph

def build_graph(domain: str, user_id: str, mode: str = "full"):
    """Build the agent pipeline graph.

    Modes:
        interactive: core path only (intent -> reply). Low latency.
        background:  post-reply nodes only (insights -> memwrite). Run async after reply.
        full:        everything end-to-end. Default.
    """
    llm = get_chat_llm()
    fast_llm = get_fast_llm()  # cheaper/faster model for classification

    # Background-only: post-reply processing
    if mode == "background":
        bg = StateGraph(dict)
        bg.add_node("insights",  timed_node("insights",  insights_node_fn(fast_llm, domain, user_id)))
        bg.add_node("feedback",  timed_node("feedback",  feedback_node_fn(fast_llm, domain)))
        bg.add_node("memwrite",  timed_node("memwrite",  memwrite_node_fn(user_id)))

        bg.add_edge("__start__", "insights")
        bg.add_edge("insights",  "feedback")
        bg.add_edge("feedback",  "memwrite")
        bg.add_edge("memwrite",  "__end__")
        return bg.compile()

    # Core interactive path
    g = StateGraph(dict)
    g.add_node("intent",  timed_node("intent",  intent_node_fn(fast_llm, domain)))
    g.add_node("policy",  timed_node("policy",  policy_node_fn(domain)))
    g.add_node("planner", timed_node("planner", planner_node_fn(fast_llm, domain)))
    g.add_node("tools",   timed_node("tools",   tools_node_fn(fast_llm, domain, user_id)))
    g.add_node("reply",   timed_node("reply",   reply_node_fn(llm, domain, user_id)))

    g.add_edge("__start__", "intent")
    g.add_edge("intent",    "policy")
    g.add_edge("policy",    "planner")
    g.add_edge("planner",   "tools")
    g.add_edge("tools",     "reply")

    if mode == "interactive":
        g.add_edge("reply", "__end__")
        return g.compile()

    # Full mode: add post-reply nodes
    g.add_node("insights",  timed_node("insights",  insights_node_fn(fast_llm, domain, user_id)))
    g.add_node("feedback",  timed_node("feedback",  feedback_node_fn(fast_llm, domain)))
    g.add_node("memwrite",  timed_node("memwrite",  memwrite_node_fn(user_id)))

    g.add_edge("reply",     "insights")
    g.add_edge("insights",  "feedback")
    g.add_edge("feedback",  "memwrite")
    g.add_edge("memwrite",  "__end__")

    return g.compile()
```

## Step 5: Run the Pipeline

```python
# Low-latency path: return reply immediately, process background async
async def handle_message(thread_id: str, user_id: str, user_input: str, domain: str):
    state = new_state(thread_id, user_input)

    # Phase 1: interactive (user sees reply fast)
    interactive_graph = build_graph(domain, user_id, mode="interactive")
    state = interactive_graph.invoke(state)
    reply = state["messages"][-1]["content"]

    # Phase 2: background (runs after response is sent)
    background_graph = build_graph(domain, user_id, mode="background")
    asyncio.create_task(run_background(background_graph, state))

    return reply

async def run_background(graph, state):
    """Fire-and-forget background processing."""
    try:
        graph.invoke(state)
    except Exception as e:
        logger.error(f"Background processing failed: {e}")
```

## Step 6: Resilient Node Wrapping

Handle nodes that may have different call signatures or fail gracefully:

```python
def wrap_node(factory_attempts: list, direct_attempts: list) -> NodeFn:
    """Try multiple ways to create/call a node function.

    Handles API changes in node functions without breaking the pipeline.
    """
    for attempt in factory_attempts:
        try:
            fn = attempt()
            if callable(fn):
                return fn
        except TypeError:
            pass

    def run(state: dict) -> dict:
        for attempt in direct_attempts:
            try:
                out = attempt(state)
                return state if out is None else out
            except TypeError:
                continue
        return state

    return run
```

## Step 7: Domain Configuration

Make the pipeline configurable per domain so the same graph structure serves different use cases:

```python
from dataclasses import dataclass, field

@dataclass
class DomainConfig:
    name: str
    system_prompt: str
    available_tools: list[str]
    ephemeral_fact_keys: list[str] = field(default_factory=list)
    requires_planning: bool = True
    max_tool_calls: int = 5

DOMAINS = {
    "fitness": DomainConfig(
        name="fitness",
        system_prompt="You are a fitness coach...",
        available_tools=["log_workout", "log_nutrition", "search_exercises"],
        ephemeral_fact_keys=["current_workout", "today_calories"],
    ),
    "finance": DomainConfig(
        name="finance",
        system_prompt="You are a financial advisor...",
        available_tools=["get_portfolio", "search_stocks"],
    ),
}

def get_domain_config(domain: str) -> DomainConfig:
    if domain not in DOMAINS:
        raise ValueError(f"Unknown domain: {domain}")
    return DOMAINS[domain]
```

## Design Principles

1. **Use cheap models for classification, expensive models for generation.** Intent classification and planning use `fast_llm`; reply generation uses the full `chat_llm`. This cuts cost and latency.

2. **Separate interactive from background.** The user should see a reply in under 2 seconds. Insights extraction, memory writes, and feedback processing can happen after the response is sent.

3. **Every node gets timing.** Wrap all nodes with `timed_node`. When latency degrades, the logs tell you exactly which node is slow.

4. **State flows forward, never backward.** Nodes read from state and append to it. No node deletes or overwrites another node's output. This makes debugging trivial — inspect the final state to see every node's contribution.

5. **Nodes are composable.** Each node function is created by a factory that takes its dependencies. The graph builder composes them. You can add, remove, or reorder nodes without touching node internals.

6. **Mode parameter, not separate graphs.** One `build_graph` function with a mode parameter keeps the pipeline definition in one place. Adding a new mode is a few lines, not a new file.
