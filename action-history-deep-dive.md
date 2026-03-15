# Action History Deep Dive: FileBackedDict vs DynamoDB

This document walks through exactly how the action history works, why `FileBackedDict` breaks with multiple pods, and how `DynamoDBActionHistory` fixes it -- using concrete examples with real data.

---

## What the Action History Actually Stores

Every time the consumer processes an alert, it writes a record to the action history keyed by the alert's unique ID. Here's what a real entry looks like:

```json
{
    "alert-abc-123": {
        "time_actioned": "2026-03-14 15:30:00",
        "last_action_executed": "oozie_parrot",
        "all_alert_actions_succeeded": true,
        "sd_failed_build_urls": []
    }
}
```

This record says: "Alert `alert-abc-123` was processed on March 14th. The last action that ran was `oozie_parrot`. All actions succeeded. No Screwdriver builds failed."

The purpose is **deduplication**. When the same alert comes through Kafka again (which happens often -- BigPanda sends updates on the same incident), the consumer checks the history first:

```python
# automation_consumer.py, line 262
if self.action_runner.check_alert_actioned(alert):
    logging.info("Alert %s has already been processed. Skipping...")
    return
```

Which calls:

```python
# action_runner.py, line 91-99
def check_alert_actioned(self, alert: dict) -> bool:
    alert_actioned = self.action_history.get(alert['id'], None)
    if alert_actioned:
        return True
    return False
```

If the alert ID is found in the history, skip it. If not, process it.

---

## How FileBackedDict Works (The Original)

`FileBackedDict` extends Python's built-in `dict`. It adds one behavior: **every mutation rewrites the entire file**.

```python
class FileBackedDict(dict):
    def __init__(self, filename):
        self.filename = filename
        if os.path.exists(filename):
            with open(filename, 'r') as file:
                data = json.load(file)
            super().__init__(data)       # Load file contents into memory
        else:
            super().__init__()           # Start with empty dict

    def __setitem__(self, key, value):
        super().__setitem__(key, value)  # Update the in-memory dict
        self._write_to_file()            # Rewrite the ENTIRE file

    def _write_to_file(self):
        with open(self.filename, 'w') as file:    # 'w' = truncate + write
            json.dump(self, file, indent=4)
```

### What "Truncate" Means

The `'w'` mode in `open(self.filename, 'w')` does two things **in order**:

1. **Immediately erases the entire file contents** (sets file size to 0 bytes)
2. Opens the file for writing from the beginning

This is important. The file is **empty** between step 1 and when `json.dump` finishes writing. If anything goes wrong during that window (crash, power loss, slow disk flush), the file is left empty or with partial content.

### Concrete Example: Single Pod (Works Fine)

Let's say there's one pod processing alerts. The history file starts empty:

```
history.json: {}
```

**Step 1**: Alert `alert-111` arrives. Pod processes it.

```python
self.action_history["alert-111"] = {
    "time_actioned": "2026-03-14 15:30:00",
    "last_action_executed": "oozie_parrot",
    "all_alert_actions_succeeded": True
}
```

What happens internally:
1. `super().__setitem__("alert-111", {...})` -- updates the in-memory dict
2. `self._write_to_file()` -- truncates history.json, writes the full dict

```
history.json: {"alert-111": {"time_actioned": "...", ...}}
```

**Step 2**: Alert `alert-222` arrives. Pod processes it.

```python
self.action_history["alert-222"] = {...}
```

1. In-memory dict now has both `alert-111` and `alert-222`
2. File is truncated, then **both** entries are written

```
history.json: {"alert-111": {...}, "alert-222": {...}}
```

**Step 3**: Alert `alert-111` arrives again (Kafka re-delivers it).

```python
check_alert_actioned(alert)  # calls self.action_history.get("alert-111")
# Returns the dict -> returns True -> alert is skipped
```

This works perfectly with one pod because the in-memory dict and the file are always in sync.

---

## Why It Breaks With Two Pods Sharing a File

Now imagine two pods reading and writing the **same** file on a shared EFS volume.

### Problem 1: Lost Writes

Starting state:
```
history.json: {}
```

**T=0**: Pod A starts up, reads the file:
```
Pod A memory: {}
Pod B memory: (not started yet)
File:         {}
```

**T=1**: Pod B starts up, reads the file:
```
Pod A memory: {}
Pod B memory: {}
File:         {}
```

**T=2**: Pod A processes `alert-111`, writes to file:
```
Pod A memory: {"alert-111": {...}}     <- updated
Pod B memory: {}                        <- STALE (doesn't know about alert-111)
File:         {"alert-111": {...}}      <- written by Pod A
```

**T=3**: Pod B processes `alert-222`, writes to file:
```
Pod A memory: {"alert-111": {...}}
Pod B memory: {"alert-222": {...}}      <- updated
File:         {"alert-222": {...}}      <- Pod B truncated and rewrote with ITS dict
```

