# Scaling the Kafka Consumer to Multiple Replicas with DynamoDB

## The Problem

The Kafka consumer maintains an **action history** -- a record of every alert it has already processed. This prevents the consumer from re-running remediation actions (Screwdriver jobs, Slack notifications, etc.) on alerts it has already handled. It also maintains an **outage cache** that tracks ongoing outage detections to avoid redundant BigPanda API calls and duplicate Slack notifications during alert floods.

### How It Worked Before (Single Replica)

Both of these were stored locally:

- **Action History**: A `FileBackedDict` -- a Python `dict` subclass that rewrites a JSON file to disk on every mutation. The file lived on a PVC (Persistent Volume Claim) backed by AWS EFS.
- **Outage Cache**: A plain in-memory Python dictionary. Lost on pod restart, but acceptable for a single short-lived cache.

The `FileBackedDict` implementation (`file_backed_dict.py`) works like this:

```python
class FileBackedDict(dict):
    def __setitem__(self, key, value):
        super().__setitem__(key, value)
        self._write_to_file()

    def _write_to_file(self):
        with open(self.filename, 'w', encoding='utf-8') as file:
            json.dump(self, file, indent=4)
```

Every single write opens the file, **truncates it entirely**, and rewrites the full dictionary as JSON.

### Why This Breaks With Multiple Replicas

When Kafka scales to multiple consumers in the same consumer group, partitions are distributed among pods. Each pod processes different alerts. But they all need to check the same deduplication state -- otherwise pod A doesn't know that pod B already handled an alert during a previous partition assignment.

With `FileBackedDict` on a shared EFS volume:

1. **Lost writes**: Pod A reads the file into memory, Pod B reads the same file. Pod A writes alert-1, Pod B writes alert-2. Pod B's write truncates the file and rewrites its version -- alert-1 is gone.
2. **Torn writes / JSON corruption**: `open('w')` truncates immediately. If Pod A crashes mid-write (or the EFS flush is slow), the file is left with a partial JSON document.
3. **Silent data loss**: `FileBackedDict` catches `JSONDecodeError` by overwriting the corrupt file with `{}` -- wiping the entire history.

The outage cache has a different but related problem: it's purely in-memory. With multiple pods, each pod maintains its own independent cache. If pod A detects an outage, pod B has no idea and will make the same redundant API calls and send duplicate Slack notifications.

This is why the deployment was locked to a single replica:

```yaml
# omega.yaml (before)
autoscale:
  minReplicas: 1
  maxReplicas: 1
```

### Why DynamoDB

We needed a shared persistence layer that:

- Supports concurrent reads/writes from multiple pods without corruption
- Has built-in TTL for automatic cleanup (replacing the 7-day file cleanup cron)
- Requires zero operational overhead (no database to manage)
- Works with existing EKS auth (IRSA)
- Is fast enough for the consumer's throughput (single-digit ms latency)

DynamoDB checks all of these. It's serverless, pay-per-request, and `boto3` picks up IRSA credentials automatically in Omega EKS.

---

## The Solution: Three Repositories, One Goal

The changes span three Git repositories that together form the full stack:

| Repository | Branch | Purpose |
|---|---|---|
| `yahoo.kafka_consumer` | `feature/dynamodb-scaling` | Consumer application code |
| `yahoo.kafka_consumer-k8s-eks-deploy` | `feature/dynamodb-scaling` | Omega EKS deployment configuration |
| `sre-terraform-kafka-consumer-eks` | `feature/dynamodb-tables` | AWS infrastructure (Terraform) |

### Deployment Order

1. **Terraform first** -- create DynamoDB tables + IAM policy
2. **Consumer code second** -- build new Docker image with DynamoDB backend support
3. **EKS deploy last** -- deploy 3 replicas using new image with DynamoDB config

---

## Repository 1: Consumer Application Code

### New File: `dynamodb_action_history.py`

