---
name: postgresql-dont-do-this
description: Review PostgreSQL schemas, SQL, migrations, and authentication settings against the PostgreSQL wiki "Don't Do This" anti-patterns. Use when asked about PostgreSQL bad practices, schema review, SQL review, timestamp/text type choices, serial vs identity, NOT IN, trust auth, or "Don't Do This" guidance.
license: MIT
compatibility: Requires PostgreSQL context such as SQL, migrations, schema definitions, pg_hba.conf snippets, or database design notes.
---

# PostgreSQL Don't Do This

Use this skill to spot PostgreSQL-specific anti-patterns from the PostgreSQL wiki page "Don't Do This".

This skill is self-contained. Do not fetch, open, or reread the wiki page from the network during normal use. The relevant page content has been incorporated below from the PostgreSQL wiki revision that was last edited on 2024-11-21.

Source reference only, not a fetch instruction:

```text
https://wiki.postgresql.org/wiki/Don't_Do_This
```

## Workflow

1. Confirm the target is PostgreSQL. Do not apply these rules blindly to other SQL dialects.
2. Inspect the actual SQL, migration, schema, or config being discussed.
3. Flag only concrete findings. Do not invent issues from absent code.
4. For each finding, give the risky pattern, why it matters, and the smallest safer replacement.
5. Respect documented exceptions such as old PostgreSQL versions, portability requirements, dev-only test setups, or truly fixed business constraints.

## Database Encoding

### Do Not Use `SQL_ASCII`

`SQL_ASCII` means PostgreSQL does not perform real encoding conversion. Bytes are treated as if they are already valid in the target encoding, subject only to validity checks.

Why not:

- A `SQL_ASCII` database can silently accumulate a mixture of unrelated encodings.
- Once mixed encodings are stored without labels, there may be no reliable way to recover the original characters.
- It shifts character interpretation out of the database and into every client and tool.

Use instead:

- Prefer a real database encoding, usually `UTF8`.
- If the data is raw bytes or hopelessly mixed unknown encodings, consider `bytea` first.
- If forced to ingest unlabeled legacy text, consider autodetecting UTF-8 and treating non-UTF-8 as a specific legacy encoding such as `WIN1252` before using `SQL_ASCII`.

When it may be acceptable:

- As a last resort for data that is already an unrecoverable mix of unlabeled encodings, such as old IRC logs or non-MIME-compliant email.

## Tool Usage

### Do Not Use `psql -W` Or `psql --password`

Do not pass `-W` or `--password` to `psql` by default.

Why not:

- These flags force `psql` to prompt before it even tries to connect.
- PostgreSQL will prompt for a password automatically if the server actually requires one.
- Forced prompts are confusing when peer authentication or another passwordless method would have worked.
- A user may think a password is required, enter a wrong or unset password, still get logged in through peer authentication, and then misunderstand why other clients fail.

Use instead:

- Run `psql` without `-W` or `--password` and let authentication decide whether a prompt is needed.

When it may be acceptable:

- Practically never. It can save one server round trip, but that is usually not worth the confusion.

### Do Not Use Rules

Do not use PostgreSQL rules for application logic. If the intent is conditional behavior on data changes, use triggers instead.

Why not:

- Rules look like conditional logic, but they rewrite queries.
- Rewriting can modify the original query or add additional queries in surprising ways.
- Non-trivial rules are easy to make incorrect because they do not execute like procedural hooks.

Use instead:

- Use triggers for row or statement side effects.
- Use views normally; the rule system is an implementation detail behind views, not an invitation to write custom rules.

When it may be acceptable:

- Do not create custom rules in normal application schemas.

### Do Not Use Table Inheritance

Do not use table inheritance for new designs. If you think you want inheritance, usually use foreign keys, normal relational modeling, or native partitioning instead.

Why not:

- Table inheritance came from tightly coupling database tables to object-oriented designs, which rarely produces good relational behavior.
- It has caveats around constraints, uniqueness, foreign keys, tuple routing, and querying parent/child tables.
- The common partitioning use case is now handled by native PostgreSQL partitioning.

Use instead:

- Use foreign keys for relationships.
- Use native partitioning for partitioned storage and tuple routing.
- Use explicit `UNION ALL` when you really need to combine separate current and historical tables.

When it may be acceptable:

