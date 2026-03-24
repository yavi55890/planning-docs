# CloudWatch Integration

> **Branch**: `feature/cloudwatch-metrics-and-logs`  
> **Date**: March 2026  
> **Status**: Implemented (not yet merged)  
> **Related**: [AWS Services Integration Plan](aws-services-integration-plan.md) · [Action Strategy Roadmap](action_strategy_roadmap.md)

---

## Table of Contents

1. [Overview](#overview)
2. [YAML Configuration](#yaml-configuration)
3. [Feature 1: CloudWatch Custom Metrics](#feature-1-cloudwatch-custom-metrics)
   - [ConsumerMetrics Class](#consumermetrics-class)
   - [Metrics Emitted](#metrics-emitted)
   - [Where Metrics Are Emitted](#where-metrics-are-emitted)
   - [Dimensions](#dimensions)
4. [Feature 2: CloudWatch Logs](#feature-2-cloudwatch-logs)
   - [How It Works](#how-it-works)
   - [Logging Behavior Changes](#logging-behavior-changes)
5. [Config Validation](#config-validation)
6. [Terraform Infrastructure Changes](#terraform-infrastructure-changes)
   - [CloudWatch Log Group](#cloudwatch-log-group)
   - [IAM Policy](#iam-policy)
7. [EKS Deployment Changes](#eks-deployment-changes)
8. [Files Changed](#files-changed)
9. [How to Enable](#how-to-enable)
10. [How to Disable](#how-to-disable)
11. [Prerequisites](#prerequisites)

---

## Overview

This change adds two opt-in observability features to the Kafka consumer, both controlled by a new `cloudwatch` section in the YAML configuration:

1. **CloudWatch Custom Metrics** — emits operational metrics (incident processing times, Screwdriver job success/failure counts, outage detections, etc.) to Amazon CloudWatch via the `PutMetricData` API.

2. **CloudWatch Logs** — switches the consumer to stdout-only logging so that the CloudWatch Observability EKS add-on can automatically ship logs to CloudWatch Logs. No CloudWatch SDK calls are made for logging; the EKS add-on handles log shipping transparently.

Both features default to **disabled**. They are designed for EKS deployments where the consumer runs as a multi-replica pod with IRSA credentials. Non-EKS deployments (single-server, local dev) can simply omit the `cloudwatch` section and nothing changes.

No new Python dependencies are required — `boto3` is already installed.

---

## YAML Configuration

Add a `cloudwatch` section to the consumer's YAML config file. All fields have sensible defaults; only `metrics_enabled` and `logs_enabled` are needed to activate each feature.

```yaml
cloudwatch:
  metrics_enabled: true          # Emit custom metrics to CloudWatch (default: false)
  logs_enabled: true             # Switch to stdout-only logging for CloudWatch Logs (default: false)
  namespace: SRE/KafkaConsumer   # CloudWatch metrics namespace (default: SRE/KafkaConsumer)
  region: us-east-1              # AWS region for CloudWatch API calls (default: us-east-1)
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `metrics_enabled` | bool | `false` | When `true`, the consumer emits custom metrics to CloudWatch |
| `logs_enabled` | bool | `false` | When `true`, the file log handler is skipped (stdout only) |
| `namespace` | string | `SRE/KafkaConsumer` | CloudWatch namespace for all emitted metrics |
| `region` | string | `us-east-1` | AWS region for the CloudWatch API client |

If the `cloudwatch` section is absent entirely, both features are disabled with zero overhead.

---

## Feature 1: CloudWatch Custom Metrics

### ConsumerMetrics Class

A new module `consumer_metrics.py` provides the `ConsumerMetrics` class. It wraps `boto3.client('cloudwatch')` with two methods:

- **`put(metric_name, value, unit, dimensions)`** — emits a single metric data point. When disabled, this is a no-op that returns immediately.
- **`timed(metric_name, dimensions)`** — a context manager that measures wall-clock time and emits the elapsed duration as a `Seconds` metric.

The class is instantiated once in `AutomationConsumer.__init__()` and passed to `ActionRunner` so both layers emit through the same client:

```python
# In AutomationConsumer.__init__()
cw_config = yaml_config.get('cloudwatch', {})
self.metrics = ConsumerMetrics(
    enabled=cw_config.get('metrics_enabled', False),
    namespace=cw_config.get('namespace', 'SRE/KafkaConsumer'),
    region=cw_config.get('region', 'us-east-1'),
)

# Passed to ActionRunner
self.action_runner = ActionRunner(yaml_config, metrics=self.metrics)
```

When `metrics_enabled` is `false` (the default), the `boto3` client is never created and `put()` / `timed()` return immediately — zero AWS calls, zero overhead.

When `metrics_enabled` is `true`, `boto3` is imported and a CloudWatch client is created. Each `put()` call makes a `PutMetricData` API call. Failures are logged as warnings and do not interrupt incident processing.

### Metrics Emitted

| Metric Name | Unit | Description |
|-------------|------|-------------|
| `IncidentsProcessed` | Count | Incremented when an incident is successfully processed to completion |
| `IncidentsSkipped` | Count | Incremented when an incident is skipped (outage, duplicate, assignment conflict, action failure) |
| `IncidentProcessingDuration` | Seconds | Wall-clock time for the entire `process_incident()` call, from start to finish |
| `AlertProcessingDuration` | Seconds | Wall-clock time for processing a single alert (including Screwdriver job wait) |
| `ScrewdriverJobSuccess` | Count | Incremented when a Screwdriver job completes successfully |
| `ScrewdriverJobFailure` | Count | Incremented when a Screwdriver job fails or times out |
| `ScrewdriverJobDuration` | Seconds | Wall-clock time from SD job trigger to completion (or failure/timeout) |
| `OutageDetections` | Count | Incremented when `check_outage` detects an outage (alert threshold exceeded) |
| `OutageCacheHits` | Count | Incremented when an incident is skipped due to a cached outage detection |
| `ActionHistoryDuplicates` | Count | Incremented when an incident is skipped because its alerts were already processed |

### Where Metrics Are Emitted

Each metric is emitted from a specific point in the processing pipeline:

**In `automation_consumer.py`:**

| Method | Metric(s) | Trigger |
|--------|-----------|---------|
| `start_consumer()` | `IncidentProcessingDuration`, `IncidentsProcessed`, `IncidentsSkipped` | Wraps `process_incident()` with `timed()`, then emits processed/skipped based on result |
| `_incident_already_actioned()` | `ActionHistoryDuplicates` | Emitted when returning `True` (alert already in action history) |
| `_should_skip_for_cached_outage()` | `OutageCacheHits` | Emitted when returning `True` (outage cache overlap found) |
| `_process_all_alerts()` | `AlertProcessingDuration` | Each alert's `initiate_alert_actions()` call is wrapped with `timed()` |

**In `action_runner.py`:**

| Method | Metric(s) | Trigger |
|--------|-----------|---------|
| `run_sd_automation()` | `ScrewdriverJobDuration`, `ScrewdriverJobSuccess` or `ScrewdriverJobFailure` | Duration is measured from method entry to after `sd_event_done_wait()`. Success/failure emitted based on `event_successful` |
| `run_check_outage()` | `OutageDetections` | Emitted when `total_alerts > alert_threshold` (outage detected) |

### Dimensions

Some metrics include CloudWatch dimensions for filtering and grouping:

| Metric | Dimension | Value |
|--------|-----------|-------|
| `IncidentProcessingDuration` | `ConditionName` | Name of the matched incident condition (e.g., `oozie_failures`) |
| `IncidentsProcessed` | `ConditionName` | Same as above |
| `IncidentsSkipped` | `ConditionName` | Same as above |
| `ScrewdriverJobDuration` | `JobName` | Screwdriver job name (e.g., `oozie_apollo`) |
| `ScrewdriverJobSuccess` | `JobName` | Same as above |
| `ScrewdriverJobFailure` | `JobName` | Same as above |

Dimensions allow CloudWatch dashboards and alarms to filter metrics by condition or job name.

---

## Feature 2: CloudWatch Logs

### How It Works

The consumer already logs to both a file and stdout (console). When `cloudwatch.logs_enabled` is `true`, the file handler is skipped — the consumer logs only to stdout. The CloudWatch Observability EKS add-on automatically captures stdout from all containers and ships it to CloudWatch Logs.

This approach requires **no CloudWatch SDK integration for logging**. The consumer simply writes to stdout; the EKS add-on handles the rest.

### Logging Behavior Changes

| `logs_enabled` | File Handler | Console Handler | Log Destination |
|----------------|-------------|-----------------|-----------------|
| `false` (default) | Created (writes to `global.logfile`) | Created (stdout) | File + stdout |
| `true` | **Skipped** | Created (stdout) | stdout only (CloudWatch Logs via EKS add-on) |

The change is in `set_logger()` in `utilities.py`:

```python
def set_logger(logfile: str, log_level: int = logging.INFO, cloudwatch_logs_only: bool = False):
    # Console handler is always added (stdout)
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)

    # File handler is skipped when CloudWatch Logs captures stdout instead
    if not cloudwatch_logs_only and logfile:
        file_handler = logging.FileHandler(logfile)
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)
```

In `main.py`, the CloudWatch config is read and passed to `set_logger()`:

```python
cw_config = yaml_config.get('cloudwatch', {})
cloudwatch_logs_only = cw_config.get('logs_enabled', False)
set_logger(yaml_config["global"].get("logfile", ""), log_level=log_level,
           cloudwatch_logs_only=cloudwatch_logs_only)
```

The log format remains unchanged:
```
[PID][LEVEL][TIMESTAMP][module.function][line] - message
```

---

## Config Validation

A new `_validate_cloudwatch_section()` method was added to `config_validator.py`, following the same pattern as `_validate_storage_section()`:

- `cloudwatch` must be a dictionary
- `metrics_enabled` and `logs_enabled` must be booleans (if present)
- When `metrics_enabled` is `true`, `namespace` and `region` must be strings (if present)

The validation is wired into `validate_config()` and only runs when a `cloudwatch` section exists in the config:

```python
if "cloudwatch" in config:
    self._validate_cloudwatch_section(config["cloudwatch"])
```

Configs without a `cloudwatch` section pass validation without any changes.

---

## Terraform Infrastructure Changes

**Repository**: `sre-terraform-kafka-consumer-eks`  
**File**: `terraform/global-common/cloudwatch.tf` (new)

### CloudWatch Log Group

A CloudWatch Logs log group is created for the consumer pods:

```hcl
resource "aws_cloudwatch_log_group" "consumer_logs" {
  name              = "/eks/kafka-consumer/${local.cloudwatch_env_suffix}"
  retention_in_days = var.environment == "prod" ? 30 : 7
}
```

- **Prod**: `/eks/kafka-consumer/prod` with 30-day retention
- **Dev**: `/eks/kafka-consumer/dev` with 7-day retention

This log group is the target for the CloudWatch Observability EKS add-on to ship container stdout logs into.

### IAM Policy

A `cloudwatch-access` inline policy is attached to every IRSA role (the same roles used for DynamoDB access). It grants `cloudwatch:PutMetricData` permission, scoped to the `SRE/KafkaConsumer` namespace using a condition key:

```hcl
resource "aws_iam_role_policy" "cloudwatch_access" {
  for_each = data.aws_iam_role.kafka_consumer_irsa
  name     = "cloudwatch-access"
  role     = each.value.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Action    = ["cloudwatch:PutMetricData"]
        Resource  = ["*"]
        Condition = {
          StringEquals = {
            "cloudwatch:namespace" = "SRE/KafkaConsumer"
          }
        }
      }
    ]
  })
}
```

**Why `Resource = "*"`**: The `PutMetricData` API does not support resource-level ARNs. The `cloudwatch:namespace` condition restricts the permission so the pods can only write metrics to the `SRE/KafkaConsumer` namespace.

**CloudWatch Logs permissions are NOT included**: The EKS add-on uses its own service account and IAM role to ship logs. The consumer pods do not need CloudWatch Logs permissions.

The policy reuses `data.aws_iam_role.kafka_consumer_irsa` from `dynamodb.tf`. Terraform resolves cross-file references within the same module automatically, so no refactoring of `dynamodb.tf` is needed.

---

## EKS Deployment Changes

**Repository**: `yahoo.kafka_consumer-k8s-eks-deploy`  
**Files modified**: `deploy_target/omega/config/config-development.yaml`, `deploy_target/omega/config/config-production.yaml`

The `cloudwatch` section was added to both environment configs, placed after the `storage` section:

```yaml
cloudwatch:
  metrics_enabled: true
  logs_enabled: true
  namespace: SRE/KafkaConsumer
  region: us-east-1
```

No changes were needed to `omega.yaml`, `app_start`, or any other deployment scripts. The consumer reads the CloudWatch config from the YAML file at startup, and IRSA credentials are provided automatically by the EKS platform.

---

## Files Changed

### `yahoo.kafka_consumer` (consumer app)

| File | Change Type | Description |
|------|-------------|-------------|
| `src/yahoo/kafka_consumer/consumer_metrics.py` | **New** | `ConsumerMetrics` class with `put()` and `timed()` methods. No-op when disabled. Uses lazy `boto3` import. |
| `src/yahoo/kafka_consumer/automation_consumer.py` | Modified | Imports and initializes `ConsumerMetrics`. Emits `IncidentProcessingDuration`, `IncidentsProcessed`, `IncidentsSkipped`, `ActionHistoryDuplicates`, `OutageCacheHits`, `AlertProcessingDuration`. Passes `metrics` to `ActionRunner`. |
| `src/yahoo/kafka_consumer/action_runner.py` | Modified | Accepts optional `metrics` parameter in `__init__()`. Emits `ScrewdriverJobDuration`, `ScrewdriverJobSuccess`, `ScrewdriverJobFailure` in `run_sd_automation()`. Emits `OutageDetections` in `run_check_outage()`. |
| `src/yahoo/kafka_consumer/utilities.py` | Modified | `set_logger()` accepts `cloudwatch_logs_only` parameter. When `True`, skips file handler. |
| `src/yahoo/kafka_consumer/main.py` | Modified | Reads `cloudwatch.logs_enabled` from config and passes to `set_logger()`. |
| `src/yahoo/kafka_consumer/config_validator.py` | Modified | New `_validate_cloudwatch_section()` method. Wired into `validate_config()`. |
| `src/yahoo/kafka_consumer/example-config.yaml` | Modified | Added commented `cloudwatch` section with documentation. |

### `sre-terraform-kafka-consumer-eks` (Terraform)

| File | Change Type | Description |
|------|-------------|-------------|
| `terraform/global-common/cloudwatch.tf` | **New** | CloudWatch log group (env-based name, retention), IAM policy for `PutMetricData` scoped to `SRE/KafkaConsumer` namespace on all IRSA roles. |

### `yahoo.kafka_consumer-k8s-eks-deploy` (EKS deployment)

| File | Change Type | Description |
|------|-------------|-------------|
| `deploy_target/omega/config/config-development.yaml` | Modified | Added `cloudwatch` section with metrics and logs enabled. |
| `deploy_target/omega/config/config-production.yaml` | Modified | Added `cloudwatch` section with metrics and logs enabled. |

---

## How to Enable

1. **Terraform**: Apply the `cloudwatch.tf` changes to create the log group and attach IAM policies to IRSA roles.

2. **YAML Config**: Add the `cloudwatch` section to the consumer's config file:
   ```yaml
   cloudwatch:
     metrics_enabled: true
     logs_enabled: true
     namespace: SRE/KafkaConsumer
     region: us-east-1
   ```

3. **Deploy**: Deploy the updated consumer image with the new code, along with the updated config.

4. **Verify**: Check CloudWatch in the AWS console:
   - **Metrics**: Navigate to CloudWatch > Metrics > Custom Namespaces > `SRE/KafkaConsumer`
   - **Logs**: Navigate to CloudWatch > Log Groups > `/eks/kafka-consumer/prod` (or `/dev`)

---

## How to Disable

Remove the `cloudwatch` section from the YAML config, or set individual features to `false`:

```yaml
cloudwatch:
  metrics_enabled: false
  logs_enabled: false
```

When disabled:
- No `boto3` CloudWatch client is created
- No `PutMetricData` API calls are made
- Logging reverts to the default dual-handler behavior (file + stdout)
- Zero runtime overhead

---

## Prerequisites

- **IRSA roles** must have the `cloudwatch-access` IAM policy attached (handled by Terraform)
- **CloudWatch Observability EKS add-on** must be installed on the EKS cluster for log shipping (cluster-level setup, not per-app). Confirm with the platform team whether this is enabled on `omega-aws.centraltech-nonprod1.use1` and `omega-aws.centraltech-prod1.use1`.
- **boto3** must be installed (already a dependency in `setup.cfg`)

---

## References

- [AWS Services Integration Plan](aws-services-integration-plan.md) — Full AWS services analysis and prioritization
- [CloudWatch PutMetricData API](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html) — AWS documentation
- [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/container-insights-detailed-metrics.html) — EKS observability add-on
- [consumer_metrics.py](../src/yahoo/kafka_consumer/consumer_metrics.py) — Metrics emitter implementation
- [automation_consumer.py](../src/yahoo/kafka_consumer/automation_consumer.py) — Consumer pipeline with metrics integration
- [action_runner.py](../src/yahoo/kafka_consumer/action_runner.py) — Action execution with metrics integration
