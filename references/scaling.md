# LangGraph Reference: Scaling Metrics & Execution Model

## The Pregel-Based Execution Model

LangGraph's reliability is rooted in the **Bulk Synchronous Parallel (BSP)** algorithm, specifically the Pregel model. This ensures deterministic concurrency and state consistency, even in cyclical graphs.

## Execution Models Comparison

| Feature | Traditional Sequential | LangGraph Deterministic Parallelism |
|---------|----------------------|-------------------------------------|
| Concurrency Management | Manual threading/mutex; high race risk | Independent state copies; updates at step boundaries |
| State Consistency | Shared memory corruption risks | Versioned channels ensure atomic updates |
| Fault Tolerance | Full process restarts expensive | Checkpointing allows O(1) resumption |
| Logic Flow | Primarily DAGs only | Native cycles and sophisticated looping |

## Scaling Metrics (O(n) Characteristics)

| Metric / Action | Starting Invocation | Planning a Step | Running a Step | Finishing a Step |
|-----------------|--------------------|-----------------|----------------|--------------------|
| Number of Nodes | O(n) | O(1) | O(1) | O(n) |
| Number of Channels | O(n) | O(n) | O(n) | O(n) |
| Active Nodes | O(1) | O(n) | O(n) | O(n) |
| Length of History | O(1) | O(1) | O(1) | O(1) |

**Note**: Length of History is O(1) because checkpointing fetches only the latest state - no need to replay previous steps.

## Collaboration Patterns

### Shared Scratchpad (Collaboration)
- All agents share message history
- Use for small 2-3 agent teams
- Full context required for coordination

### Isolated Context (Supervisor)
- Agents maintain independent scratchpads
- Only final results shared
- Critical for context window management
- Prevents "context bloating"

## Worker Hierarchy Pattern

Following the Build.inc model, a four-tier system:

```
Master Agent → Role Agent → Sequence Agent → Task Agent
```

Each agent as a modular subgraph allows:
- Independent testing
- Model swapping (GPT-4 for Master, SLMs for Tasks)
- No disruption to global orchestration