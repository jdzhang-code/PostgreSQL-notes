# Day 1 Notes: psql and SQL Q&A (2026-11-09)

Notes from questions that came up while practicing on PostgreSQL 16 (running in Docker) with the `shop` practice database and the [Pagila](https://github.com/devrimgunduz/pagila) sample database. Every example was run against a local Pagila instance and the results shown are real.

## Contents

1. [What is DBeaver](#1-what-is-dbeaver)
2. [Reading the output of `\d customers`](#2-reading-the-output-of-d-customers)
3. [Executing statements in psql](#3-executing-statements-in-psql)
4. [ORDER BY ties and pagination](#4-order-by-ties-and-pagination)
5. [NULL vs. empty string: COALESCE and NULLIF](#5-null-vs-empty-string-coalesce-and-nullif)
6. [LIKE vs. ILIKE](#6-like-vs-ilike)
7. [Quick reference](#quick-reference)

---

## 1. What is DBeaver

DBeaver is a free, cross-platform GUI database client. It connects to the same database as psql; the only difference is the interface: psql is typed commands, DBeaver is point-and-click. The free Community edition is enough for practice.

| Task | psql | DBeaver |
| --- | --- | --- |
| View a table's structure | `\d customers` | Expand the table in the left tree; see Columns / Constraints / Indexes |
| See how tables relate | Read the foreign keys in `\d` | Auto-generated ER diagram |
| Browse and filter data | Write a `SELECT` | Open the table's Data tab and scroll like a spreadsheet |
| Write and run SQL | Type it in | SQL editor with autocomplete; Ctrl/Cmd+Enter runs the current statement |

To connect to Postgres running in local Docker: create a new PostgreSQL connection with Host `localhost`, Port `5432`, Database `shop` or `pagila`, and the username and password from `docker-compose.yml`.

Recommendation: use psql as the main tool while learning, because a command line is often all you have when debugging on a server. Use DBeaver for ER diagrams and quick data browsing.

---

## 2. Reading the output of `\d customers`

`\d <table>` shows a table's full definition in five parts: the header, columns, indexes, check constraints, and "Referenced by". In the header `Table "public.customers"`, `public` is the schema (think of it as a folder). Tables go in `public` unless you create another schema.

### Columns

The five headings: **Column** is the name; **Type** is the data type; **Collation** is the text sort/compare rule (blank means the database default); **Nullable** shows `not null` if the column can't be NULL; **Default** is the value filled in when an insert doesn't supply one.

| Column | Type | Nullable | Default | Meaning |
| --- | --- | --- | --- | --- |
| id | bigint | not null | generated always as identity | Auto-increment ID. `always` means you can't supply it manually — inserting an explicit id raises an error |
| email | text | not null | — | Email, required |
| name | text | not null | — | Name, required |
| city | text | nullable | — | City; the only column allowed to be NULL |
| tier | text | not null | `'basic'::text` | Membership tier. `::text` is cast syntax |
| created_at | timestamptz | not null | `now()` | Signup time. Stored in UTC, displayed in the session's time zone |
| profile | jsonb | not null | `'{}'::jsonb` | Semi-structured data; defaults to an empty JSON object |

In Postgres, `text` (no length limit) is the usual string type; there's rarely a reason to use `varchar(n)`.

### Indexes

- `customers_pkey PRIMARY KEY, btree (id)`: `id` is the primary key — not null and unique. Postgres automatically creates a B-tree index for it, so lookups by `id` are fast.
- `customers_email_key UNIQUE CONSTRAINT, btree (email)`: no two customers can share an email. A unique constraint is also enforced by a B-tree index, so lookups by `email` are fast too.

B-tree indexes support `=`, `<`, `>`, `BETWEEN`, and `ORDER BY`.

### Check constraints

`customers_tier_check CHECK (tier = ANY (ARRAY['basic'::text, 'silver'::text, 'gold'::text]))` means `tier` must be one of those three values. The table was created with `IN ('basic', 'silver', 'gold')`; Postgres stores it internally as `= ANY (ARRAY[...])`. The two are equivalent. Inserting `'vip'` is rejected.

### Referenced by

The `customer_id` column in `orders` has a foreign key, `orders_customer_id_fkey`, pointing to `customers.id`. Two effects:

- When inserting an order, `customer_id` must be an existing customer id.
- A customer who still has orders can't be deleted directly.

The same foreign key appears under "Foreign-key constraints" in `\d orders`.

### Worth remembering

- **Constraint names are auto-generated** as `table_column_suffix`: `_pkey` primary key, `_key` unique, `_check` check, `_fkey` foreign key. When an error names a constraint, you immediately know which rule blocked you.
- **`city` and `created_at` have no index**, so filtering on them requires a full table scan. They make good before/after examples when practicing `EXPLAIN`.

---

## 3. Executing statements in psql

SQL statements end with a semicolon `;` — press Enter after it to execute. Backslash meta-commands (`\d`, `\dt`, `\x`) don't need a semicolon; Enter runs them.

### What went wrong

I typed `select title, rental_rate AS price from film limit 5` without a semicolon, then typed `ls` and `\i`:

1. Without the semicolon, psql assumed the statement was unfinished, and the prompt changed from `pagila=#` to `pagila-#`.
2. `ls` isn't a psql command, so it was appended to the query buffer as SQL text. The buffer became `select ... limit 5 ls`, and typing `;` now would give a syntax error.
3. `\i` runs a SQL file and requires a filename, hence `\i: missing required argument`.

Fix: type `\r` (or press Ctrl+C) to clear the buffer. Once the prompt is back to `=#`, re-enter the statement with a semicolon.

### Reading the prompt

| Prompt | Meaning | What to do |
| --- | --- | --- |
| `pagila=#` | Ready for a new statement | — |
| `pagila-#` | Previous statement not finished, usually a missing semicolon | If the buffer is clean, type `;` alone; if junk got in, `\r` |
| `pagila(#` | Unclosed parenthesis | Add `)` or `\r` |
| `pagila'#` | Unclosed single quote | Add `'` or Ctrl+C |

### Useful commands

| Command | Effect |
| --- | --- |
| `;` or `\g` | Execute the statement in the buffer |
| `\r` | Clear the query buffer |
| `\i joins.sql` | Run a SQL file |
| `\! ls`, `\! pwd` | Run a shell command from inside psql |
| `\cd ~/pg-learn` | Change psql's working directory |

### Which machine does `\i` read from?

- If you entered via `docker exec -it pg-learn psql ...`, psql runs **inside the container**. `\i` and `\! ls` see the container's filesystem, not files on your Mac.
- To run a file from your Mac, either:
    - exit psql and run `docker exec -i pg-learn psql -U app -d pagila < joins.sql` in the Mac terminal, or
    - connect with a locally installed psql (e.g. from Homebrew `libpq`): `psql -h localhost -U app -d pagila`. Then `\i joins.sql` reads from the Mac's current directory.

---

## 4. ORDER BY ties and pagination

Whenever you paginate with `LIMIT` / `OFFSET`, the last column in `ORDER BY` must uniquely identify a row. Otherwise rows can be duplicated or skipped across pages.

### What I saw

Both queries fetch page 2 (rows 11–20):

```sql
select title, length from film order by length desc, title limit 10 offset 10;
select title, length from film order by length desc        limit 10 offset 10;
```

Both returned the same 10 films, but in the second query films with equal length came back in a jumbled order. `order by length desc` only guarantees descending length; the order among rows with the same length is undefined by the SQL standard.

### Why this causes bugs

The data near the page boundary:

| length | Films | Which page |
| --- | --- | --- |
| 185 | 10 | Exactly fills page 1 |
| 184 | 8 | Page 2 |
| 183 | 5 | 2 on page 2, 3 on page 3 |

Without a tiebreaker, *which* 2 of the five 183-minute films land on page 2 is undefined. Page 3 is a separate query execution and may pick one of the same films again. A user paging through would see some films twice and others never.

### The fix

```sql
select film_id, title, length
from film
order by length desc, film_id
limit 10 offset 10;
```

Using `title` as the tiebreaker happens to work on this data (there are currently no duplicate titles), but `title` has no unique constraint, so inserting a film with a duplicate title would bring the problem back. Only the primary key `film_id` is guaranteed unique.

---

## 5. NULL vs. empty string: COALESCE and NULLIF

`COALESCE` only replaces NULL, not the empty string `''`. To replace both, first turn empty strings into NULL with `NULLIF(x, '')`.

### Example

```sql
SELECT address_id, address,
       address2 IS NULL                        AS is_null,
       COALESCE(address2, '(none)')            AS a,
       COALESCE(NULLIF(address2, ''), '(none)') AS b
FROM address ORDER BY address_id LIMIT 6;
```

| address_id | address | is_null | a | b |
| --- | --- | --- | --- | --- |
| 1 | 47 MySakila Drive | t | (none) | (none) |
| 2 | 28 MySQL Boulevard | t | (none) | (none) |
| 3 | 23 Workhaven Lane | t | (none) | (none) |
| 4 | 1411 Lillydale Drive | t | (none) | (none) |
| 5 | 1913 Hanoi Way | f | *(blank)* | (none) |
| 6 | 1121 Loja Avenue | f | *(blank)* | (none) |

- **Column a**: rows 1–4 are NULL and get replaced. Rows 5–6 are `''`, returned as-is, so they look blank.
- **Column b**: `NULLIF` first turns `''` into NULL, then `COALESCE` replaces it, so all 6 rows show `(none)`.
- Across the whole `address` table (603 rows), only 4 rows are NULL (the two store and two staff addresses) and 599 are `''`. Approach **a** replaces 4 rows; approach **b** replaces all 603.

### Why COALESCE only looks at NULL

`COALESCE` is defined as "return the first argument that is not NULL" — it only performs an `IS NULL` test. `COALESCE(x, y)` is equivalent to:

```sql
CASE WHEN x IS NOT NULL THEN x ELSE y END
```

And an empty string is not NULL:

- **NULL**: no value, or the value is unknown — e.g. "this field was never filled in".
- **`''`**: a real value, a string of length zero — e.g. "we checked, and there is no second address line".

```sql
SELECT '' IS NULL;            -- f
SELECT NULL IS NULL;          -- t
SELECT length('');            -- 0
SELECT length(NULL);          -- NULL
SELECT NULL = NULL;           -- NULL (not true)
SELECT COALESCE(0, 99);       -- 0
SELECT COALESCE(false, true); -- false
SELECT COALESCE('', 'x');     -- '' (empty string)
SELECT COALESCE(NULL, 'x');   -- x
```

This differs from Python's `a or b`, where `''`, `0`, and `False` are all falsy. SQL's `COALESCE` only cares whether a value is NULL. (Oracle treats `''` as NULL, but PostgreSQL, MySQL, and SQL Server all distinguish them.)

### Two things to watch for

- **psql displays both NULL and `''` as blank**, so you can't tell them apart by eye. Run `\pset null '∅'` to make NULL display as `∅`, or add an `IS NULL` column as above.
- **This comes up constantly in data cleaning.** Different sources represent "missing" inconsistently. In data pipelines it's common to normalize with `NULLIF(x, '')` first, then process.

---

## 6. LIKE vs. ILIKE

`LIKE` is case-sensitive; `ILIKE` is case-insensitive (the I stands for *insensitive*). Otherwise the rules are identical.

Pagila's film titles are all uppercase:

```sql
SELECT count(*) FROM film WHERE title LIKE  '%love%';  -- 0
SELECT count(*) FROM film WHERE title LIKE  '%LOVE%';  -- 10
SELECT count(*) FROM film WHERE title ILIKE '%love%';  -- 10
```

### Wildcards

| Pattern | Meaning |
| --- | --- |
| `%` | Any number of characters, including zero |
| `_` | Exactly one character |
| `'a%'` | Starts with a |
| `'%man'` | Ends with man |
| `'_ce%'` | 2nd and 3rd characters are ce |
| `'%\_%'` | Contains a literal underscore (backslash escape) |

### Notes

- **`ILIKE` is PostgreSQL-specific.** The portable equivalent is `lower(title) LIKE '%love%'`.
- **NULL never matches**: `NULL LIKE '%'` is NULL, not true.
- **Indexes**: a prefix match like `LIKE 'ABC%'` can use a suitable B-tree index. `ILIKE` and leading-wildcard patterns like `'%abc%'` can't use a plain B-tree and fall back to a full scan. To speed them up, use a GIN index with the `pg_trgm` extension, or an expression index on `lower(title)`.
- **`~~` and `~~*` in `EXPLAIN` output** are the internal operator names for `LIKE` and `ILIKE`.

---

## Quick reference

1. SQL in psql ends with `;`. A `-#` prompt means the statement isn't finished; clear a messed-up buffer with `\r`.
2. Run shell commands inside psql as `\! ls`. psql inside a container can't see files on the host.
3. `\d <table>` shows the structure. Constraint suffixes `_pkey`, `_key`, `_check`, `_fkey` mean primary key, unique, check, foreign key.
4. When paginating, end `ORDER BY` with a unique column (usually the primary key).
5. `COALESCE` only handles NULL; convert empty strings first with `NULLIF(x, '')`. Use `\pset null '∅'` to tell them apart.
6. `ILIKE` is case-insensitive but can't use a plain B-tree index.