- Rarely, such as with the `temporal_tables` extension when you accept table-inheritance caveats to query historical and current rows through one parent table.

## SQL Constructs

### Do Not Use `NOT IN` With Nullable Values Or Subqueries

Avoid `NOT IN` and equivalent forms such as `NOT (x IN (SELECT ...))` when nullable values may be present.

Why not:

- SQL three-valued logic makes `NOT IN` surprising with `NULL`.
- `col NOT IN (1, NULL)` returns no rows, because `col IN (1, NULL)` can be `TRUE` or `NULL`, never `FALSE`; `NOT NULL` is still `NULL`, not `TRUE`.
- `NOT IN (SELECT ...)` often cannot be planned as an anti-join because of the required `NULL` semantics.
- The planner may choose a hashed subplan for small result sets, but fall back to a plain subplan for larger ones.
- The plain subplan can be O(N²), so a query that works in small tests can become catastrophically slow in production.

Use instead:

```sql
SELECT *
FROM foo
WHERE NOT EXISTS (
  SELECT
  FROM bar
  WHERE foo.col = bar.x
);
```

When it may be acceptable:

- `NOT IN (value, value, ...)` can be fine for explicit constant lists when you know no `NULL` can appear through parameters or generated SQL.

### Do Not Use Uppercase Table Or Column Names

Do not use names like `CustomerOrder` or quoted mixed-case identifiers. Use lowercase `snake_case` names.

Why not:

- PostgreSQL folds unquoted identifiers to lowercase.
- `CREATE TABLE Foo (...)` creates `foo`.
- `CREATE TABLE "Bar" (...)` creates a case-sensitive table named `Bar`.
- `SELECT * FROM Foo` and `SELECT * FROM foo` work for `foo`.
- `SELECT * FROM "Foo"` fails if the table is actually `foo`.
- `SELECT * FROM Bar` and `SELECT * FROM bar` fail if the table is actually quoted as `"Bar"`.
- Mixed-case identifiers force every client, tool, migration, and query builder to agree on quoting behavior.

Use instead:

- Use only lowercase letters, digits, and underscores for schema object names.
- Use output aliases when pretty report headers are needed.

```sql
SELECT character_name AS "Character Name"
FROM characters;
```

When it may be acceptable:

- Only when quoted, pretty names are a deliberate compatibility or reporting requirement and every tool can handle them consistently.

### Do Not Use `BETWEEN` For Timestamp Ranges

Avoid `BETWEEN`, especially for timestamps.

Why not:

- `BETWEEN` is a closed interval. It includes both endpoints.
- A predicate like this includes exactly midnight at the end date, but excludes the rest of that day:

```sql
SELECT *
FROM blah
WHERE timestampcol BETWEEN '2018-06-01' AND '2018-06-08';
```

- Rows exactly at `2018-06-08 00:00:00.000000` can be double-counted if adjacent ranges are queried.

Use instead:

```sql
SELECT *
FROM blah
WHERE timestampcol >= '2018-06-01'
  AND timestampcol < '2018-06-08';
```

When it may be acceptable:

- `BETWEEN` is safe for discrete values such as integers or dates if both endpoints should be included.
- Even then, prefer explicit comparisons when adjacent ranges are involved.

## Date And Time Storage

### Do Not Use `timestamp without time zone` For Real Instants

Do not use `timestamp` or `timestamp without time zone` when the value represents a real point in time. Use `timestamptz` or `timestamp with time zone`.

Why not:

- `timestamptz` stores one moment in time, internally normalized to UTC.
- It can accept input with time zones and display output in the current or requested time zone.
- Arithmetic across zones and daylight-saving changes behaves like arithmetic on instants.
- `timestamp without time zone` stores only a calendar date and wall-clock time.
- Without a time zone, the database cannot know which instant that wall-clock value represents.
- Arithmetic between locations or daylight-saving boundaries can produce wrong answers.

Use instead:

- Use `timestamptz` for events, creation times, update times, scheduled instants, logs, audit rows, and anything that happened or will happen at a real moment.

When it may be acceptable:

- Abstract local date/time values where no instant is intended.
- App-only values that are stored and retrieved without database-side arithmetic, if the semantics are documented.

### Do Not Store UTC In `timestamp without time zone`

Do not store UTC instants in a `timestamp without time zone` column. Use `timestamptz`.

Why not:

