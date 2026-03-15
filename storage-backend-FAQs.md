# Storage Backend FAQ

Common questions about the `file` vs `dynamodb` storage backends and when each one applies.

---

## Is FileBackedDict broken? Does it cause corruption everywhere?

**No.** `FileBackedDict` works perfectly fine when a single consumer process writes to its own file. The corruption problems only occur when **multiple processes write to the same file concurrently**.

Here's a breakdown by deployment scenario:

| Scenario | Corruption Risk | Deduplication |
|---|---|---|
| 1 consumer, 1 server, 1 file | None | Works (single process sees its own history) |
| 2 consumers, 1 server, shared file | **Yes** -- lost writes, torn writes, silent data loss | Broken (file corruption wipes history) |
| 3 pods in EKS, shared file via EFS | **Yes** -- same problems, amplified by network filesystem latency | Broken |
| 3 consumers, 3 servers, each with own file | None | **Partial** -- each consumer only sees its own history, not what the others processed |

The last scenario is important: even if you avoid corruption by giving each consumer its own file, you lose cross-consumer deduplication. Consumer A doesn't know that Consumer B already processed an alert during a previous Kafka partition assignment. The alert gets processed twice.

**Bottom line**: `FileBackedDict` isn't inherently broken. It's a perfectly good solution for single-consumer deployments. The problem is that it can't safely share state across multiple processes -- and sharing state is a requirement for multi-consumer scaling.

---

## Why DynamoDB? Isn't that complex for teams not running in Omega EKS?

The DynamoDB backend was built specifically for our Omega EKS deployment, where:

- IRSA (IAM Roles for Service Accounts) handles authentication automatically
- The Terraform infrastructure repo creates the tables and IAM policies
- `boto3` picks up credentials with zero configuration in application code
- DynamoDB is serverless, so there's nothing to manage operationally

For teams **not** running in Omega EKS, DynamoDB does add setup overhead:

- Creating DynamoDB tables in their AWS account
- Setting up IAM credentials (no IRSA available outside EKS)
- Ensuring network access to AWS from their infrastructure
- Managing credential rotation

**However, this doesn't affect them unless they choose to scale.** The storage backend defaults to `file` when no `storage` section is present in the config. A team running a single consumer on a server changes nothing -- they get the original `FileBackedDict` behavior with no additional dependencies.

### What if a team wants to scale but isn't on EKS?

If they're on AWS at all (e.g., EC2 instances), `boto3` supports multiple credential sources beyond IRSA:

- **EC2 instance profiles** -- attach an IAM role to the instance, `boto3` picks it up automatically
- **Environment variables** -- set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
- **Credentials file** -- standard `~/.aws/credentials` file
- **SSO** -- AWS SSO integration

So a team on EC2 with an instance profile would have DynamoDB access with zero code changes.

If they're not on AWS at all, they'd need to set up AWS credentials manually, which is admittedly more friction. In that case, a future enhancement could add a third backend option (e.g., Redis) that might better fit non-AWS environments.

---

## Do we need to pick a backend for every team?

No. The architecture is designed so that **each team configures their own backend** in their YAML config file:

```yaml
# Option 1: File backend (default, no config needed)
# Just don't include a storage section at all.

# Option 2: DynamoDB backend (for multi-replica EKS deployments)
storage:
  backend: dynamodb
  region: us-east-1
  action_history_table: my-team-action-history
  outage_cache_table: my-team-outage-cache
  ttl_days: 7
```

When `backend: dynamodb` is specified, the config validator enforces that `region`, `action_history_table`, `outage_cache_table`, and `ttl_days` are all provided. There are no hidden defaults -- each team explicitly specifies their own table names and region, so there's no risk of accidentally connecting to someone else's infrastructure.

---

## Summary

| Question | Answer |
|---|---|
| Does FileBackedDict corrupt data on its own? | No, only when multiple processes share the same file |
| Can I run one consumer with FileBackedDict? | Yes, works perfectly, no changes needed |
| Do I need DynamoDB to run this consumer? | No, only if you want multi-replica scaling |
| Does DynamoDB auth work outside EKS? | Yes, `boto3` supports EC2 instance profiles, env vars, credential files, SSO |
| What if a team isn't on AWS at all? | They can run a single consumer with the file backend; multi-consumer scaling without AWS would need a future backend (e.g., Redis) |
| Are there hardcoded table names or regions? | No, everything is supplied via the YAML config |