Drop-in replacement for `FileBackedDict`. Exposes the same dict-like interface so no calling code needs to change.

```python
class DynamoDBActionHistory:
    def __init__(self, table_name: str, region: str, ttl_days: int):
        dynamodb = boto3.resource("dynamodb", region_name=region)
        self._table = dynamodb.Table(table_name)
        self._cache: dict[str, dict] = {}
        self._dirty_keys: set[str] = set()
```

Key design decisions:

- **`get()` always reads from DynamoDB** -- this is the authoritative cross-pod deduplication check. When pod B calls `check_alert_actioned()`, it hits DynamoDB to see if pod A already processed that alert.
- **`__setitem__` writes to DynamoDB immediately AND caches locally** -- ensures the write is durable before the consumer moves on.
- **`__getitem__` returns from local cache and marks the key dirty** -- this supports the existing pattern where callers mutate nested dict values (e.g., appending to `sd_failed_build_urls`), then call `sync()`.
- **`sync()` flushes dirty cache entries** -- called after all alert actions succeed to persist nested mutations.
- **TTL is set automatically** -- each item gets a `ttl` attribute set to `now + ttl_days`. DynamoDB's TTL feature deletes expired items automatically, replacing the old `find -mtime +7 -delete` cron in `app_start`.

### New File: `dynamodb_outage_cache.py`

Drop-in replacement for the in-memory `OutageCache`. Same public interface: `get_entry()`, `cache_detection()`, `check_terms_overlap()`, and the static `extract_query_terms()`.

```python
class DynamoDBOutageCache:
    def __init__(self, table_name: str, region: str):
        dynamodb = boto3.resource("dynamodb", region_name=region)
        self._table = dynamodb.Table(table_name)
```

When pod A detects an outage and caches it, pod B will see that cache entry via `get_entry()` and skip redundant API calls. The `ttl` attribute on each item handles expiration -- both at the application level (age check in `get_entry()`) and at the DynamoDB level (automatic deletion).

### Modified File: `action_runner.py`

The `ActionRunner.__init__()` method now selects the storage backend based on the `storage` section in the YAML config:

```python
storage_config = yaml_config.get('storage', {})
storage_backend = storage_config.get('backend', 'file')

if storage_backend == 'dynamodb':
    from yahoo.kafka_consumer.dynamodb_action_history import DynamoDBActionHistory
    from yahoo.kafka_consumer.dynamodb_outage_cache import DynamoDBOutageCache
    region = storage_config['region']
    self.action_history = DynamoDBActionHistory(
        table_name=storage_config['action_history_table'],
        region=region,
        ttl_days=storage_config['ttl_days'],
    )
    self.outage_cache = DynamoDBOutageCache(
        table_name=storage_config['outage_cache_table'],
        region=region,
    )
else:
    self.action_history = FileBackedDict(AUTOMATION_ACTION_HISTORY_FILE)
    self.outage_cache = OutageCache()
```

The DynamoDB imports are deferred (inside the `if` block) so `boto3` is not required when using the file backend -- keeping local development simple.

If no `storage` section is present in the config, it defaults to `file` -- the original behavior. This means **existing single-replica deployments continue to work without any config changes**.

### Modified File: `config_validator.py`

Added validation for the optional `storage` section. When `backend: dynamodb` is specified, all DynamoDB parameters are **required** -- no silent fallback defaults:

```python
if backend == "dynamodb":
    for field in ["region", "action_history_table", "outage_cache_table"]:
        if field not in storage_config:
            self.errors.append(f"storage.{field} is required when backend is 'dynamodb'")
        elif not isinstance(storage_config[field], str):
            self.errors.append(f"storage.{field} must be a string")

    if "ttl_days" not in storage_config:
        self.errors.append("storage.ttl_days is required when backend is 'dynamodb'")
    elif not isinstance(storage_config["ttl_days"], int) or storage_config["ttl_days"] <= 0:
        self.errors.append("storage.ttl_days must be a positive integer")
```

