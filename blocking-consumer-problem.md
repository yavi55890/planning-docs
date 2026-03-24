# Blocking Consumer Problem: `max.poll.interval.ms` Exceeded During Screwdriver Polling

## Problem Statement

When the Kafka consumer triggers a Screwdriver (SD) job as part of incident automation, it blocks the main thread while waiting for the SD event to complete via `sd_event_done_wait()`. If the SD job takes longer than `max.poll.interval.ms` (default: 600,000ms / 10 minutes), the consumer is ejected from the consumer group. The affected pod becomes a zombie — alive but unable to receive messages — while the remaining consumers absorb its partition(s).

## Observed Behavior (Dev Cluster — 2026-03-16)

### Setup

- 3 consumer pods scaled up, 3 Kafka partitions on the SRE topic
- DynamoDB-backed action history

### What Happened

1. Two consumers each picked up an incident. The third consumer sat idle (expected — message key hashing distributed data across only 2 of 3 partitions).
2. Three SD jobs were triggered total. One ran successfully, one collapsed (SD blocked it due to concurrent pipeline constraints), and one ran after the first completed. **2 out of 3 jobs ran, 1 collapsed.**
3. On the pod where the SD job collapsed, the consumer was stuck in `sd_event_done_wait()` polling the COLLAPSED event every 60 seconds. After 10 minutes of no `poll()` calls, librdkafka ejected the consumer from the group:

```
%4|1773686361.180|MAXPOLL|kafka-consumer--development-use1-74f99f59c4-ctgzs#consumer-1|
[thrd:main]: Application maximum poll interval (600000ms) exceeded by 174ms
(adjust max.poll.interval.ms for long-running message processing): leaving group
```

4. The consumer shut down, the pod restarted, but then got stuck again at `"Stopping event 54820914..."` — either the SD stop-event API call hung, or the consumer entered an unrecoverable state.
5. **Result:** This pod stopped processing messages entirely. The other two consumers handled all subsequent incidents (5+ more fired), and this pod's logs remained frozen.

### Log Timeline

| Time | Event |
|------|-------|
| 18:29:29 | SD event 54820914 triggered |
| ~18:29–18:39 | `sd_event_done_wait()` blocks main thread, polling SD every 60s |
| 18:39:21 | `max.poll.interval.ms` (600,000ms) exceeded — consumer leaves group |
| 18:39:23 | SSL connection closed by broker |
| 18:39:31 | Consumer shutdown triggered, `consumer.close()` called |
| Post-restart | SD event polling resumes: 54820600 (QUEUED → RUNNING → failed), then 54820914 (QUEUED → BLOCKED → COLLAPSED × 10 polls) |
| +600s | SD event 54820914 timed out, `"Stopping event..."` — logs freeze here |

## Root Cause

The consumer loop in `automation_consumer.py` is **synchronous and single-threaded**. The call chain is:

```
start_consumer() → poll() → process_incident() → action_runner.run_sd_automation() → sd_event_done_wait()
```

`sd_event_done_wait()` is a fully blocking call that polls SD every `polling_interval` seconds (60s) until the job completes, fails, or reaches `polling_timeout` (600s). During this entire window, the consumer never calls `poll()`, so:

- `max.poll.interval.ms` is exceeded
- The consumer is removed from the group
- Partitions are rebalanced to remaining consumers
- The affected pod becomes unresponsive

This is not a configuration issue — it's an architectural one. The blocking call and the Kafka heartbeat mechanism are fundamentally incompatible.

## Why Increasing `max.poll.interval.ms` Doesn't Fix It

Setting `max.poll.interval.ms` to 30 or 60 minutes only delays the problem. SD jobs can be QUEUED, BLOCKED, or COLLAPSED for unpredictable durations. The `polling_timeout` (currently 600s) caps the wait, but:

- A COLLAPSED job sits in the polling loop for the full timeout before giving up
- Multiple SD jobs triggered sequentially (per alert) compound the blocking time
- The timeout is per SD event — processing an incident with N alerts means up to N × 600s of blocking

## Proposed Solutions

### Option 1: Background Thread + `pause()`/`resume()` (Recommended)

