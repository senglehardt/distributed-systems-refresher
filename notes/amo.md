Note: This was written by AI.

In distributed systems, there are three delivery semantics:

| Semantic | Guarantee | Issue |
|----------|-----------|-------|
| **At-least-once** | Command executes ≥1 times | May execute multiple times; breaks non-idempotent ops |
| **At-most-once** | Command executes ≤1 times | May not execute at all; client left hanging |
| **Exactly-once** | Command executes exactly 1 time | The ideal, but requires both above |

**In the Paxos lab, they combine:**

1. **Paxos provides at-least-once**: The consensus protocol guarantees that once a command is chosen (majority accepts it), it will eventually be executed on all replicas. Retries and leader elections ensure liveness.

2. **AMO adds deduplication for at-most-once**: `GenericAMOApplication` tracks `(client, seq_num)` pairs and returns cached results on retransmission, preventing duplicate execution.

3. **Together = exactly-once**: 
   - Client retries if no response (at-least-once via Paxos)
   - Server deduplicates via seq_num (at-most-once via AMO)
   - Result: command executes exactly once, even if network drops the response

**Example**: If a client sends `PUT key=x value=y` and the reply is lost:
- Without AMO: Client retries → key gets set twice (broken)
- With AMO: Client retries → server sees seq_num already executed → returns cached result (correct)

This is why AMO is essential in Paxos: Paxos alone only guarantees "will eventually be applied," not "will only be applied once." The application layer must enforce exactly-once.