This ensures teams deploying with DynamoDB explicitly provide their own region, table names, and TTL -- no chance of accidentally connecting to someone else's tables.

### Modified File: `constants.py`

Removed all DynamoDB-related defaults. The only constant that remains from the storage system is `AUTOMATION_ACTION_HISTORY_FILE` for the file backend fallback.

### Modified File: `setup.cfg`

Added `boto3` to `install_requires`.

### Configuration Example

The `storage` section in the YAML config (added to `example-config.yaml`):

```yaml
# Storage backend for action history and outage cache (optional, defaults to "file")
# Use "file" for local dev / single-replica deployments (FileBackedDict on disk)
# Use "dynamodb" for multi-replica EKS deployments (shared state via DynamoDB)
# When backend is "dynamodb", all fields below are required.
storage:
  backend: dynamodb
  region: us-east-1
  action_history_table: kafka-consumer-action-history-prod
  outage_cache_table: kafka-consumer-outage-cache-prod
  ttl_days: 7
```

---

## Repository 2: EKS Deployment Configuration

### Modified File: `omega.yaml`

The core change -- scale from 1 to 3 replicas and remove PVC mounts:

```yaml
# Before
autoscale:
  minReplicas: 1
  maxReplicas: 1
pvcMounts:
  - name: action-history
    claimName: kafka-consumer-history-dev
    mountPath: /data/automation_consumer

# After
autoscale:
  minReplicas: 3
  maxReplicas: 3
# (pvcMounts section removed entirely)
```

Both the `component` (dev) and `deploy-production` (prod) jobs were updated identically.

### Modified File: `scripts/app_start`

Removed all file-based history logic:

- Removed `ACTION_HISTORY_FILE` environment variable export
- Removed the `find -mtime +7 -delete` cleanup block that pruned old history files from the PVC
- Added `STORAGE_BACKEND=dynamodb` environment variable export

### Modified Files: `config-production.yaml` and `config-development.yaml`

Added the `storage` section with environment-specific DynamoDB table names:

```yaml
# config-production.yaml
storage:
  backend: dynamodb
  region: us-east-1
  action_history_table: kafka-consumer-action-history-prod
  outage_cache_table: kafka-consumer-outage-cache-prod
  ttl_days: 7
```

```yaml
# config-development.yaml
storage:
  backend: dynamodb
  region: us-east-1
  action_history_table: kafka-consumer-action-history-dev
  outage_cache_table: kafka-consumer-outage-cache-dev
  ttl_days: 7
```

### Deleted Files: `pvc-prod.yaml` and `pvc-nonprod.yaml`

The PersistentVolumeClaim definitions for EFS are no longer needed. The PVCs should be cleaned up from the clusters manually before deploying.

---

## Repository 3: Terraform Infrastructure

### New File: `terraform/global-common/dynamodb.tf`

This file creates all the AWS resources needed for DynamoDB access. Since prod and nonprod are separate AWS accounts with separate `terraform apply` runs, the environment suffix is derived from the `var.environment` variable:

```hcl
locals {
  dynamodb_table_prefix = "kafka-consumer"
  dynamodb_env_suffix   = var.environment == "prod" ? "prod" : "dev"
}
```

#### DynamoDB Tables

Two tables per environment, both using on-demand billing (no capacity planning):

**`kafka-consumer-action-history-{env}`** -- stores processed alert records:
```hcl
resource "aws_dynamodb_table" "action_history" {
  name         = "${local.dynamodb_table_prefix}-action-history-${local.dynamodb_env_suffix}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "alert_id"

  attribute {
    name = "alert_id"
    type = "S"
  }

  ttl {
    attribute_name = "ttl"
    enabled        = true
  }
}
```

- **Partition key**: `alert_id` (string) -- the unique BigPanda alert identifier
- **TTL**: Enabled on the `ttl` attribute. DynamoDB automatically deletes items after the epoch timestamp in this field, replacing the old `find -mtime +7 -delete` cron job

