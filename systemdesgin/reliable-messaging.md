# Reliable Messaging Pattern (RPOPLPUSH Pattern)

## Overview

The Reliable Messaging pattern ensures that no messages are lost even if a consumer crashes during processing. It transforms a simple "fire-and-forget" queue into a guaranteed delivery system using an atomic transfer mechanism.

## Problem

In a standard queue implementation, when a worker pops a message from the queue, the message is immediately removed. If the worker crashes before completing the processing, the message is permanently lost.

## Solution

Use an **In-flight List** (also called Working List or Processing List) as a safety net. Instead of removing messages directly, atomically transfer them to a temporary processing list until work is confirmed complete.

## How It Works

1. **Pickup**: Atomically move a message from the Pending queue to the In-flight list
2. **Process**: The worker executes the job logic
3. **Acknowledge**: Once finished successfully, the worker removes the message from the In-flight list
4. **Recovery**: A separate "Janitor" process scans the In-flight list for stale messages and moves them back to Pending

### Visual Flow

```
Pending Queue          In-flight List         Completed
┌─────────┐           ┌─────────┐
│ Job 3   │           │ Job 1   │  ──────>   ✓ Processed
│ Job 2   │  ───────> │ Job 2   │
│ Job 1   │  Atomic   │         │
└─────────┘   Move    └─────────┘
                           │
                           │ (If timeout)
                           │
                           ▼
                      Back to Pending
```

## Key Components

### The Janitor (Garbage Collection)

A background process that handles worker failures by:
1. Scanning the In-flight list periodically
2. Identifying messages older than a defined timeout (e.g., 5 minutes)
3. Moving them back to the Pending queue for retry

This prevents "Zombie Jobs" - messages stuck in the In-flight list from crashed workers.

## Benefits

- **At-least-once Delivery**: Guarantees every job is processed at least once
- **Visibility**: Monitor the In-flight list to see jobs currently being processed
- **Fault Tolerance**: Handles consumer crashes gracefully without data loss
- **Simplicity**: Uses basic primitives without complex distributed coordination

## Trade-offs

**Advantages**
- ✅ Guaranteed message processing
- ✅ Real-time visibility into processing state
- ✅ Automatic recovery from worker crashes

**Disadvantages**
- ❌ Messages may be processed more than once (requires idempotent operations)
- ❌ Requires additional Janitor process
- ❌ Slightly higher latency than simple pop operations
- ❌ Additional memory overhead for In-flight list

## Real-world Equivalents

This pattern is conceptually similar to **AWS SQS Visibility Timeout**:
- SQS "hides" messages during processing
- This pattern moves messages to an In-flight list
- Both ensure messages aren't lost if workers crash
- Both require timeout-based recovery mechanisms

## Use Cases

- **Job Processing Systems**: Background job queues that require guaranteed execution
- **Event Processing**: Event-driven architectures where event loss is unacceptable
- **Task Distribution**: Distributing work across multiple workers with failure recovery
- **Message Brokers**: Building lightweight message queue systems

## Best Practices

1. **Idempotent Operations**: Design job handlers to be safely retryable since messages may be processed more than once
2. **Timeout Configuration**: Set appropriate timeouts based on expected job processing time
3. **Monitoring**: Track In-flight list size to detect stuck or slow jobs
4. **Janitor Frequency**: Balance between quick recovery and system overhead
5. **Dead Letter Queue**: Consider a separate queue for jobs that repeatedly fail after multiple retries

## References

- [Redis RPOPLPUSH Pattern](https://redis.io/commands/rpoplpush/) - Official Redis documentation on the reliable queue pattern
- [AWS SQS Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) - Managed service implementation
- [Redis University RU101](https://university.redis.com/) - Introduction to Redis Data Structures