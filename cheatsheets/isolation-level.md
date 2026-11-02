# Isolation Level Cheat Sheet

A companion to [Chapter 7: Transactions](../ddia/07-transactions.md), with anomaly tables and example transaction timelines.

Database transactions can interleave reads and writes, so applications need guarantees about what each transaction can observe.
An isolation level defines those guarantees.
Locks and multi-version concurrency control, or MVCC, are implementation techniques that a database can combine.
The sections below describe common models; exact behavior depends on the database and its settings.
Each section also notes which PostgreSQL isolation level, if any, uses that model.

- Statement: a single SQL operation, such as `SELECT` or `UPDATE`.
- Transaction: one or more statements committed or rolled back together.

| Anomaly | Meaning |
| --- | --- |
| Dirty read | Reading another transaction's uncommitted changes. |
| Nonrepeatable read | Reading the same row again and seeing a different committed value. |
| Phantom read | Repeating a query and seeing a changed set of matching rows because another transaction committed changes. |
| Read skew | Reading related values from different moments and seeing an inconsistent combination. |
| Lost update | Writing a value based on a stale read and overwriting another transaction's update. |
| Write skew | Transactions update different rows after reading a shared condition, and together violate that condition. |

The category tables assume each task keeps its related reads and writes in one transaction, and all concurrent transactions use the stated isolation model.
**Possible** means the model permits the anomaly without additional protection.
**Prevented** means the model rules it out, aborting conflicting transactions when necessary.
Each **Possible** entry links to an example that states which isolation models it applies to.

