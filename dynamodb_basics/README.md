# DynamoDB Serverless Database

**Services:** Amazon DynamoDB · IAM  
**Goal:** Design and configure a DynamoDB table with a thoughtful key schema, load sample data, and understand query patterns — the database layer in a serverless architecture.

---

## What I Built

Created a DynamoDB table designed around a realistic use case (user session tracking), chose the partition and sort key structure deliberately, and tested query patterns to understand how DynamoDB's access model differs fundamentally from relational databases.

---

## Table Design

**Use case:** User session tracking for a web application

```
Table Name: UserSessions

Primary Key:
  Partition Key (PK): userId       (String)
  Sort Key (SK):      sessionStart (String - ISO 8601 timestamp)

Attributes:
  - userId:        "user_001"
  - sessionStart:  "2026-01-15T09:30:00Z"
  - sessionEnd:    "2026-01-15T10:15:00Z"
  - ipAddress:     "192.168.1.100"
  - device:        "mobile"
  - status:        "expired"

Billing Mode: On-Demand (PAY_PER_REQUEST)
```

---

## Key Design Decisions

**Partition key selection**
Chose `userId` as the partition key because the most common access pattern is "get all sessions for a user." DynamoDB distributes data by partition key — a good partition key distributes load evenly and aligns with your primary read pattern.

**Sort key for range queries**
Added `sessionStart` as a sort key so sessions for a user can be queried in time order, or filtered by date range using `BETWEEN` conditions. Without a sort key, you can only fetch a single item — not all sessions for a user.

**On-demand billing over provisioned capacity**
Chose PAY_PER_REQUEST for unpredictable workloads. Provisioned capacity is more cost-efficient at steady, predictable load — but requires capacity planning. Picking the right billing model is a real operational decision.

**No joins by design**
DynamoDB is not a relational database. The schema is designed around access patterns, not normalized data structure. This is the most important mental shift when working with NoSQL.

---

## Sample Queries Tested

**Get all sessions for a user:**
```
PK = "user_001"
```

**Get sessions within a time window:**
```
PK = "user_001"
SK BETWEEN "2026-01-01T00:00:00Z" AND "2026-01-31T23:59:59Z"
```

---

## Troubleshooting Encountered

**Issue:** Query returned zero results even though items existed in the table.  
**Root cause:** Used `Scan` with a filter expression instead of `Query` — and had the partition key value wrong (wrong case).  
**Fix:** Switched to `Query` with the exact partition key value. DynamoDB is case-sensitive.  
**Lesson:** `Scan` reads every item in the table and then filters — it's expensive and slow at scale. Always prefer `Query` with a known partition key. Also: DynamoDB key values are case-sensitive; "User_001" ≠ "user_001".

---

## What I'd Do Differently in Production

- Define a **Global Secondary Index (GSI)** on `status` to support queries like "get all active sessions" without a full table scan
- Enable **DynamoDB Streams** to trigger Lambda functions on data changes (event-driven architecture)
- Set **TTL (Time to Live)** on session records to automatically expire old data and control table size
- Use **IAM resource-based conditions** to restrict which Lambda functions or services can read/write to this specific table
