---
name: langgraph-review
description: Production-grade LangGraph code review. Use when reviewing LangGraph implementations, agent architectures, state management, graph topology, multi-agent systems, or production reliability patterns. Triggers on mentions of LangGraph, agents, state machines, graphs, checkpoints, or LangSmith.
disable-model-invocation: true
---

# LangGraph Technical Capability Manual: Production-Grade Code Review

## Purpose

Review LangGraph code for production readiness, focusing on architectural integrity, reliability, and performance. This skill ensures implementations follow the "Control over Agency" philosophy essential for enterprise systems.

## Core Philosophy: Control Over Agency

The primary mandate in production systems is **developer-controlled orchestration** over high-level agent abstractions. High-level wrappers introduce "Black Box" failure modes. For enterprise stability:
- Prioritize **deterministic state machines** over autonomous agent behaviors
- Ensure **granular control** of prompts, LLM invocations, and tool execution
- Implement **hard safety guardrails** that high-level abstractions bypass

## Code Review Checklist

### Category 1: State Architecture

#### 1.1 State Minimalism
**Check**: Are state channels restricted to essential data?
- Look for "catch-all" JSON strings or bloated state objects
- Verify each channel has a clear purpose
- **Success Criterion**: No unused keys, minimal payload

```python
# BAD: Over-serialized, loss of type safety
class GlobalState(TypedDict):
    data_blob: str  # Single JSON string containing everything

# GOOD: Granular control, typed channels
class AgentState(TypedDict):
    conversation_history: Annotated[list[AnyMessage], add_messages]
    current_intent: str
    security_clearance: bool
    internal_scratchpad: list[str]
```

#### 1.2 Strict Typing
**Check**: Is state defined using TypedDict with explicit types?
- All fields must have explicit type annotations
- Use Annotated types for reducers
- **Success Criterion**: 100% type-safety for all state transitions

#### 1.3 Reducer Logic
**Check**: Are message reducers (`add_messages`) used correctly?
- History management prevents context bloat
- Custom reducers should be documented
- **Success Criterion**: History stays within model context window limits

### Category 2: Flow Control

#### 2.1 Recursion Limits
**Check**: Is `recursion_limit` explicitly set for every graph invocation?
- Hard limit prevents infinite loops
- Default is typically 25-50 steps
- **Success Criterion**: Zero risk of runaway loops consuming token budget

```python
# GOOD: Explicit recursion limit
graph.invoke(inputs, config={"recursion_limit": 50})
```

#### 2.2 Deterministic Safety Edges
**Check**: Are critical guardrails handled by rule-based routers?
- Security checks, permissions, validation should NOT rely on LLM routing
- Use deterministic edges for safety-critical transitions
- **Success Criterion**: High-risk transitions impossible without valid state flags

```python
def route_after_scan(state: AgentState):
    # Deterministic check - NOT LLM-based
    if not state.get("security_scan_passed"):
        return "unauthorized_access_node"
    # Semantic routing for non-critical paths
    return "execute_tool_node"
```

#### 2.3 Parallel Efficiency
**Check**: Are independent nodes explicitly parallelized?
- Multi-tool searches, independent computations should run in parallel
- LangGraph handles data races automatically via Pregel model
- **Success Criterion**: Total latency significantly lower than sequential execution

### Category 3: Reliability & Human-in-the-Loop