**`alert-111` is gone from the file.** Pod B's in-memory dict only had `alert-222`, so when it called `_write_to_file()`, it truncated the file and wrote only `{"alert-222": {...}}`.

**T=4**: Pod A restarts (routine deployment, OOM, etc.). It reads the file:
```
Pod A memory: {"alert-222": {...}}      <- loaded from file, alert-111 is GONE
```

**T=5**: Alert `alert-111` comes through Kafka again. Pod A checks:
```python
self.action_history.get("alert-111")  # Returns None -- not in dict!
```

Pod A processes it **again**. The Screwdriver job runs again. Slack notification fires again. Duplicate work.

### Problem 2: Torn Writes (JSON Corruption)

Consider what happens at the filesystem level when `_write_to_file()` runs:

```python
def _write_to_file(self):
    with open(self.filename, 'w') as file:    # File is NOW EMPTY (0 bytes)
        json.dump(self, file, indent=4)        # Writes JSON byte by byte
```

The moment `open('w')` executes, the file is truncated to 0 bytes. Then `json.dump` writes the JSON content. But writing isn't instant -- it's a stream of bytes going to the filesystem.

If the file contains 1000 alert entries, the JSON might be 50KB. On EFS (a network filesystem), this write goes over the network. Now imagine:

**Scenario**: Pod A is writing a large JSON file. It has written 30KB of the 50KB. At this moment the pod crashes (OOM kill, node failure, etc.).

The file now contains:

```json
{
    "alert-111": {
        "time_actioned": "2026-03-14 15:30:00",
        "last_action_executed": "oozie_parrot",
        "all_alert_actions_succeeded": true
    },
    "alert-222": {
        "time_actioned": "2026-03-14 15:31:00",
        "last_action_execu
```

That's not valid JSON. The file is corrupted -- a "torn write."

### Problem 3: Silent Data Loss After Corruption

When any pod starts up and tries to read the corrupted file:

```python
def __init__(self, filename):
    if os.path.exists(filename):
        try:
            with open(filename, 'r') as file:
                data = json.load(file)            # FAILS: invalid JSON
        except json.JSONDecodeError:
            with open(filename, 'w') as file:
                json.dump({}, file)               # Overwrites with empty dict!
            super().__init__()                     # Starts with EMPTY history
```

The `JSONDecodeError` handler **replaces the corrupted file with an empty dictionary**. Every alert the consumer has ever processed is forgotten. The consumer will re-process everything.

### The Chain Reaction

These three problems feed into each other:

```
Multiple pods writing same file
        │
        ▼
  Lost writes (Problem 1)
        │
        ▼
  Stale data on pod restart
        │
        ▼
  Duplicate alert processing
```

```
Pod crash during file write
        │
        ▼
  Torn write / corrupt JSON (Problem 2)
        │
        ▼
  JSONDecodeError on next startup
        │
        ▼
  File overwritten with {} (Problem 3)
        │
        ▼
  ENTIRE history wiped
        │
        ▼
  Every alert gets re-processed
```

---

## How DynamoDBActionHistory Fixes This

`DynamoDBActionHistory` stores each alert as a **separate row** in a DynamoDB table. There's no single file to truncate. No shared in-memory state. Each operation is atomic at the item level.

### The DynamoDB Table Structure

Each row in the `kafka-consumer-action-history` table looks like:

| alert_id (PK) | time_actioned | last_action_executed | all_alert_actions_succeeded | ttl |
|---|---|---|---|---|
| alert-111 | 2026-03-14 15:30:00 | oozie_parrot | true | 1742212200 |
| alert-222 | 2026-03-14 15:31:00 | oozie_eagle | true | 1742212260 |
| alert-333 | 2026-03-14 15:32:00 | oozie_apollo | false | 1742212320 |

- `alert_id` is the partition key (primary key) -- each alert is its own independent row
- `ttl` is a Unix epoch timestamp 7 days in the future -- DynamoDB automatically deletes the row when the time passes

### Same Two-Pod Scenario, But With DynamoDB

**T=0**: Both pods start. No local state to load -- DynamoDB is the source of truth.

**T=2**: Pod A processes `alert-111`:
```python
self.action_history["alert-111"] = {
    "time_actioned": "...",
    "last_action_executed": "oozie_parrot",
    "all_alert_actions_succeeded": True
}
```
This calls `__setitem__` which does:
```python
def __setitem__(self, key, value):
    self._cache[key] = value          # Cache locally
    self._put_item(key, value)        # Write to DynamoDB immediately
```

DynamoDB table now has:
| alert_id | ... |
|---|---|
| alert-111 | ... |

**T=3**: Pod B processes `alert-222`:
```python
self.action_history["alert-222"] = {...}
```
Pod B writes `alert-222` as a **separate row**. It doesn't touch `alert-111` at all.

