# jq Masterclass
## Essential JSON Query Tool for CKS Forensics & Auditing

---

# What is jq?

**jq** = "JSON query language" - slice, filter, transform JSON data.

**Why for CKS**: Audit logs are JSON. API responses are JSON. You need jq to parse them quickly.

**Real exam use**: 
```bash
# Without jq (useless)
cat audit.log | head -1
# Outputs: giant unreadable JSON blob

# With jq (instant insights)
cat audit.log | jq '.user.username'
# Outputs: "alice" (just the username)
```

---

# jq Basics (Essential Syntax)

## 1. Pretty-Print JSON (Human Readable)

```bash
echo '{"name":"pod","status":"running"}' | jq .
# Output:
# {
#   "name": "pod",
#   "status": "running"
# }
```

## 2. Select Fields (Access Nested Values)

```bash
# Get top-level field
jq '.name' < data.json

# Get nested field
jq '.metadata.name' < pod.json

# Get deeply nested
jq '.spec.containers[0].name' < pod.json
```

## 3. Array Access

```bash
# Get first element
jq '.[0]' array.json

# Get all elements (iteration)
jq '.[]' array.json

# Get specific index
jq '.[2]' array.json
```

## 4. Pipe Operator (Chain Operations)

```bash
# Get 'name' from each object
jq '.[] | .name' data.json

# Get username from audit logs
jq '.[] | .user.username' audit.json
```

---

# jq Filtering (select)

## 5. Filter by Condition

```bash
# Select errors (status >= 400)
jq '.[] | select(.responseStatus.code >= 400)' audit.json

# Select specific user
jq '.[] | select(.user.username == "alice")' audit.json

# Select Secret operations
jq '.[] | select(.objectRef.resource == "secrets")' audit.json

# Multiple conditions (AND)
jq '.[] | select(.verb == "delete" and .objectRef.resource == "pods")' audit.json
```

## 6. String Matching

```bash
# Starts with
jq '.[] | select(.objectRef.name | startswith("test-"))' audit.json

# Contains
jq '.[] | select(.user.username | contains("admin"))' audit.json

# NOT equal
jq '.[] | select(.user.username != "system")' audit.json
```

---

# jq Transformation

## 7. Transform/Restructure Data

```bash
# Create custom object
jq '.[] | {user: .user.username, action: .verb, time: .requestReceivedTimestamp}' audit.json

# Output:
# {
#   "user": "alice",
#   "action": "create",
#   "time": "2026-09-26T10:15:30Z"
# }
```

## 8. Array Operations

```bash
# Count elements
jq 'length' data.json

# Get unique values
jq '[.[] | .user.username] | unique' audit.json

# Sort by field
jq 'sort_by(.requestReceivedTimestamp)' audit.json

# Group by field
jq 'group_by(.user.username)' audit.json
```

---

# jq Output Formatting

## 9. Raw Output (No JSON Quotes)

```bash
# Default (with quotes)
jq '.user.username' audit.json
# Output: "alice"

# Raw (no quotes) - best for CLI
jq -r '.user.username' audit.json
# Output: alice
```

## 10. CSV Export

```bash
# Export as CSV
jq -r '[.user.username, .verb, .objectRef.resource] | @csv' audit.json
# Output:
# "alice","create","pods"
# "bob","delete","secrets"
```

---

# Real CKS Audit Logging Examples

## Example 1: Find All Secret Creations

```bash
cat audit.log | jq '.[] | select(.objectRef.resource=="secrets" and .verb=="create")'
```

## Example 2: Find Failed Authorizations

```bash
cat audit.log | jq '.[] | select(.responseStatus.code == 403 or .responseStatus.code == 401) | {time: .requestReceivedTimestamp, user: .user.username, status: .responseStatus.code}'
```

## Example 3: Find Pod Exec (Shell Access)

```bash
cat audit.log | jq '.[] | select(.objectRef.resource=="pods/exec") | {time: .requestReceivedTimestamp, user: .user.username, pod: .objectRef.name}'
```

## Example 4: Count Actions by User

```bash
cat audit.log | jq -s 'group_by(.user.username) | map({user: .[0].user.username, count: length}) | sort_by(-.count)'

# Output:
# [
#   {"user": "alice", "count": 245},
#   {"user": "bob", "count": 87}
# ]
```

---

# jq Cheat Sheet

```bash
# BASICS
jq '.' file.json                              # Pretty print
jq '.field' file.json                         # Get field
jq '.field.nested' file.json                  # Nested field
jq '.[]' file.json                            # Iterate all

# FILTERING
jq '.[] | select(.status=="failed")' file.json
jq '.[] | select(.count > 10)' file.json

# TRANSFORMATION
jq '.[] | {name: .name, status: .status}' file.json
jq '[.[] | .name] | unique' file.json

# OUTPUT
jq -r '.[]' file.json                         # Raw (no quotes)
jq -r '[.name, .status] | @csv' file.json     # CSV export

# COMBINING
jq -s 'group_by(.user)' file.json             # Group all
jq 'sort_by(.timestamp)' file.json            # Sort
```

---

# Pre-Exam Speed Targets

- **Pretty-print JSON**: <5 sec
- **Extract single field**: <5 sec
- **Filter by condition**: <10 sec
- **Transform + export CSV**: <20 sec