#### 3.1 Persistence Layer
**Check**: Is a Postgres/Redis checkpointer enabled for multi-turn threads?
- Production systems require durable state persistence
- Enables "Time Travel" - inspect, rewind, rewire trajectories
- **Success Criterion**: System can resume from any step after failure

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver(connection_string)
graph.invoke(inputs, config={"configurable": {"thread_id": "user-123"}})
```

#### 3.2 Strategic Interruption
**Check**: Are `interrupt()` calls placed before sensitive tool executions?
- Financial transactions, data deletions, external API writes
- Human approval required before proceeding
- **Success Criterion**: No sensitive operations without human verification

#### 3.3 Concurrency Strategy
**Check**: Is a double-texting strategy defined?
- How does the system handle new inputs on active threads?
- Options: Reject, Queue, Interrupt, Rollback
- **Success Criterion**: New inputs don't cause state corruption

### Category 4: Performance & Scaling

#### 4.1 Serialization Check
**Check**: Is state size being monitored?
- Bloated state increases serialization latency
- Linear degradation in Finish/Start times
- **Success Criterion**: State payload under 100KB for typical web agents

#### 4.2 Streaming Implementation
**Check**: Are token-level and update-level streams implemented?
- `values`: Full state stream
- `updates`: Changed channels only
- `messages`: Real-time tokens
- **Success Criterion**: Time to First Token under 2 seconds

```python
async for event in graph.astream_events(inputs, version="v2"):
    if event["event"] == "on_chat_model_stream":
        yield event["data"]["chunk"]
```

### Category 5: Multi-Agent Design

#### 5.1 Modular Subgraphs
**Check**: Are complex agent behaviors encapsulated in independent subgraphs?
- Each agent should be testable in isolation
- Model swapping should be possible without graph changes
- **Success Criterion**: Individual agents swappable without modifying Master graph

#### 5.2 Supervisor Efficiency
**Check**: Does the supervisor pattern minimize unnecessary LLM chatter?
- Over-orchestration creates latency without quality gains
- Supervisor should route to specialist in < 1 step overhead
- **Success Criterion**: Clear delegation, minimal communication overhead

## Common Anti-Patterns to Flag

### 1. The "Loop Agent" Anti-Pattern
**Warning**: Pure loop agents where LLM directs its own process
- Risk: Infinite reasoning loops
- Risk: Hallucinated tool calls
- **Fix**: Use StateGraph with explicit termination conditions

### 2. Over-Orchestration
**Warning**: Supervisor approves every trivial sequential step
- Creates massive communication overhead
- No quality improvement for simple flows
- **Fix**: Supervisor only for cross-agent coordination

### 3. Missing Termination Criteria
**Warning**: Vague or implicit graph termination
- "The LLM will know when to stop" is not acceptable
- **Fix**: Explicit "FINAL ANSWER" node or step counter

### 4. Context Bloated States
**Warning**: Agents sharing full message history unnecessarily
- Hits context window limits
- Degrades performance
- **Fix**: Isolated context with result aggregation (Supervisor pattern)

## Review Process

1. **Identify the architecture**: Single agent vs. multi-agent vs. hierarchical
2. **Map the state flow**: Check each channel, reducer, and update pattern
3. **Verify safety edges**: All security/permission checks are deterministic
4. **Check persistence**: Checkpointer configured for production use
5. **Validate termination**: Every cycle has explicit exit conditions
6. **Assess performance**: Parallelization opportunities, streaming implementation
7. **Review HITL**: Human approval gates for sensitive operations

## Output Format

Provide a structured review:

```
## Architecture Assessment
[Overall pattern and approach]

## Critical Issues
[Must-fix before production]

## Warnings
[Should address for production quality]

## Recommendations
[Nice-to-have improvements]

## Checklist Summary
- [ ] State Minimalism: [PASS/FAIL]
- [ ] Strict Typing: [PASS/FAIL]
- [ ] Reducer Logic: [PASS/FAIL]
- [ ] Recursion Limits: [PASS/FAIL]
- [ ] Deterministic Safety Edges: [PASS/FAIL]
- [ ] Parallel Efficiency: [PASS/FAIL/N/A]
- [ ] Persistence Layer: [PASS/FAIL/N/A]
- [ ] Strategic Interruption: [PASS/FAIL/N/A]
- [ ] Concurrency Strategy: [PASS/FAIL/N/A]
- [ ] Serialization Check: [PASS/FAIL]
- [ ] Streaming Implementation: [PASS/FAIL/N/A]
- [ ] Modular Subgraphs: [PASS/FAIL/N/A]
- [ ] Supervisor Efficiency: [PASS/FAIL/N/A]
```

Be concise, actionable, and focused on production readiness. Prioritize reliability and safety over flexibility.