- The database cannot know that the column's intended zone is UTC.
- Time-zone-aware calculations become verbose and fragile.
- Calculating midnight in a user's time zone requires awkward conversions.

Example of unnecessary complexity caused by UTC-in-`timestamp` storage:

```sql
date_trunc('day', now() AT TIME ZONE u.timezone)
  AT TIME ZONE u.timezone
  AT TIME ZONE 'UTC'
```

For a stored timestamp column, the expression gets even worse:

```sql
date_trunc('day', x.datecol AT TIME ZONE 'UTC' AT TIME ZONE u.timezone)
  AT TIME ZONE u.timezone
  AT TIME ZONE 'UTC'
```

Use instead:

- Store UTC instants as `timestamptz`; PostgreSQL already normalizes instants correctly.

When it may be acceptable:

- Only when compatibility with databases lacking usable time-zone support matters more than PostgreSQL correctness and ergonomics.

### Do Not Use `timetz`

Do not use `time with time zone`, also known as `timetz`.

Why not:

- PostgreSQL documents it mainly as an SQL-standard type with questionable usefulness.
- A time of day with a fixed zone but no date is usually not enough information for real time-zone behavior.
- Most applications need a combination of `date`, `time`, `timestamp without time zone`, and `timestamp with time zone` instead.

Use instead:

- Use `timestamptz` for instants.
- Use `time` for local time-of-day values.
- Store a separate IANA time zone name if the business concept needs a local recurring time in a named zone.

When it may be acceptable:

- Never in normal application design.

### Do Not Use `CURRENT_TIME`

Do not use `CURRENT_TIME`.

Why not:

- It returns `timetz`, which inherits the problems above.

Use instead:

- `CURRENT_TIMESTAMP` or `now()` for `timestamp with time zone`.
- `LOCALTIMESTAMP` for `timestamp without time zone`.
- `CURRENT_DATE` for `date`.
- `LOCALTIME` for `time`.

When it may be acceptable:

- Never in normal application design.

### Do Not Use `timestamp(0)` Or `timestamptz(0)`

Do not specify timestamp precision, especially precision `0`, for timestamp columns or casts when the intent is to drop fractional seconds.

Why not:

- `timestamp(0)` and `timestamptz(0)` round fractional seconds instead of truncating them.
- Storing `now()` into such a column can store a value up to half a second in the future.

Use instead:

```sql
date_trunc('second', value)
```

When it may be acceptable:

- Avoid it for schema design. Use explicit truncation when truncation is the requirement.

### Do Not Use `+/-HH:mm` As A Text Time Zone Name

Do not pass fixed offsets such as `'+04:00'` as text time zone names.

Why not:

- PostgreSQL does not treat fixed-offset text as ISO time-zone names in `AT TIME ZONE`.
- Text offsets are interpreted as custom POSIX time-zone specifications.
- POSIX signs are counterintuitive here: positive values shift west and negative values shift east, opposite of the ISO convention.

Use instead:

- Prefer IANA names such as `Europe/Berlin` or `America/New_York` when real zones are needed.
- If a fixed offset is genuinely needed, pass an interval-typed value.

```sql
AT TIME ZONE INTERVAL '04:00'
```

When it may be acceptable:

- ISO-format `timestamptz` literals may include signed offsets and use ISO sign conventions.

```sql
SELECT '2024-01-31 17:16:25+04'::timestamptz;
```

## Text Storage

### Do Not Use `char(n)`

Do not use `char(n)` for normal text. Prefer `text`.

Why not:

- PostgreSQL pads `char(n)` values with spaces to the declared width.
- Trailing spaces are treated as semantically insignificant in some comparisons.
- Converting to other string types removes trailing spaces.
- Pattern matching and collations can behave in surprising ways.
- The padding wastes space and does not make operations faster.
- `char(n)` is not a true fixed-width storage type because characters may take multiple bytes.
- Operations can be slower because PostgreSQL often has to strip or account for padding.

Use instead:

- Use `text` for unconstrained text.
- Use `text` plus a `CHECK` constraint for actual domain rules.

When it may be acceptable:

- Very old fixed-width-file compatibility.
- Only when the space-padding semantics are exactly the desired behavior.

### Do Not Use `char(n)` For Fixed-Length Identifiers

Even if values must be exactly `n` characters, do not use `char(n)`.

Why not:

