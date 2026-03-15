# Testing the DynamoDB Code Locally

## The Problem

We added two new DynamoDB-backed classes (`DynamoDBActionHistory` and `DynamoDBOutageCache`) to support multi-replica scaling. The normal deployment cycle looks like:

1. Push code to GitHub
2. Screwdriver builds a new Docker image
3. Omega EKS picks up the new image and deploys it
4. Hope it works

That's a slow feedback loop for verifying new code, especially code that talks to an AWS service. We needed a way to test the DynamoDB integration locally without any of that.

## The Solution: `moto`

[moto](https://github.com/getmoto/moto) is a Python library that mocks AWS services entirely in-process. When you wrap code with `moto`'s `mock_aws()` context manager, all `boto3` calls get intercepted and handled by an in-memory fake that implements the full DynamoDB API.

Our `DynamoDBActionHistory` and `DynamoDBOutageCache` classes have no idea they're not talking to real DynamoDB. Same code paths, same `boto3` calls, same behavior -- just no network, no credentials, no AWS account needed.

### Why `moto` over alternatives

| Approach | Pros | Cons |
|---|---|---|
| **`moto` (what we use)** | No setup, runs in-process, fast, pytest-native | Doesn't catch region/endpoint quirks |
| **DynamoDB Local (Docker)** | Closer to real DynamoDB | Requires Docker, slower, more setup |
| **Real DynamoDB dev tables** | Truly real | Needs AWS creds + Terraform applied first |
| **Manual testing in Omega** | Tests the full stack | Slow deploy cycle, hard to debug |

`moto` covers 95% of what we need. If something passes with `moto` and fails in production, it's almost certainly a permissions or networking issue -- not a code bug.

## Test Setup

### Dependencies

`moto[dynamodb]` and `pytest` are listed as test extras in `setup.cfg`:

```ini
[options.extras_require]
test =
    moto[dynamodb]
    pytest
```

Install with:

```bash
pip install -e ".[test]"
```

### How the Fixtures Work

Each test file follows the same pattern:

1. **Fake AWS credentials** -- `moto` requires environment variables to be set, but any values work:

```python
@pytest.fixture(scope="function", autouse=True)
def aws_credentials():
    os.environ["AWS_ACCESS_KEY_ID"] = "testing"
    os.environ["AWS_SECRET_ACCESS_KEY"] = "testing"
    os.environ["AWS_SECURITY_TOKEN"] = "testing"
    os.environ["AWS_SESSION_TOKEN"] = "testing"
    os.environ["AWS_DEFAULT_REGION"] = "us-east-1"
```

2. **Create a mock DynamoDB table** -- Uses `mock_aws()` as a context manager. The table is created with the same schema as our Terraform definition, and torn down automatically after each test:

```python
@pytest.fixture(scope="function")
def dynamodb_table(aws_credentials):
    with mock_aws():
        dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
        dynamodb.create_table(
            TableName="test-action-history",
            KeySchema=[{"AttributeName": "alert_id", "KeyType": "HASH"}],
            AttributeDefinitions=[{"AttributeName": "alert_id", "AttributeType": "S"}],
            BillingMode="PAY_PER_REQUEST",
        )
        yield dynamodb.Table("test-action-history")
```

3. **Instantiate the class under test** -- Uses the mocked table, so all `boto3` calls inside the class hit the in-memory fake:

```python
@pytest.fixture(scope="function")
def history(dynamodb_table):
    return DynamoDBActionHistory(
        table_name="test-action-history", region="us-east-1", ttl_days=7
    )
```

Every test function gets a fresh table and a fresh instance. No state leaks between tests.

## What's Tested

### `test_dynamodb_action_history.py` (12 tests)

Tests the `DynamoDBActionHistory` class, which is the DynamoDB-backed replacement for `FileBackedDict`.

| Test Class | What It Covers |
|---|---|
| **TestSetAndGet** | `get()` returns `None`/default for missing keys, stores and retrieves values, strips internal metadata (`alert_id`, `ttl`) from results, overwrites existing keys |
| **TestContains** | `in` operator works for both missing and present keys |
| **TestGetItem** | `__getitem__` raises `KeyError` for missing keys, returns mutable references so callers can update nested dicts in-place (this is how `action_runner.py` updates build status) |
| **TestSync** | `sync()` flushes locally-modified entries back to DynamoDB (verified by reading the raw table), clears the dirty key set after flush |
| **TestSdFailedBuildUrls** | `sd_failed_build_urls` roundtrips correctly between Python lists and DynamoDB sets |
| **TestTTL** | TTL epoch is computed correctly and written to the table (verified against `time.time() + ttl_days * 86400`) |

The **TestGetItem + TestSync** combination is especially important because it mirrors the real usage pattern in `action_runner.py`:

```python
entry = self.action_history[alert_id]    # __getitem__ -> mutable ref
entry["status"] = "success"              # mutate locally
entry["sd_failed_build_urls"].append(url)
self.action_history.sync()               # flush to DynamoDB
```

### `test_dynamodb_outage_cache.py` (14 tests)

Tests the `DynamoDBOutageCache` class, which replaces the in-memory `OutageCache`.

| Test Class | What It Covers |
|---|---|
| **TestCacheDetectionAndGetEntry** | Returns `None` for empty cache, caches and retrieves outage data, expired entries return `None` (uses `unittest.mock.patch` to fake the clock), valid entries return correct structure |
| **TestCheckTermsOverlap** | No overlap when cache is empty, partial overlap returns correct terms, no overlap returns empty set, full overlap detected correctly |
| **TestExtractQueryTerms** | Extracts `host` from tags and fallback fields, extracts `description` from tags, extracts custom tags like `oozie_pipeline`, handles empty alert lists |
| **TestDynamoDBTableContent** | Verifies raw DynamoDB item structure (correct hash key, TTL, query_key, terms) |

### `test_config_validator_storage.py` (14 tests)

Tests the `_validate_storage_section()` method of `ConfigValidator`.

| Test | What It Verifies |
|---|---|
| No storage section | No errors (storage is optional) |
| `backend: file` | Accepted without errors |
| `backend: dynamodb` with all fields | Accepted without errors |
| `backend: redis` | Rejected ("must be 'file' or 'dynamodb'") |
| Missing `region` | Error: "storage.region is required" |
| Missing `action_history_table` | Error: "storage.action_history_table is required" |
| Missing `outage_cache_table` | Error: "storage.outage_cache_table is required" |
| Missing `ttl_days` | Error: "storage.ttl_days is required" |
| `ttl_days` not an int | Error: "must be a positive integer" |
| `ttl_days` zero or negative | Error: "must be a positive integer" |
| `region` not a string | Error: "storage.region must be a string" |
| Storage not a dict | Error: "storage must be a dictionary" |
| Missing all 4 required fields | Exactly 4 errors |

## Running the Tests

```bash
# All tests (including the original import test)
python -m pytest tests/ -v

# Just the DynamoDB tests
python -m pytest tests/test_dynamodb_action_history.py tests/test_dynamodb_outage_cache.py -v

# Just the config validator tests
python -m pytest tests/test_config_validator_storage.py -v

# Run with short output
python -m pytest tests/ -q
```

### Expected Output

```
tests/test_config_validator_storage.py   14 passed
tests/test_dynamodb_action_history.py    12 passed
tests/test_dynamodb_outage_cache.py      14 passed
tests/test_import.py                      1 passed

41 passed in ~3s
```

## What This Doesn't Test

These tests verify that our code correctly interacts with the DynamoDB API. They do **not** test:

- **IAM permissions** -- `moto` doesn't enforce IAM policies. If the IRSA role is missing `dynamodb:PutItem`, the tests will still pass but production will fail. This is validated by Terraform and verified after deployment.
- **Network connectivity** -- Omega's MSD policies allowing egress to DynamoDB on port 443. Verified in the Terraform network policy configuration.
- **Table existence** -- The tests create tables in fixtures. In production, tables are created by Terraform. If someone deletes a table, the tests won't catch it.
- **Concurrent multi-pod behavior** -- `moto` runs in a single process. The DynamoDB consistency guarantees (atomic `PutItem`, conditional writes) are what makes multi-replica safe, and those are AWS's problem, not ours.

For those concerns, the deployment to the non-prod Omega cluster serves as the integration test.

## Adding New Tests

If you modify `DynamoDBActionHistory` or `DynamoDBOutageCache`, follow the existing pattern:

1. Use the `history` or `cache` fixture to get a class instance connected to a mocked table
2. Call methods on the instance
3. Assert on return values, or read the raw `dynamodb_table` fixture to verify what was actually written

For testing expiry/TTL behavior, use `unittest.mock.patch` to control `time.time()`:

```python
from unittest.mock import patch

def test_expired_entry(self, cache):
    past = int(time.time()) - 1000
    with patch("yahoo.kafka_consumer.dynamodb_outage_cache.time") as mock_time:
        mock_time.time.return_value = past
        cache.cache_detection("action", {"x"}, "host", ttl_seconds=60)

    # Now reading without the mock, real time has advanced past the TTL
    assert cache.get_entry("action") is None
```