**`kafka-consumer-outage-cache-{env}`** -- stores outage detection results:
```hcl
resource "aws_dynamodb_table" "outage_cache" {
  name         = "${local.dynamodb_table_prefix}-outage-cache-${local.dynamodb_env_suffix}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "action_name"

  attribute {
    name = "action_name"
    type = "S"
  }

  ttl {
    attribute_name = "ttl"
    enabled        = true
  }
}
```

- **Partition key**: `action_name` (string) -- the name of the `check_outage` action from the config
- **TTL**: Same mechanism, expiration controlled by the `outage_cache_ttl_seconds` in each action's config

#### IAM Policy for IRSA

The pods authenticate to DynamoDB via **IRSA** (IAM Roles for Service Accounts). The IRSA role is already created by the `tf_omega_eks_identity` module in `omega-app-iam.tf`. This file just attaches a policy granting access to the two DynamoDB tables:

```hcl
data "aws_iam_role" "kafka_consumer_irsa" {
  name = "${var.athenz_domain}.${var.services[0].name}"
}

resource "aws_iam_role_policy" "dynamodb_access" {
  name = "dynamodb-action-history-access"
  role = data.aws_iam_role.kafka_consumer_irsa.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:DescribeTable",
      ]
      Resource = [
        aws_dynamodb_table.action_history.arn,
        aws_dynamodb_table.outage_cache.arn,
      ]
    }]
  })
}
```

The policy follows least-privilege -- only the specific DynamoDB actions needed, scoped to only these two table ARNs. No `dynamodb:*` or wildcard resources.

At runtime, Kubernetes mounts a projected service account token inside each pod. `boto3` auto-discovers it via the `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` environment variables injected by the EKS mutating webhook. No credential management code needed.

#### Network / Egress

DynamoDB uses HTTPS on port 443. The existing Omega network policy (`tf-omega-app-network-policy` module) already allows egress on port 443 to any destination by default. No additional egress rules are required.

---

## How It All Comes Together

```
                        ┌─────────────────────────────────┐
                        │        Kafka (MSK)              │
                        │   Topic: sre (3+ partitions)    │
                        └──────────┬──────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
              ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
              │  Pod 1    │ │  Pod 2    │ │  Pod 3    │
              │  (replica)│ │  (replica)│ │  (replica)│
              └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                    │              │              │
                    │         IRSA (auto)         │
                    │              │              │
              ┌─────▼──────────────▼──────────────▼─────┐
              │              DynamoDB                    │
              │  ┌─────────────────────────────────┐    │
              │  │ action-history table             │    │
              │  │ PK: alert_id  |  TTL: 7 days    │    │
              │  └─────────────────────────────────┘    │
              │  ┌─────────────────────────────────┐    │
              │  │ outage-cache table               │    │
              │  │ PK: action_name  |  TTL: varies │    │
              │  └─────────────────────────────────┘    │
              └─────────────────────────────────────────┘
```

1. Kafka distributes partitions across 3 consumer pods (same consumer group)
2. Each pod processes its assigned alerts independently
3. Before acting on an alert, the pod calls `check_alert_actioned()` which reads from DynamoDB -- if any other pod already processed it, the alert is skipped
4. Before checking for outages, the pod reads the outage cache from DynamoDB -- if another pod already detected the same outage, redundant API calls and Slack notifications are avoided
5. After processing, the pod writes results to DynamoDB where all replicas can see them
6. DynamoDB TTL automatically cleans up old entries -- no cron jobs, no file cleanup scripts

### Backward Compatibility

- If `storage` is omitted from the config, the consumer uses `FileBackedDict` + in-memory `OutageCache` -- identical to the original behavior
- Local development doesn't require DynamoDB or `boto3` credentials
- The `FileBackedDict` and `OutageCache` classes are untouched -- they still work for single-replica or local setups