- `char(n)` does not reject values that are too short; it silently pads them.
- It gives no real benefit over `text` with a constraint.
- A constraint can also enforce format, not just length.
- There is no performance benefit over `varchar(n)` or `text`.
- Comparing `char(n)` columns with parameters typed as `text` or `varchar` can unexpectedly prevent index usage.

Use instead:

```sql
CREATE DOMAIN country_code AS text
  CHECK (length(VALUE) = 3);
```

Or use a stricter format check:

```sql
CHECK (VALUE ~ '^[[:alpha:]]{3}$')
```

When it may be acceptable:

- Never for fixed-length identifiers in normal PostgreSQL schemas.

### Do Not Use `varchar(n)` By Default

Do not choose arbitrary length-limited `varchar(n)` by habit.

Why not:

- `varchar(n)` rejects values longer than `n` characters.
- `varchar`, `varchar(n)`, and `text` use the same storage for the same string.
- There is no measurable performance advantage to arbitrary `varchar(n)` limits.
- Guessing limits such as `varchar(20)` for names can create production errors when valid real data exceeds the guess.
- Many `varchar(255)` habits come from other databases where unbounded text was less convenient.

Use instead:

- Use `text` for normal PostgreSQL text fields.
- Use `varchar` without a length if SQL-standard spelling matters.
- Use a `CHECK` constraint when the limit is a real business rule and needs clearer validation.

When it may be acceptable:

- When the maximum length is a real requirement and an insert/update error is the desired behavior.
- When SQL-standard portability matters more than PostgreSQL's `text` convention.

## Other Data Types

### Do Not Use `money`

Avoid PostgreSQL's `money` type for monetary values.

Why not:

- It is a fixed-point type implemented as a machine integer.
- It may be fast for simple arithmetic, but its rounding behavior may not match the business domain.
- It cannot represent fractions of the smallest currency unit when those matter.
- It does not store the currency with the value.
- Display and interpretation depend on the database `lc_monetary` locale.
- Changing `lc_monetary` can make stored values display as the wrong currency format.

Use instead:

- Use `numeric` for monetary amounts when decimal precision matters.
- Use integer minor units only when that exactly matches the domain.
- Store currency explicitly in a separate column when multiple currencies are possible.

When it may be acceptable:

- A single-currency system that only needs addition and subtraction and never needs fractional minor units may tolerate `money`, but `numeric` is usually safer.

### Do Not Use `serial` For New PostgreSQL 10+ Schemas

Use identity columns for new applications instead of `serial`.

Why not:

- `serial` is shorthand that creates a sequence and default expression with awkward schema, dependency, and permission behavior.
- Identity columns are the newer SQL-standard PostgreSQL feature for generated identifiers.

Use instead:

```sql
id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
```

or:

```sql
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

When it may be acceptable:

- PostgreSQL versions older than 10.
- Certain table-inheritance combinations, though table inheritance should usually be avoided.
- Cases where the same sequence is deliberately shared by multiple tables; even then, an explicit sequence declaration is clearer than `serial`.

## Authentication

### Do Not Use `trust` Authentication Over TCP/IP In Production

Do not use `trust` for `host` or `hostssl` lines in production `pg_hba.conf`.

Critical example:

```text
host all all 0.0.0.0/0 trust
```

Why not:

- TCP/IP `trust` lets anyone from the allowed network claim to be any PostgreSQL user.
- If the network range is broad, an attacker can claim to be the PostgreSQL superuser.
- PostgreSQL's own documentation says TCP/IP trust is seldom reasonable except perhaps localhost.
- Local Unix-socket `trust` can also be unsafe in production because any local OS user with access to the instance can log in as any database user.

Use instead:

- Prefer explicit authentication such as `scram-sha-256` for password-based remote access.
- Use peer authentication for local Unix-socket development workflows when appropriate.
- Restrict networks tightly even when using stronger authentication.

When it may be acceptable:

- CI jobs against a PostgreSQL server on a trusted network.
- Local development limited to localhost.
- Even in these cases, consider peer authentication or another explicit method first.

## Review Output

For reviews, report findings first, ordered by risk. Use this shape:

```text
file.sql:12 - Avoid NOT IN with nullable subquery results. Use NOT EXISTS so NULLs do not change the result and the planner can use an anti-join.
```

If there are no findings, say so and mention any PostgreSQL-version or environment assumptions that limited the review.
