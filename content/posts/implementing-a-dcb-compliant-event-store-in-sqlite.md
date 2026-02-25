+++
date = '2026-02-25T07:30:29-05:00'
title = 'Implementing a DCB-Compliant Event Store in SQLite'
tags = ['Event Sourcing', 'DCB']
+++

Dynamic Consistency Boundary doesn't require specialized infrastructure. A four-column SQLite table is enough.

The [DCB specification](https://dcb.events/specification/) defines the minimal feature set an Event Store must provide to support Dynamic Consistency Boundary. This article walks through how to implement it using SQLite, requirement by requirement.

This approach relies on SQLite's global file-level lock to guarantee consistency. It cannot be applied as-is with MariaDB, MySQL, or PostgreSQL. See the appendix for details.

---

## The READ Requirements

Let's go through each READ requirement from the spec and map it to a column.

### Event Type and Tag Filtering

> "... MUST provide a way to filter Events based on their Event Type and/or Tag"

This requires two columns.

`type TEXT` stores the **Event Type**, a plain string like `CourseDefined`.

`tags TEXT` stores the **Tags**, as a JSON object where keys and values are strings. For example, a `StudentSubscribedToCourse` affecting student `s1` and course `c1` would have:

```json
{"courseId": "c1", "studentId": "s1"}
```

Filtering by Event Type uses `IN`, filtering by Tag uses `JSON_EXTRACT`:

```sql
WHERE `type` IN ('CourseDefined', 'CourseCapacityChanged')
AND IFNULL(JSON_EXTRACT(`tags`, '$.courseId'), '') = 'c1'
```

The `IFNULL` wrapper ensures that if the key does not exist in the JSON object, the comparison safely returns false instead of NULL.

### Starting Sequence Position

> "... SHOULD provide a way to read Events from a given starting Sequence Position"

This requires one column.

`sequence INTEGER PRIMARY KEY AUTOINCREMENT` stores the **Sequence Position**. It is unique, assigned by the database on insert, and always increasing. It may contain gaps.

Reading from a given position is a simple range condition:

```sql
WHERE `sequence` >= 100 -- starting position
```

### Additional Filter Options

> "... MAY provide further filter options, e.g. for ordering or to limit the number of Events to load at once"

`ORDER BY sequence ASC` returns events in chronological order. `LIMIT` allows pagination.

### Event Data

`data TEXT` stores the **Event Data**, the opaque data of the event. The spec does not define any read filtering on this column. It is simply returned as part of each event.

### The Minimal Table

From these requirements, we get the minimal schema:

```sql
CREATE TABLE `events` (
    `sequence` INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    `type` TEXT NOT NULL,
    `tags` TEXT NOT NULL,
    `data` TEXT NOT NULL
);
```

Four columns. That is all a DCB-compliant Event Store strictly requires.

In practice, three additional columns are worth adding: `uid` for unique identification, `metadata` for technical context like correlation ids, and `time` for the recorded timestamp. The spec does not require them, but they have their place in any production system.

### The Full Read Query

Here is the complete read query for our example. The goal is to load the events needed to check whether course `c1` exists and what its current capacity is:

```sql
SELECT *
FROM `events`
WHERE `sequence` >= 0
AND `type` IN ('CourseDefined', 'CourseCapacityChanged')
AND IFNULL(JSON_EXTRACT(`tags`, '$.courseId'), '') = 'c1'
ORDER BY `sequence` ASC;
```

Suppose this query returns the following events:

| sequence | type              | tags      |
|----------|-------------------------|------------------------|
| 12       | `CourseDefined`           | `{"courseId": "c1"}`     |
| 47       | `CourseCapacityChanged`   | `{"courseId": "c1"}`     |
| 562      | `CourseCapacityChanged`   | `{"courseId": "c1"}`     |

The highest `sequence` returned is **562**. This value will be used as the `after` parameter in the Append Condition.

---

## The WRITE Requirements

### The Traditional Approach: Optimistic Locking with Aggregates

Before looking at how DCB handles writes, it is worth seeing how a traditional aggregate-based Event Store solves the same problem, because the contrast is instructive.

With aggregates, every event belongs to a stream identified by a type and an id. Each event in a stream has an incrementing version number. The schema looks like this:

```sql
CREATE TABLE `event_store` (
    `stream_type` TEXT NOT NULL,
    `stream_id` TEXT NOT NULL,
    `stream_version` INTEGER NOT NULL,
    `event_type` TEXT NOT NULL,
    `event_data` TEXT NOT NULL,
    UNIQUE (`stream_type`, `stream_id`, `stream_version`)
);
```

The unique index on `(stream_type, stream_id, stream_version)` is the entire optimistic locking mechanism. No application logic required.

Suppose the table already contains these events:

| stream_type | stream_id | stream_version | type              |
|-------------|-----------|----------------|-------------------------|
| `course`      | 4         | 1              | `CourseDefined`           |
| `course`      | 4         | 2              | `CourseCapacityChanged`   |
| `course`      | 4         | 3              | `CourseCapacityChanged`   |

**Success scenario**: a writer inserts version 4. No conflict, the row is accepted.

```sql
INSERT INTO `event_store` (`stream_type`, `stream_id`, `stream_version`, `event_type`, `event_data`)
VALUES ('course', '4', 4, 'CourseCapacityChanged', '{"new": 20}');
-- 1 row inserted
```

**Concurrency failure scenario**: two writers both try to insert version 4 at the same time. The second one gets a unique constraint violation.

```sql
INSERT INTO `event_store` (`stream_type`, `stream_id`, `stream_version`, `event_type`, `event_data`)
VALUES ('course', '4', 4, 'CourseCapacityChanged', '{"new": 50}');
-- UNIQUE constraint failed: `event_store`.`stream_type`, `event_store`.`stream_id`, `event_store`.`stream_version`
```

The database enforces consistency. The application catches the exception and retries. Simple and elegant.

The limitation is that the consistency boundary is fixed: one aggregate, one stream. When a business rule spans multiple aggregates, this model breaks down. That is the problem DCB solves, but with it comes a different challenge: the unique index trick no longer works.

### The Conditional INSERT

With DCB there are no streams, no version numbers, and no unique index to rely on. We need a different mechanism. Before introducing the DCB solution, let's look at the underlying SQL technique it relies on.

Imagine a waitlist system. A waitlist is capped at 10 spots. We want to add an entry only if the waitlist is not yet full.

```sql
INSERT INTO `waitlist` (`event_id`, `user_id`)
SELECT 'e1', 'u1'
WHERE (SELECT COUNT(*) FROM `waitlist` WHERE `event_id` = 'e1') < 10;
```

This is a conditional INSERT: an `INSERT ... SELECT ... WHERE` where the WHERE clause controls whether any rows are actually inserted. If the waitlist is already full, the condition is false, zero rows are inserted, and no error is raised. The caller detects the failure by checking the number of affected rows: zero means the waitlist was full.

DCB can leverage this technique as one way to enforce consistency at write time.

### Atomic Persistence

> "... MUST provide a way to atomically persist one or more Event(s)"

A single `INSERT` statement is atomic by nature in SQLite. To insert multiple events atomically in a single statement, we use `UNION` in the source subquery, with a `union_index` column to preserve insertion order:

```sql
INSERT INTO `events` (`type`, `tags`, `data`)
SELECT `type`, `tags`, `data`
FROM (
    SELECT
        0 `union_index`,
        'SomethingHappened' `type`,
        '{"courseId":"c1"}' `tags`,
        '{}' `data`
    UNION
    SELECT
        1 `union_index`,
        'SomethingElseHappened' `type`,
        '{"courseId":"c1"}' `tags`,
        '{}' `data`
    ORDER BY `union_index` ASC
);
```

### The Append Condition

> "... MUST fail if the Event Store contains at least one Event matching the Append Condition, if specified"

The **Append Condition** is defined in the spec as:

```
AppendCondition {
  failIfEventsMatch: Query
  after?: SequencePosition
}
```

`failIfEventsMatch` is the same Query used during the read. `after` is the highest Sequence Position the client was aware of when building the decision model, which is the `MAX(sequence)` observed at read time.

The optimistic locking works in two phases.

**Phase 1 - Read**: execute the Query, build the decision model, and record the highest `sequence` returned. In our example, that value is **562**.

**Phase 2 - Write**: attempt the append using a conditional INSERT. The WHERE clause re-runs the exact same Query inside a subquery and checks that `MAX(sequence)` still equals the value recorded in phase 1. If another writer inserted a relevant event in the meantime, `MAX(sequence)` will have increased, the condition will be false, and zero rows will be inserted.

Here is the full INSERT for our course `c1` example, appending a new `CourseCapacityChanged` with `after = 562`:

```sql
INSERT INTO `events` (`type`, `tags`, `data`)
SELECT `type`, `tags`, `data`
FROM (
    SELECT
        0 `union_index`,
        'CourseCapacityChanged' `type`,
        '{"courseId": "c1"}' `tags`,
        '{"previous": 10, "new": 20}' `data`
    ORDER BY `union_index` ASC
)
WHERE (
    -- The exact same Query used during the read
    SELECT IFNULL(MAX(`sequence`), 0)
    FROM `events`
    WHERE `type` IN ('CourseDefined', 'CourseCapacityChanged')
    AND IFNULL(JSON_EXTRACT(`tags`, '$.courseId'), '') = 'c1'
) = 562; -- MAX sequence observed during the read
```

If `MAX(sequence)` is still 562, the condition holds and the event is inserted. Otherwise, a concurrent writer got there first: zero rows are inserted, no error is raised. The application detects the failure by checking the number of affected rows.

### When `after` is omitted

> "if omitted, no Events will be ignored, effectively failing if any Event matches the specified Query"

When `after` is not provided, the expected value is 0. The condition becomes `= 0`, which passes only when no matching events exist at all. This is the right approach for creation scenarios. For example, creating a user that must not already exist:

```sql
WHERE (
    SELECT IFNULL(MAX(`sequence`), 0)
    FROM `events`
    WHERE `type` = 'UserCreated'
    AND IFNULL(JSON_EXTRACT(`tags`, '$.userId'), '') = '123'
) = 0;
-- passes only if no UserCreated exists for userId 123
```


### Why Not Pessimistic Locking?

With pessimistic locking, a writer would acquire a table-level lock before reading, hold it through the entire decision-making process, and release it only after writing. This guarantees consistency but blocks all other writers for the full duration, even when their changes are completely unrelated.

The conditional INSERT takes a different approach. The read phase runs without any lock. Only the write is atomic. If another writer inserted a relevant event in the meantime, the condition fails and zero rows are inserted. The cost is paid only when there is an actual conflict, not on every write.

---

## Conclusion

A DCB-compliant Event Store does not require specialized infrastructure. A standard relational database, a four-column table, and carefully written SQL are enough.

The key insight is that the same Query used to read events is reused verbatim inside the conditional INSERT to enforce the Append Condition. The database handles consistency atomically, with no locks and no complex application logic.

This is the approach used by the [PdoEventStore](https://github.com/backslashphp/backslash/tree/2.x/src/PdoEventStore) component of [Backslash](https://backslashphp.github.io/), an open-source PHP Event Sourcing library.

---

## Appendix: A Note on MariaDB, MySQL, and PostgreSQL

The conditional INSERT technique described in this article works safely with SQLite because SQLite uses a global file-level lock. Only one writer can execute at a time, so two concurrent sessions will naturally serialize: the second one evaluates the `WHERE` clause only after the first has finished. No race condition is possible.

With MariaDB, MySQL, and PostgreSQL, multiple sessions can evaluate the `WHERE` clause simultaneously. Two sessions with overlapping queries can both observe the same `MAX(sequence)`, both conclude that the condition is satisfied, and both insert their events. The autoincrement guarantees distinct `sequence` values, but both events are appended when only one should have been. The consistency boundary is violated.

To prevent this, wrap the conditional INSERT in a transaction with `SERIALIZABLE` isolation:

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;

INSERT INTO `events` (`type`, `tags`, `data`)
SELECT `type`, `tags`, `data`
FROM (
    SELECT
        0 `union_index`,
        'CourseCapacityChanged' `type`,
        '{"courseId": "c1"}' `tags`,
        '{"previous": 10, "new": 20}' `data`
    ORDER BY `union_index` ASC
)
WHERE (
    SELECT IFNULL(MAX(`sequence`), 0)
    FROM `events`
    WHERE `type` IN ('CourseDefined', 'CourseCapacityChanged')
    AND IFNULL(JSON_EXTRACT(`tags`, '$.courseId'), '') = 'c1'
) = 562;

COMMIT;
```

`SERIALIZABLE` isolation ensures that two sessions with overlapping queries cannot both evaluate the `WHERE` clause against the same state of the table. The second session blocks until the first has committed, then re-evaluates the condition against the updated state.

The trade-off is that sessions with overlapping queries will block each other. In practice, this only affects writers that are competing over the same consistency boundary, which is exactly the behavior we want.