DynamoDB table now has:
| alert_id | ... |
|---|---|
| alert-111 | ... |
| alert-222 | ... |

Both entries are safe. No truncation. No overwriting.

**T=4**: Pod A restarts. Alert `alert-111` comes through Kafka again:
```python
def check_alert_actioned(self, alert):
    alert_actioned = self.action_history.get(alert['id'])
```

This calls `get()`:
```python
def get(self, key, default=None):
    if key in self._cache:
        return self._cache[key]                    # Check local cache first
    resp = self._table.get_item(Key={"alert_id": key})  # Then check DynamoDB
    item = resp.get("Item")
    if item is None:
        return default
    return item
```

Even though Pod A restarted and has an empty local cache, it queries DynamoDB directly and finds the `alert-111` row that it wrote before the restart. The alert is correctly skipped.

### Why Pod Crashes Are Safe

If Pod A crashes while writing to DynamoDB, one of two things happened:

1. **The PutItem succeeded before the crash** -- the row is in DynamoDB, safe and sound
2. **The PutItem didn't complete before the crash** -- the row is not in DynamoDB, and the alert will be re-processed on the next Kafka delivery (which is the correct behavior -- better to re-process than to silently lose tracking)

There's no partial/corrupt state. DynamoDB writes are atomic at the item level -- the row either exists completely or doesn't exist at all.

### The Local Cache and Dirty Keys

You might wonder: "If we always read from DynamoDB, why have a local cache at all?"

It's because of how the existing code mutates nested values. Look at this pattern in `action_runner.py`:

```python
# Step 1: Create the initial record
self.action_history[alert['id']] = {
    'time_actioned': '...',
    'last_action_executed': '',
    'all_alert_actions_succeeded': False
}

# Step 2: Run each action, updating the record in-place
self.action_history[alert['id']]['last_action_executed'] = action_def['name']

# Step 3: Mark success
self.action_history[alert['id']]['all_alert_actions_succeeded'] = True

# Step 4: Flush to DynamoDB
self.action_history.sync()
```

Step 1 uses `__setitem__` -- writes to DynamoDB immediately. Good.

Step 2 uses `__getitem__` to get the dict, then mutates a nested value. Python has no way to detect that `['last_action_executed']` was changed on the returned dict. The local cache solves this:

```python
def __getitem__(self, key):
    if key in self._cache:
        self._dirty_keys.add(key)    # Mark as "might have been modified"
        return self._cache[key]      # Return the mutable dict reference
    item = self.get(key)
    self._cache[key] = item
    self._dirty_keys.add(key)
    return item
```

The caller gets back a reference to the cached dict. Any mutations they make (like `['last_action_executed'] = 'oozie_parrot'`) modify the cached copy. The key is added to `_dirty_keys`.

Step 4 calls `sync()`, which flushes all dirty entries to DynamoDB:

```python
def sync(self):
    for key in list(self._dirty_keys):
        if key in self._cache:
            self._put_item(key, self._cache[key])  # Write the mutated dict to DynamoDB
    self._dirty_keys.clear()
```

This batches the nested mutations into a single DynamoDB write at the end instead of writing after every `['field'] = value` assignment.

### Automatic Cleanup via TTL

Every item written to DynamoDB includes a `ttl` field:

```python
def _ttl_epoch(self) -> int:
    return int(time.time()) + (self._ttl_days * 86400)  # now + 7 days in seconds

def _put_item(self, key, value):
    item = {"alert_id": key, "ttl": self._ttl_epoch()}
    # ... add all the value fields ...
    self._table.put_item(Item=item)
```

If `ttl_days` is 7, an item written on March 14th gets `ttl = 1742212200` (March 21st as a Unix epoch). After March 21st, DynamoDB's background process automatically deletes the row. No cron jobs, no cleanup scripts, no file management.

This replaces the old `app_start` script logic:

```bash
# BEFORE (in app_start, now removed):
find "$HISTORY_DIR" -name "history_*.json" -type f -mtime +7 -delete
```

---

## Side-by-Side Comparison

| Aspect | FileBackedDict | DynamoDBActionHistory |
|---|---|---|
| **Storage** | Single JSON file on disk/EFS | One row per alert in DynamoDB |
| **Write operation** | Truncate file, rewrite ALL entries | Write only the one affected row |
| **Concurrent writes** | Last writer wins, others are lost | Each row is independent, no conflicts |
| **Crash during write** | File left with partial JSON (corrupt) | DynamoDB write is atomic -- row exists or doesn't |
| **Corrupt data handling** | Wipes entire history to `{}` | N/A -- corruption can't happen |
| **Cross-pod reads** | Must re-read file (stale in memory) | `get()` queries DynamoDB directly |
| **Cleanup** | Cron job (`find -mtime +7 -delete`) | DynamoDB TTL (automatic, zero maintenance) |
| **Scaling** | Unsafe beyond 1 replica | Safe with any number of replicas |
