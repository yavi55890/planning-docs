# AWS Services Integration Plan

> **Author**: SRE Automation Team  
> **Date**: March 2026  
> **Status**: Planning  
> **Related**: [Enrichment & Batch Processing Proposal](enrichment-and-batch-processing-proposal.md) · [Action Strategy Roadmap](action_strategy_roadmap.md)

---

## Table of Contents

1. [Current AWS Footprint](#current-aws-footprint)
2. [Tier 1: Immediate Priorities](#tier-1-immediate-priorities)
   - [CloudWatch Custom Metrics & Alarms](#1-cloudwatch-custom-metrics--alarms)
   - [CloudWatch Logs (Centralized Logging)](#2-cloudwatch-logs-centralized-logging)
   - [SQS Dead Letter Queue](#3-sqs-dead-letter-queue-for-failed-incidents)
3. [Tier 2: Near-Term Additions](#tier-2-near-term-additions)
   - [S3 Audit Trail & Archival](#4-s3-for-audit-trail-and-archival)
   - [SNS Notification Fan-Out](#5-sns-for-notification-fan-out)
   - [Lambda for Async SD Polling](#6-lambda-for-async-screwdriver-polling)
4. [Tier 3: Future Architecture](#tier-3-future-architecture)
   - [Step Functions Pipeline Orchestration](#7-step-functions-for-pipeline-orchestration)
   - [EventBridge Event-Driven Extensibility](#8-eventbridge-for-event-driven-extensibility)
5. [Prioritization Summary](#prioritization-summary)
6. [Implementation Notes](#implementation-notes)

---

## Current AWS Footprint

| Service | Usage | Module |
|---------|-------|--------|
| **Amazon MSK** | Kafka cluster (IAM or SCRAM auth) | `automation_consumer.py` |
| **DynamoDB** | Action history + outage cache (multi-replica) | `dynamodb_action_history.py`, `dynamodb_outage_cache.py` |
| **AWS STS** | Temporary credentials via Athenz for MSK IAM | `utilities.py` |
| **EKS** | Container orchestration (Omega) | External deployment |

**Not currently used**: CloudWatch, CloudWatch Logs, SQS, SNS, S3, Lambda, Step Functions, EventBridge, Secrets Manager, Parameter Store, X-Ray.

---

## Tier 1: Immediate Priorities

These are low-effort, high-impact services that address critical operational gaps.

### 1. CloudWatch Custom Metrics & Alarms

**Problem**: The consumer has zero operational observability. There is no way to know how fast incidents are being processed, whether Screwdriver jobs are failing at elevated rates, or if the consumer is falling behind on the Kafka stream — unless someone manually inspects log files on individual pods.

**Proposed Metrics**:

| Metric Name | Unit | Description |
|-------------|------|-------------|
| `IncidentsProcessed` | Count | Total incidents successfully processed |
| `IncidentsSkipped` | Count | Incidents skipped (outage, duplicate, unmatched, already assigned) |
| `IncidentProcessingDuration` | Seconds | End-to-end wall-clock time per incident |
| `AlertProcessingDuration` | Seconds | Per-alert processing time (including SD job wait) |
| `ScrewdriverJobSuccess` | Count | Successful SD job completions |
| `ScrewdriverJobFailure` | Count | Failed SD job completions |
| `ScrewdriverJobDuration` | Seconds | Time from SD job trigger to completion |
| `OutageDetections` | Count | Outage gating events |
| `OutageCacheHits` | Count | Early bailouts from cached outage |
| `ActionHistoryDuplicates` | Count | Incidents skipped due to prior processing |
| `EnrichmentMergeCount` | Count | (Future) Incidents merged during enrichment phase |
| `BatchAlertsProcessed` | Count | (Future) Alerts handled via batch SD action |

**Proposed Alarms**:

| Alarm | Condition | Action |
|-------|-----------|--------|
| Consumer Lag High | MSK consumer lag > N minutes | SNS → Slack notification |
| SD Failure Rate Elevated | `ScrewdriverJobFailure` / total > 20% over 15 min | SNS → Slack + PagerDuty |
| Processing Duration High | `IncidentProcessingDuration` p95 > 30 min | SNS → Slack notification |
| DLQ Messages Present | SQS `ApproximateNumberOfMessagesVisible` > 0 | SNS → Slack notification |
| Outage Spike | `OutageDetections` > 3 in 10 min | SNS → Slack notification |

**Implementation Approach**:

A thin wrapper class around `boto3.client('cloudwatch').put_metric_data()` integrated into the existing processing pipeline. Key emission points:

- `process_incident()` — duration, success/skip counts
- `run_sd_automation()` — SD job duration, success/failure
- `_should_skip_for_cached_outage()` — cache hit count
- `run_check_outage()` — outage detection count
- `_incident_already_actioned()` — duplicate count

**Sketch**:

```python
import boto3
from datetime import datetime, timezone

class ConsumerMetrics:
    """Lightweight CloudWatch metrics emitter for the Kafka consumer."""

    def __init__(self, namespace='SRE/KafkaConsumer', region='us-east-1', enabled=True):
        self.namespace = namespace
        self.enabled = enabled
        self._client = boto3.client('cloudwatch', region_name=region) if enabled else None

    def put(self, metric_name, value, unit='Count', dimensions=None):
        if not self.enabled:
            return
        metric_data = {
            'MetricName': metric_name,
            'Value': value,
            'Unit': unit,
            'Timestamp': datetime.now(timezone.utc),
        }
        if dimensions:
            metric_data['Dimensions'] = [
                {'Name': k, 'Value': str(v)} for k, v in dimensions.items()
            ]
        try:
            self._client.put_metric_data(
                Namespace=self.namespace,
                MetricData=[metric_data]
            )
        except Exception as e:
            logging.warning("Failed to emit CloudWatch metric %s: %s", metric_name, e)

    def time_and_emit(self, metric_name, dimensions=None):
        """Context manager that measures elapsed time and emits it as a metric."""
        # Returns a context manager for timing blocks of code
        ...
```

Usage in `automation_consumer.py`:

```python
start = time.time()
result = self.process_incident(bp_incident, matched_condition)
self.metrics.put(
    'IncidentProcessingDuration',
    time.time() - start,
    unit='Seconds',
    dimensions={'ConditionName': matched_condition.get('name', 'unknown')}
)
self.metrics.put('IncidentsProcessed' if result else 'IncidentsSkipped', 1)
```

**Effort**: Low — `boto3` is already a dependency. ~100 lines for the metrics class, ~30 lines of emission calls across existing methods.

**Config addition**:

```yaml
cloudwatch:
  enabled: true
  namespace: SRE/KafkaConsumer
  region: us-east-1
```

---

### 2. CloudWatch Logs (Centralized Logging)

**Problem**: Logs are written to `/var/log/kafka_consumer.log` inside each pod. In a multi-replica EKS deployment, logs are scattered across N pods with no unified way to search, correlate, or alert on log patterns.

**Solution**: Use the CloudWatch Observability EKS add-on (recommended by AWS over manual setup). It automatically ships container stdout/stderr to CloudWatch Logs.

**Required Changes**:

1. **Switch logging output to stdout/stderr** instead of file — this is the Kubernetes logging best practice regardless
2. **Install the CloudWatch Observability EKS add-on** on the cluster (one-time cluster-level setup)
3. **Configure log retention** (e.g., 30 days)

**What This Enables**:

- **CloudWatch Logs Insights** — SQL-like queries across all pod logs:
  ```
  fields @timestamp, @message
  | filter @message like /outage detected/
  | sort @timestamp desc
  | limit 50
  ```
- **Metric filters** — automatically create CloudWatch metrics from log patterns (e.g., count `ERROR` lines per minute)
- **Cross-pod correlation** — search for an incident ID and see all log lines from every pod that touched it
- **Subscription filters** — stream specific log patterns to Lambda, S3, or Elasticsearch for further analysis

**Effort**: Very low — the add-on handles log shipping. The main code change is updating the logging config to write to stdout instead of (or in addition to) a file. The `logfile` config key can remain for backward compatibility.

---

### 3. SQS Dead Letter Queue for Failed Incidents

**Problem**: When the consumer fails to process an incident and fallback actions also fail, the incident payload is silently dropped. The consumer logs an error and moves on to the next message. There is no mechanism to capture, inspect, or replay failed work.

**Solution**: Push failed incident payloads to an SQS Dead Letter Queue for later investigation and replay.

**Flow**:

```
Consumer receives incident from Kafka
    │
    ├── Processing succeeds → normal completion
    │
    └── Processing fails (including fallback failures)
            │
            ├── Push incident payload to SQS DLQ
            ├── Include failure metadata (error, phase, timestamp)
            └── CloudWatch alarm fires on DLQ depth
```

**SQS Queue Configuration**:

| Setting | Value | Rationale |
|---------|-------|-----------|
| Queue Type | Standard | Order doesn't matter for failed incidents |
| Message Retention | 14 days | Max retention for investigation time |
| Visibility Timeout | 300 seconds | Time to process a replayed message |
| Encryption | SSE-SQS | Incidents may contain sensitive data |

**Message Format**:

```json
{
  "incident_id": "abc123",
  "incident_payload": "...(original Kafka message)...",
  "failure_phase": "alert_actions",
  "error": "Screwdriver job timed out after 900s",
  "matched_condition": "oozie_failures",
  "failed_at": "2026-03-16T14:30:00Z",
  "consumer_pod": "kafka-consumer-7f8b9c-x4k2n",
  "alert_count": 4,
  "supported_alert_count": 3
}
```

**Implementation Points**:

The DLQ push would be added to `process_incident()` failure paths and the outer exception handler in `start_consumer()`:

```python
def _send_to_dlq(self, incident_payload, incident_id, failure_phase, error):
    """Push a failed incident to the SQS dead letter queue."""
    if not self.dlq_url:
        return
    try:
        self.sqs_client.send_message(
            QueueUrl=self.dlq_url,
            MessageBody=json.dumps({
                'incident_id': incident_id,
                'incident_payload': incident_payload,
                'failure_phase': failure_phase,
                'error': str(error),
                'failed_at': datetime.now(timezone.utc).isoformat(),
                'consumer_pod': socket.gethostname(),
            }),
            MessageAttributes={
                'IncidentId': {'StringValue': incident_id, 'DataType': 'String'},
                'FailurePhase': {'StringValue': failure_phase, 'DataType': 'String'},
            }
        )
    except Exception as e:
        logging.error("Failed to send incident %s to DLQ: %s", incident_id, e)
```

**Replay Strategy**: A separate script or Lambda function that reads messages from the DLQ and re-publishes them to the Kafka topic (or directly invokes processing). This can be triggered manually after the root cause is fixed.

**Effort**: Low — SQS client calls are straightforward. Queue provisioning is a one-time setup (Terraform/CloudFormation or console).

**Config addition**:

```yaml
dead_letter_queue:
  enabled: true
  queue_url: "https://sqs.us-east-1.amazonaws.com/123456789/kafka-consumer-dlq"
  region: us-east-1
```

---

## Tier 2: Near-Term Additions

Medium-effort services that provide strong value once the Tier 1 foundation is in place.

### 4. S3 for Audit Trail and Archival

**Use Case 1 — Processing Audit Trail**:

After each incident is processed (success or failure), write a structured JSON record to S3:

```
s3://sre-automation-audit/kafka-consumer/
  └── 2026/03/16/
      ├── incident-abc123-success.json
      └── incident-def456-failure.json
```

Each record includes: incident ID, matched condition, actions executed, per-action results, timing data, alert details, and the original payload. This provides a durable, searchable audit trail for post-mortems, compliance, and pattern analysis.

**Use Case 2 — DynamoDB Record Archival**:

The action history DynamoDB table uses TTL (`ttl_days`) to expire old records. When TTL fires, records are permanently deleted. With DynamoDB Streams → Lambda → S3, expired records can be archived to S3 before deletion — cold storage at nearly zero cost.

**Use Case 3 — SD Job Output Storage**:

Store Screwdriver job outputs (build logs, per-host results) in S3, linked by incident ID. Useful for debugging failed batch jobs.

**Effort**: Low-medium. S3 writes are trivial. The DynamoDB Streams archival pipeline is a well-documented AWS pattern.

---

### 5. SNS for Notification Fan-Out

**Problem**: Every notification is a direct Slack webhook call hardcoded in the action runner. Adding a new notification channel (PagerDuty, email, a second Slack channel) requires code changes.

**Solution**: Publish to an SNS topic instead of calling Slack directly. SNS fans out to multiple subscribers:

```
Consumer publishes to SNS topic
    ├── Subscriber 1: Lambda → Slack webhook
    ├── Subscriber 2: Email to sre-oncall@
    ├── Subscriber 3: SQS queue for audit logging
    └── Subscriber 4: (future) PagerDuty integration
```

Adding a new notification channel becomes a subscriber configuration change — zero code changes in the consumer.

**Consideration**: This adds a layer of indirection. For teams that only need Slack, the direct webhook approach is simpler. SNS makes sense when notification routing needs to be flexible or when multiple channels are required.

**Effort**: Low-medium. Replace `send_slack_msg()` calls with SNS publishes. Set up a Lambda function to forward SNS messages to Slack webhooks.

---

### 6. Lambda for Async Screwdriver Polling

**Problem**: The consumer blocks for the entire duration of each Screwdriver job (often 10+ minutes per job). During this time, no other incidents can be processed.

**Solution**: Offload SD job monitoring to a Lambda function. The consumer triggers the SD job, invokes a Lambda to poll it, and moves on immediately.

**Flow**:

```
Consumer:
  1. Trigger SD job → get event_id
  2. Invoke Lambda (async) with event_id + callback info
  3. Record "pending" status in DynamoDB
  4. Move on to next incident immediately

Lambda:
  1. Poll SD job at configured interval
  2. On completion: write result to DynamoDB
  3. Optionally: send SNS notification with result

Consumer (later, or via notification):
  1. Check DynamoDB for completed jobs
  2. Run completion/fallback actions based on result
```

**This is essentially the `screwdriver_async` action type from the [Action Strategy Roadmap](action_strategy_roadmap.md)**, but using Lambda as the polling agent instead of building async polling into the consumer.

**Advantages over in-process async**:
- Consumer is completely decoupled from SD job duration
- Lambda scales independently — can poll many jobs concurrently
- Lambda handles its own retries and timeouts
- No threading or asyncio complexity in the consumer

**Effort**: Medium. Requires a Lambda function (Python, reusing `sdv4_tools`), IAM roles, DynamoDB schema for tracking pending jobs, and a new result-checking mechanism in the consumer.

---

## Tier 3: Future Architecture

Higher-effort services that represent a more fundamental architectural evolution.

### 7. Step Functions for Pipeline Orchestration

**Concept**: Model the entire `process_incident()` pipeline as an AWS Step Functions state machine. Each phase becomes a state with built-in retry, error handling, and parallelism.

**State Machine Design**:

```
                    ┌──────────────┐
                    │  Start       │
                    │  (receive    │
                    │   incident)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Enrichment  │
                    │  (parallel)  │
                    │  ├─ merge    │
                    │  └─ outage   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Assignment  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Incident    │
                    │  Actions     │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Batch/Alert │
                    │  Processing  │◄──── Wait state for SD job
                    └──────┬───────┘
                           │
                   ┌───────┴───────┐
                   │               │
            ┌──────▼──────┐ ┌──────▼──────┐
            │  Completion │ │  Fallback   │
            │  Actions    │ │  Actions    │
            └─────────────┘ └─────────────┘
```

**What Step Functions Provides for Free**:

- **Visual debugging** — see exactly where an incident is in the pipeline, in real-time, in the AWS console
- **Built-in retry with exponential backoff** — per-state retry configuration, no manual retry logic
- **Parallel states** — native support for batch/parallel alert processing
- **Wait states** — poll SD jobs without blocking anything
- **Error handling** — catch, retry, and fallback at every state boundary
- **Execution history** — full audit trail of every state transition for 90 days

**Architectural Shift**: The Kafka consumer becomes a thin dispatcher that reads messages and starts Step Functions executions. All processing logic moves into Lambda functions (one per state) or direct SDK integrations. The consumer never blocks.

**Trade-offs**:
- Significant refactor — processing logic must be decomposed into stateless functions
- State must be passed between states via the execution input/output (max 256 KB)
- Standard Workflows cost $0.025 per 1,000 state transitions
- Adds operational complexity (more moving parts to monitor)

**Use Standard Workflows** (not Express) — incident processing can take 10-30+ minutes, exceeding Express's 5-minute limit.

**Effort**: High — multi-week effort. Best approached as a separate project after Tier 1 and Tier 2 are stable.

---

### 8. EventBridge for Event-Driven Extensibility

**Concept**: Emit structured events at each processing phase to an EventBridge event bus. Other services subscribe to events they care about, completely decoupled from the consumer.

**Events Emitted**:

| Event | Detail Type | When |
|-------|------------|------|
| `incident.matched` | Incident matched a condition | After condition matching |
| `incident.enriched` | Alerts merged/expanded | After enrichment phase |
| `incident.assigned` | Incident assigned to service account | After assignment |
| `incident.actions_complete` | Incident-level actions finished | After Phase 4 |
| `alerts.processing` | Alert processing started | Start of Phase 6 |
| `alerts.complete` | All alerts processed | End of Phase 6 |
| `incident.resolved` | Incident resolved successfully | After resolution |
| `incident.failed` | Processing failed, fallback triggered | On failure |
| `outage.detected` | Outage gated the pipeline | On outage detection |

**Use Cases**:

- **Dashboarding service** subscribes to `incident.resolved` and `incident.failed` to track success rates
- **Compliance service** subscribes to all events for audit trail
- **ML pipeline** subscribes to events to build pattern recognition for future agentic workflows
- **Cross-team notifications** — other teams subscribe to events relevant to their services

**Effort**: Medium-high. Event emission is simple (`boto3.client('events').put_events()`), but the value depends on building consumers for these events.

---

## Prioritization Summary

| Priority | Service | Effort | Impact | Status |
|----------|---------|--------|--------|--------|
| **P0** | CloudWatch Metrics + Alarms | Low | High | Not started |
| **P0** | CloudWatch Logs | Very Low | High | Not started |
| **P0** | SQS Dead Letter Queue | Low | High | Not started |
| **P1** | S3 Audit Trail | Low-Med | Medium | Not started |
| **P1** | Lambda Async SD Polling | Medium | High | Not started |
| **P2** | SNS Fan-Out | Low-Med | Medium | Not started |
| **P3** | Step Functions | High | Very High | Not started |
| **P3** | EventBridge | Med-High | Medium | Not started |

---

## Implementation Notes

### IAM Permissions

All new services require IAM permissions for the EKS pods (via IRSA). The existing IRSA role used for DynamoDB would need to be extended:

| Service | Required Permissions |
|---------|---------------------|
| CloudWatch | `cloudwatch:PutMetricData`, `cloudwatch:DescribeAlarms` |
| CloudWatch Logs | Handled by EKS add-on (pod-level: stdout only) |
| SQS | `sqs:SendMessage`, `sqs:GetQueueAttributes` |
| S3 | `s3:PutObject`, `s3:GetObject` on the audit bucket |
| SNS | `sns:Publish` on the notification topic |
| Lambda | `lambda:InvokeFunction` on the SD polling function |

### Config Schema Extensions

New top-level config sections for each service:

```yaml
# CloudWatch metrics
cloudwatch:
  enabled: true
  namespace: SRE/KafkaConsumer
  region: us-east-1

# Dead letter queue
dead_letter_queue:
  enabled: true
  queue_url: "https://sqs.us-east-1.amazonaws.com/123456789/kafka-consumer-dlq"
  region: us-east-1

# Audit trail
audit:
  enabled: true
  s3_bucket: sre-automation-audit
  s3_prefix: kafka-consumer/
  region: us-east-1

# Notification fan-out (optional, replaces direct Slack webhooks)
notifications:
  backend: slack          # "slack" (direct) or "sns" (fan-out)
  sns_topic_arn: "arn:aws:sns:us-east-1:123456789:sre-automation-notifications"
```

### Dependencies

No new Python dependencies are required — `boto3` (already installed) provides clients for all proposed services.

---

## References

- [Enrichment & Batch Processing Proposal](enrichment-and-batch-processing-proposal.md) — Pipeline enrichment phase and batch SD action
- [Action Strategy Roadmap](action_strategy_roadmap.md) — Full roadmap including `screwdriver_async` and parallel processing
- [Blocking Consumer Problem](blocking-consumer-problem.md) — Analysis of the consumer blocking issue
- [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/container-insights-detailed-metrics.html) — EKS observability add-on
- [SQS Dead Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — AWS documentation
- [Step Functions Standard Workflows](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-standard-vs-express.html) — Standard vs Express comparison