The two read-committed models permit the same listed anomalies, although their blocking and within-statement read behavior differ.
Blocking, [deadlocks](#deadlocks), and transaction aborts can still occur even when these data anomalies are prevented.

The timelines below show example schedules, with time moving downward.
Each category starts with a normal flow, followed by examples of possible anomalies or conflict recovery.
The doctor examples use the rule that at least one doctor must remain on call.
Abort timing and the choice of transaction to abort depend on the database.

## Read Committed with Locks

| Dirty read | Nonrepeatable read | Phantom read | Read skew | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| Prevented | [Possible](#possible-anomaly-nonrepeatable-read) | [Possible](#possible-anomaly-phantom-read) | [Possible](#possible-anomaly-read-skew) | [Possible](#possible-anomaly-lost-update) | [Possible](#possible-anomaly-write-skew) |

- Reads exclude uncommitted data, commonly by acquiring shared locks that conflict with writes.
- Read locks can be released before the transaction ends, so later reads may see changed values or new matching rows.
- Separate read-then-write operations can lose updates unless the application protects the entire operation.

**PostgreSQL:** not offered.
Its `READ COMMITTED` level uses [Read Committed with MVCC](#read-committed-with-mvcc) instead.

### Normal Flow

The reader waits for an uncommitted write, then reads the committed value.

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially value = 100
    W->>DB: BEGIN
    W->>DB: Write value = 120
    Note over W,DB: Value 120 is uncommitted<br/>Writer holds a write lock
    R->>DB: BEGIN at Read Committed
    R->>DB: Read value
    Note over DB,R: Read waits for the writer's lock
    W->>DB: COMMIT
    Note over DB: Value 120 is committed<br/>Writer's lock is released
    DB-->>R: Return 120
    R->>DB: COMMIT
```

### Possible Anomaly: Lost Update

This example applies to both read-committed implementations without additional protection for the read-then-write operation.
Both transactions read the same counter, then write a replacement value calculated from that earlier read.
The writes are not atomic increments computed from the current database value.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    Note over A,B: Initially counter = 42<br/>Each transaction intends to add one
    A->>DB: BEGIN at Read Committed
    B->>DB: BEGIN at Read Committed
    A->>DB: Read counter
    DB-->>A: Return 42
    B->>DB: Read counter
    DB-->>B: Return 42
    Note over A,B: Both calculate replacement value 43<br/>Neither protects the full read-then-write operation
    A->>DB: Write stored value 43
    A->>DB: COMMIT
    B->>DB: Write stored value 43
    B->>DB: COMMIT
    Note over DB: Final counter = 43<br/>One increment was lost
```

Protect read-then-write logic with a SQL update based on the current row value, an appropriate lock held across the read and write, or an update conditional on an expected version.
Handle a failed version check by retrying or reporting the conflict.

## Read Committed with MVCC

| Dirty read | Nonrepeatable read | Phantom read | Read skew | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| Prevented | [Possible](#possible-anomaly-nonrepeatable-read) | [Possible](#possible-anomaly-phantom-read) | [Possible](#possible-anomaly-read-skew) | [Possible](#possible-anomaly-lost-update) | [Possible](#possible-anomaly-write-skew) |

- Each statement reads a snapshot of data committed when that statement begins, plus its transaction's own changes.
- Ordinary snapshot reads avoid blocking row updates, but conflicting writers can still block each other.
- Later statements can see newer commits, so nonrepeatable reads, phantoms, and read skew remain possible.
- Separate read-then-write operations can still lose updates.

**PostgreSQL:** `READ COMMITTED`, the default level, uses this model.

- `READ UNCOMMITTED` is accepted but behaves like `READ COMMITTED`, so dirty reads never happen.
- When an `UPDATE` or `DELETE` reaches a row that another transaction is changing, it waits for that transaction to finish.
  If the other transaction commits, PostgreSQL re-checks the `WHERE` condition against the new row version and changes the row only if it still matches.
- A single `UPDATE ... SET hits = hits + 1` is therefore safe, but reading a value in one statement and writing it back in another can still lose an update.

### Normal Flow

The reader uses the last committed version while the writer's change is still uncommitted.

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially value = 100
    W->>DB: BEGIN
    W->>DB: Write value = 120
    Note over W,DB: Value 120 is uncommitted
    R->>DB: BEGIN at Read Committed
    R->>DB: Read value
    DB-->>R: Return committed value 100 without waiting
    R->>DB: COMMIT
    W->>DB: COMMIT
    Note over W,R: Both transactions succeed<br/>The reader finishes before the writer commits
```

### Possible Anomaly: Nonrepeatable Read

This example applies to both read-committed implementations when the writer commits between the reader's statements.
A second statement in the same transaction can see a newer committed value.

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially value = 100
    R->>DB: BEGIN at Read Committed
    R->>DB: Read value
    DB-->>R: Return 100
    W->>DB: BEGIN at Read Committed
    W->>DB: Write value = 120
    W->>DB: COMMIT
    R->>DB: Read value again in the same transaction
    DB-->>R: Return the newly committed value 120
    R->>DB: COMMIT
```

### Possible Anomaly: Phantom Read

This example applies to both read-committed implementations when another transaction inserts a matching row and commits between the reader's queries.
The same query returns a different set of rows.

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially only order 1 has status open
    R->>DB: BEGIN at Read Committed
    R->>DB: Find orders with status open
    DB-->>R: Return order 1
    W->>DB: BEGIN at Read Committed
    W->>DB: Insert order 2 with status open
    W->>DB: COMMIT
    R->>DB: Repeat the same query in the same transaction
    DB-->>R: Return orders 1 and 2
    Note over R: An extra matching row appears<br/>This is a phantom read
    R->>DB: COMMIT
```

### Possible Anomaly: Read Skew

This example applies to both read-committed implementations when related values are read in separate statements.
A transfer preserves the total balance, but the reader combines a balance from before the transfer with one from afterward.

```mermaid
sequenceDiagram
    autonumber
    participant W as Transfer transaction
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially account A = 100 and account B = 100<br/>The total balance is 200
    R->>DB: BEGIN at Read Committed
    R->>DB: Read account A
    DB-->>R: Return 100
    W->>DB: BEGIN at Read Committed
    W->>DB: Set account A = 90
    W->>DB: Set account B = 110
    W->>DB: COMMIT both changes together
    R->>DB: Read account B in a separate statement
    DB-->>R: Return 110
    Note over R: Observed total = 100 + 110 = 210<br/>The committed total remained 200
    R->>DB: COMMIT
```

## Snapshot Isolation with MVCC

| Dirty read | Nonrepeatable read | Phantom read | Read skew | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| Prevented | Prevented | Prevented | Prevented | Prevented | [Possible](#possible-anomaly-write-skew) |

- Reads use a stable transaction snapshot, plus the transaction's own changes.
- Other transactions' commits do not change the values or matching rows seen by repeated snapshot reads.
- Conflicting concurrent writes to the same row cannot both commit under snapshot isolation.
- Write skew can still occur when transactions update different rows.

**PostgreSQL:** `REPEATABLE READ` uses this model.

- It also prevents phantom reads, which the SQL standard allows at this level.
- Changing a row that a concurrent transaction already changed and committed fails with a "could not serialize access" error, SQLSTATE `40001`.
  Roll back and rerun the whole transaction.

### Normal Flow

The reader keeps the same snapshot while another transaction commits a newer value.

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as Database
    participant R as Reader
    Note over W,R: Initially value = 100
    R->>DB: BEGIN with Snapshot Isolation
    R->>DB: Read value and establish snapshot
    DB-->>R: Return 100
    W->>DB: BEGIN with Snapshot Isolation
    W->>DB: Write value = 120
    W->>DB: COMMIT
    R->>DB: Read value again in the same transaction
    DB-->>R: Return 100 from the original snapshot
    R->>DB: COMMIT
    Note over W,R: Both transactions succeed<br/>The reader sees a stable view
```

### Possible Anomaly: Write Skew

This example applies to both read-committed implementations and snapshot isolation when the on-call rule is checked using ordinary reads without additional protection.
Both transactions read that two doctors are on call before either commits.
Each then uses that earlier observation to take a different doctor off call.
Both commits can succeed, leaving nobody on call.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A (Alice)
    participant DB as Database
    participant B as Transaction B (Bob)
    Note over A,B: Initially Alice and Bob are on call<br/>At least one must remain
    A->>DB: BEGIN at the selected isolation level
    B->>DB: BEGIN at the selected isolation level
    A->>DB: Read on-call doctors
    DB-->>A: Alice and Bob are on call
    B->>DB: Read on-call doctors
    DB-->>B: Alice and Bob are on call
    A->>DB: Set Alice off call
    A->>DB: COMMIT
    B->>DB: Set Bob off call
    B->>DB: COMMIT
    Note over A,B: No same-row write conflict<br/>Both commits succeed and nobody remains on call
```

## Serializable Isolation with Locks

| Dirty read | Nonrepeatable read | Phantom read | Read skew | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| Prevented | Prevented | Prevented | Prevented | Prevented | Prevented |

- Committed results must match some order of running the transactions one at a time.
- Transactions can execute concurrently when their locks are compatible.
- Two-phase locking can enforce serializability, with range or predicate protection where needed to prevent phantoms.
- Conflicting operations may block, and deadlocks may require aborting and retrying a transaction.

**PostgreSQL:** not offered.
Its `SERIALIZABLE` level uses [Serializable Snapshot Isolation](#serializable-snapshot-isolation) instead.

### Normal Flow

In this example, the transactions access separate counter rows and their locks are compatible.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    Note over A,B: Initially counter A = 10 and counter B = 20<br/>Each transaction updates only its own counter
    A->>DB: BEGIN at Serializable
    B->>DB: BEGIN at Serializable
    A->>DB: Read counter A with a shared lock
    DB-->>A: Return 10
    B->>DB: Read counter B with a shared lock
    DB-->>B: Return 20
    A->>DB: Write counter A = 11 with an exclusive lock
    B->>DB: Write counter B = 21 with an exclusive lock
    A->>DB: COMMIT and release locks
    B->>DB: COMMIT and release locks
    Note over A,B: Both transactions overlap and commit<br/>No conflicting locks in this schedule
```

### Conflict and Recovery: Blocking

This is the doctor scenario that causes write skew under the weaker models.
Locks prevent it: a transaction cannot write a row that another unfinished transaction has read, and cannot read a row that another unfinished transaction has written.
Here Alice's transaction takes Alice off call before Bob's transaction reads.
Bob's read therefore waits until Alice's transaction commits, and then sees that only Bob remains on call.
No transaction aborts, and the result matches running Alice's transaction and then Bob's.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A (Alice)
    participant DB as Database
    participant B as Transaction B (Bob)
    Note over A,B: Initially Alice and Bob are on call<br/>At least one must remain
    A->>DB: BEGIN at Serializable
    B->>DB: BEGIN at Serializable
    A->>DB: Read on-call doctors
    DB-->>A: Alice and Bob are on call, retain shared locks
    A->>DB: Set Alice off call
    Note over A,DB: A upgrades its lock on Alice to exclusive
    B->>DB: Read on-call doctors
    Note over DB,B: B waits for A's exclusive lock on Alice
    A->>DB: COMMIT and release locks
    DB-->>B: Only Bob remains on call
    Note over B: Keep Bob on call
    B->>DB: COMMIT
    Note over A,B: No abort<br/>The result matches A, then B
```

If both transactions read before either writes, as in the other sections, each then waits for the other's read lock, which is the [lock-upgrade deadlock](#serializable-isolation-with-locks-lock-upgrade).

## Serializable Snapshot Isolation

| Dirty read | Nonrepeatable read | Phantom read | Read skew | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| Prevented | Prevented | Prevented | Prevented | Prevented | Prevented |

- Combines snapshot reads with checks for dependency patterns that could produce a nonserializable result.
- Transactions run concurrently, but one may be aborted to preserve serializable results.
- Conflicting row updates may still wait for writers.
- Applications must be prepared to retry the entire transaction after a serialization failure.

**PostgreSQL:** `SERIALIZABLE` uses this model.

- It records reads with predicate locks, shown as `SIReadLock` in `pg_locks`.
  These locks never block and cannot cause deadlocks; they only track read/write dependencies between transactions.
- A serialization failure returns SQLSTATE `40001`.
  Catch that code, roll back, and rerun the whole transaction from the start without reusing data read by the failed attempt.

### Normal Flow

The transactions read and update separate counters, and the dependency checks allow both to commit.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    Note over A,B: Initially counter A = 10 and counter B = 20<br/>Each transaction updates only its own counter
    A->>DB: BEGIN at Serializable
    B->>DB: BEGIN at Serializable
    A->>DB: Read counter A from snapshot
    DB-->>A: Return 10
    B->>DB: Read counter B from snapshot
    DB-->>B: Return 20
    A->>DB: Write counter A = 11
    B->>DB: Write counter B = 21
    A->>DB: Request COMMIT
    DB-->>A: Checks pass, commit succeeds
    B->>DB: Request COMMIT
    DB-->>B: Checks pass, commit succeeds
```

### Conflict and Recovery: Serialization Failure

Here, the writes affect different rows, so they can proceed without a write/write conflict.
The dependency check prevents both transactions from committing the write-skew outcome.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A (Alice)
    participant DB as Database
    participant B as Transaction B (Bob)
    Note over A,B: Initially Alice and Bob are on call<br/>At least one must remain
    A->>DB: BEGIN at Serializable
    B->>DB: BEGIN at Serializable
    A->>DB: Read on-call doctors
    DB-->>A: Alice and Bob are on call
    B->>DB: Read on-call doctors
    DB-->>B: Alice and Bob are on call
    A->>DB: Set Alice off call
    B->>DB: Set Bob off call
    A->>DB: Request COMMIT
    DB-->>A: Commit succeeds
    B->>DB: Request COMMIT
    DB-->>B: Serialization failure, roll back B
    B->>DB: Retry the entire transaction
    B->>DB: Read on-call doctors
    DB-->>B: Only Bob remains on call
    Note over B: Keep Bob on call
    B->>DB: COMMIT
```

## Deadlocks

**Every model in this note can deadlock.**
A deadlock needs transactions that hold locks while waiting for others, in a cycle of waits.
All five models hold exclusive locks on modified rows until the transaction ends, so writes alone can form that cycle.
MVCC stops ordinary reads from blocking writes, but writers still block other writers.
Deadlocks are much more frequent under Serializable Isolation with Locks than under Read Committed with Locks, because shared read locks are held until the transaction ends.
Each retry redoes the transaction's work, so frequent deadlocks waste effort.

Serialization failures under snapshot isolation and serializable snapshot isolation are a separate kind of abort.
They are not deadlocks, but applications handle both by retrying.

The three examples below form the cycle in different ways.

| | [Write/write](#writewrite-deadlock) | [A read completes the cycle](#read-committed-with-locks-a-read-completes-the-cycle) | [Lock upgrade](#serializable-isolation-with-locks-lock-upgrade) |
| --- | --- | --- | --- |
| Applies to | All five models | Read Committed with Locks | Serializable Isolation with Locks |
| Steps | Each updates one row, then the other row, in opposite order | Each updates one row, then reads the row the other updated | Both read the same row, then both try to update it |
| Locks in the cycle | Exclusive write locks | Write locks that block reads | Shared read locks that block the upgrade to a write lock |
| Writes before the deadlock | One each | One each | None |
| After the abort | A continues, and B retries | A continues, and B retries | A commits, and B's retry reads A's new value, so no update is lost |

The [doctor example](#conflict-and-recovery-blocking) under Serializable Isolation with Locks hits this same deadlock if both transactions read before either writes.

### Write/Write Deadlock

This example applies to every model in this note.
The transactions update the same two rows in opposite order.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    A->>DB: BEGIN
    B->>DB: BEGIN
    A->>DB: Update row 1
    Note over A,DB: A holds a write lock on row 1
    B->>DB: Update row 2
    Note over DB,B: B holds a write lock on row 2
    A->>DB: Update row 2
    Note over A,DB: A waits for B's lock on row 2
    B->>DB: Update row 1
    Note over DB,B: B waits for A's lock on row 1
    Note over A,B: Each waits for the other
    DB-->>B: Abort B as the deadlock victim
    DB-->>A: A's update can now finish
    A->>DB: COMMIT
    B->>DB: Retry in a new transaction
```

### Read Committed with Locks: A Read Completes the Cycle

Read locks are released early in this model, but a read must still wait for an uncommitted write.
A transaction that holds a write lock can therefore deadlock while waiting to read.
Under Read Committed with MVCC, both reads would return the last committed versions without waiting.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    A->>DB: BEGIN at Read Committed
    B->>DB: BEGIN at Read Committed
    A->>DB: Update row 1
    Note over A,DB: A holds a write lock on row 1
    B->>DB: Update row 2
    Note over DB,B: B holds a write lock on row 2
    A->>DB: Read row 2
    Note over A,DB: A waits for B's write lock on row 2
    B->>DB: Read row 1
    Note over DB,B: B waits for A's write lock on row 1
    Note over A,B: Deadlock caused by a read waiting for a write
    DB-->>B: Abort B as the deadlock victim
    B->>DB: Retry in a new transaction
```

### Serializable Isolation with Locks: Lock Upgrade

This model holds shared read locks until the transaction ends, and any number of transactions can share one.
A write must upgrade the shared lock to an exclusive lock, which waits until no other transaction holds a shared lock on that row.
When two transactions read the same row and then both try to update it, each waits for the other's read lock.
No second row or business rule is needed; the anomaly this abort prevents is a lost update.

```mermaid
sequenceDiagram
    autonumber
    participant A as Transaction A
    participant DB as Database
    participant B as Transaction B
    Note over A,B: Initially counter = 10
    A->>DB: BEGIN at Serializable
    B->>DB: BEGIN at Serializable
    A->>DB: Read counter with a shared lock
    DB-->>A: Return 10
    B->>DB: Read counter with a shared lock
    DB-->>B: Return 10
    A->>DB: Write counter = 11
    Note over A,DB: A waits for B's shared lock on counter
    B->>DB: Write counter = 11
    Note over DB,B: B waits for A's shared lock on counter
    Note over A,B: Each waits for the other
    DB-->>B: Abort B as the deadlock victim
    DB-->>A: A's update can now finish
    A->>DB: COMMIT
    B->>DB: Retry in a new transaction
    B->>DB: Read counter
    DB-->>B: Return 11
    B->>DB: Write counter = 12
    B->>DB: COMMIT
    Note over A,B: Both increments are kept<br/>counter = 12
```

### Reducing Deadlocks

1. Access rows in a consistent order, for example by sorted primary key.
2. Keep transactions short, and avoid user interaction or network calls while holding locks.
3. Lock rows before reading them for an update, for example with `SELECT ... FOR UPDATE`, instead of upgrading a shared lock later.
4. Catch deadlock and serialization errors, roll back, and retry the entire transaction.
   In PostgreSQL, a deadlock victim fails with SQLSTATE `40P01` and a serialization failure with `40001`.