Before starting SD polling, **pause** the assigned Kafka partition(s) and run `sd_event_done_wait()` in a background thread. The main loop continues calling `poll()` (which returns nothing since the partition is paused) to maintain group membership. When the background thread finishes, **resume** the partition.

**Pros:**
- Consumer stays in the group — no rebalances, no zombie pods
- No new messages are delivered while processing (backpressure is preserved)
- Cleanest separation of concerns

**Cons:**
- Adds threading complexity
- Need to handle edge cases: what if the consumer is shut down while the background thread is running?

**Sketch:**

```python
import threading
from confluent_kafka import TopicPartition

def process_with_heartbeat(self, action_fn, *args):
    """Run a blocking action in a background thread while keeping the consumer alive."""
    assigned = self.consumer.assignment()
    self.consumer.pause(assigned)

    result = [None]
    error = [None]

    def worker():
        try:
            result[0] = action_fn(*args)
        except Exception as e:
            error[0] = e

    thread = threading.Thread(target=worker)
    thread.start()

    while thread.is_alive():
        self.consumer.poll(1.0)  # maintains group membership
        thread.join(timeout=1.0)

    self.consumer.resume(assigned)

    if error[0]:
        raise error[0]
    return result[0]
```

### Option 2: Fire-and-Forget + Deferred Status Check

Trigger the SD job, record the event ID in the action history (DynamoDB), and immediately return to the consumer loop. A separate mechanism (background thread, scheduled task, or even a separate microservice) periodically checks pending SD events and updates their status.

**Pros:**
- Consumer is never blocked — maximum throughput
- Naturally scales to many concurrent SD jobs

**Cons:**
- Requires rethinking the action result flow — current logic depends on synchronous success/failure to decide fallback actions
- More moving parts (separate checker process or thread)
- Action sequencing (alert_actions → fallback_actions → completion_actions) becomes eventual rather than immediate

### Option 3: Interleaved Polling Loop

Replace `sd_event_done_wait()` with a custom loop that alternates between checking SD status and calling `consumer.poll()`.

**Pros:**
- Simple, no threads
- Consumer stays in the group

**Cons:**
- Mixes Kafka polling with SD polling — messy
- Still blocks new message processing (poll returns messages that pile up)
- Would need to buffer or ignore new messages during SD wait

**Sketch:**

```python
def poll_sd_with_heartbeat(self, sd_tools, event_id, polling_interval, polling_timeout):
    """Poll SD event status while keeping the Kafka consumer alive."""
    start = time.time()
    while time.time() - start < polling_timeout:
        status = sd_tools.get_event_status(event_id)
        if status in ('SUCCESS', 'FAILURE', 'ABORTED', 'COLLAPSED'):
            return status == 'SUCCESS'
        # Keep consumer alive
        self.consumer.poll(0)
        time.sleep(polling_interval)
    return False
```

### Option 4: Reduce SD Polling Timeout

Lower `polling_timeout` to something well under `max.poll.interval.ms` (e.g., 300s with a 600s max poll interval). If the SD job hasn't completed by then, treat it as a failure and move on.

**Pros:**
- Zero code changes beyond config
- Prevents the worst-case blocking scenario

**Cons:**
- Legitimate long-running SD jobs will be incorrectly marked as failed
- Doesn't solve the fundamental problem — just shrinks the window
- Still blocks the consumer for up to `polling_timeout` per SD event

## Recommendation

**Option 1 (Background Thread + pause/resume)** is the best balance of correctness and complexity. It solves the root cause without requiring architectural changes to the action flow, and `confluent_kafka` natively supports `pause()`/`resume()` for exactly this kind of use case.

The implementation should:

1. Apply to any action type that could block for extended periods (SD jobs, potentially long webhooks)
2. Handle graceful shutdown — if a SIGTERM arrives while the background thread is running, wait for it or cancel cleanly
3. Log when partitions are paused/resumed for observability

## Related Configuration

| Setting | Current Value | Notes |
|---------|--------------|-------|
| `max.poll.interval.ms` | 600,000 (10 min) | Cannot be increased indefinitely |
| `polling_timeout` (SD) | 600s | Per SD event — compounds with multiple alerts |
| `polling_interval` (SD) | 60s | How often SD status is checked |
| `enable.auto.commit` | True | Offsets are committed on `poll()` — blocked consumer doesn't commit |
