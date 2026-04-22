# Known Issues Tested April 2026

This file documents errors found in the skill files when tested against **sqlx 0.8.6** (latest stable as of 2026-04-22). Each issue includes a link to the file where it occurs, the error message, a minimal reproduction, and the fix.

---

## Issue 1 — `pool.transaction()` does not exist

| Field        | Value                                                                                 |
| ------------ | ------------------------------------------------------------------------------------- |
| **Location** | `SKILL.md` (Core Patterns > Transactions) and `references/transactions-migrations.md` |
| **Severity** | Compile error                                                                         |

### Problem

Both files show calling `.transaction()` directly on a `PgPool`:

```rust
let result = pool.transaction::<_, _, sqlx::Error>(|tx| {
    Box::pin(async move {
        sqlx::query!("INSERT INTO users (name) VALUES ($1)", "Bob")
            .execute(&mut **tx).await?;
        Ok(())
    })
}).await?;
```

This fails to compile with:

```
error[E0599]: no method named `transaction` found for struct `Pool<DB>` in the current scope
```

`.transaction()` is defined on the `Connection` trait, not on `Pool`. You must acquire a connection first.

### Reproduction

```rust
// Fails
let result = pool.transaction::<_, _, sqlx::Error>(|tx| {
    Box::pin(async move { Ok(()) })
}).await?;
```

### Fix

Use `pool.acquire().await?` to get a `PoolConnection`, then call `.transaction()` on it:

```rust
use sqlx::{Connection, PgPool};

let mut conn = pool.acquire().await?;
conn.transaction::<_, _, sqlx::Error>(|tx| {
    Box::pin(async move {
        sqlx::query!("INSERT INTO users (name) VALUES ($1)", "Bob")
            .execute(&mut **tx).await?;
        Ok(())
    })
}).await?;
```

Alternatively, for manual transactions use `pool.begin().await?` which returns a `Transaction`.

---

## Issue 2 — Wrong compile-time type-override syntax for PostgreSQL/SQLite

| Field        | Value                                                            |
| ------------ | ---------------------------------------------------------------- |
| **Location** | `references/compile-time-checking.md` (Type Overrides in Macros) |
| **Severity** | Compile error                                                    |

### Problem

The skill claims the override syntax is:

```rust
"SELECT id, created_at as `created_at: chrono::DateTime<chrono::Utc>` FROM users WHERE id = $1"
```

This uses backticks, which is the **MySQL** syntax. For **PostgreSQL** and **SQLite** the correct syntax uses double quotes:

```rust
r#"SELECT id, created_at as "created_at: chrono::DateTime<chrono::Utc>" FROM users WHERE id = $1"#
```

### Reproduction

```rust
let record = sqlx::query!(
    "SELECT id, created_at as `created_at: chrono::DateTime<chrono::Utc>` FROM users WHERE id = $1",
    1i64
)
.fetch_one(&pool)
.await?;
```

This produces a SQL syntax error from the Postgres backend during macro expansion.

### Fix

Use double-quoted identifiers in Postgres / SQLite, backticks in MySQL:

```rust
// PostgreSQL / SQLite
let record = sqlx::query!(
    r#"SELECT id, created_at as "created_at: chrono::DateTime<chrono::Utc>" FROM users WHERE id = $1"#,
    1i64
)
.fetch_one(&pool)
.await?;

// MySQL
let record = sqlx::query!(
    "SELECT id, created_at as `created_at: chrono::DateTime<chrono::Utc>` FROM users WHERE id = ?",
    1i64
)
.fetch_one(&pool)
.await?;
```

---

## Issue 3 — `PgConnectOptions::set()` does not exist

| Field        | Value                                                           |
| ------------ | --------------------------------------------------------------- |
| **Location** | `references/connection-pooling.md` (PostgreSQL Connect Options) |
| **Severity** | Compile error                                                   |

### Problem

The skill shows:

```rust
let options = PgConnectOptions::new()
    ...
    .set("search_path", "my_schema,public")
    .connect();
```

`PgConnectOptions` has no method named `set`. The correct method for arbitrary key/value session parameters is `options(...)` (taking an iterable of key-value pairs).

### Reproduction

```rust
let options = PgConnectOptions::new()
    .set("search_path", "my_schema,public")
    .connect();
```

Compile error:

```
error[E0599]: no method named `set` found for struct `PgConnectOptions`
```

### Fix

Use `.options(...)` instead:

```rust
let options = PgConnectOptions::new()
    .host("localhost")
    .port(5432)
    .username("user")
    .password("password")
    .database("mydb")
    .ssl_mode(PgSslMode::Prefer)
    .application_name("my-app")
    .options([("search_path", "my_schema,public")]);

let pool = PgPoolOptions::new()
    .connect_with(options)
    .await?;
```

---

## Issue 4 — PostgreSQL COPY IN API requires trait import and uses `.send()`, not `.write()`

| Field        | Value                                                |
| ------------ | ---------------------------------------------------- |
| **Location** | `references/database-specifics.md` (PostgreSQL COPY) |
| **Severity** | Compile error                                        |

### Problem

The skill shows:

```rust
let mut writer = pool.copy_in_raw("COPY users FROM STDIN WITH (FORMAT csv)").await?;
writer.write(b"1,Alice,alice@example.com\n").await?;
writer.write(b"2,Bob,bob@example.com\n").await?;
let rows = writer.finish().await?;
```

Two issues:

1. `copy_in_raw` is defined on the `PgPoolCopyExt` trait, which must be imported.
2. The method on `PgCopyIn` is `send()`, not `write()`.
3. `send()` takes `impl Deref<Target = [u8]>`. Passing a byte-string literal (`b"..."`) produces a `&[u8; N]` which does **not** deref to `[u8]` — you must cast to `&[u8]`.

### Reproduction

```rust
let mut writer = pool.copy_in_raw("COPY users (id,name,email) FROM STDIN WITH (FORMAT csv)").await?;
writer.write(b"1,Alice,alice@example.com\n").await?;
```

Compile errors:

```
error[E0599]: no method named `copy_in_raw` found for struct `Pool<Postgres>`
error[E0599]: no method named `write` found for struct `PgCopyIn<C>`
```

### Fix

```rust
use sqlx::postgres::PgPoolCopyExt;

let mut writer = pool
    .copy_in_raw("COPY users (id, name, email) FROM STDIN WITH (FORMAT csv)")
    .await?;

writer.send(b"1,Alice,alice@example.com\n" as &[u8]).await?;
writer.send(b"2,Bob,bob@example.com\n" as &[u8]).await?;

let rows = writer.finish().await?;
```

---

## Issue 5 — `row.get()` requires `sqlx::Row` trait in scope

| Field        | Value                                                             |
| ------------ | ----------------------------------------------------------------- |
| **Location** | `SKILL.md` (Runtime Query Functions) and multiple reference files |
| **Severity** | Compile error                                                     |

### Problem

The skill repeatedly shows:

```rust
let id: i64 = row.get("id");
```

without mentioning that `sqlx::Row` must be in scope. `get()` is a trait method on `sqlx::Row`, not an inherent method on the row structs.

### Reproduction

```rust
let row = sqlx::query("SELECT id, name FROM users WHERE id = $1")
    .bind(1)
    .fetch_one(&pool)
    .await?;
let id: i64 = row.get("id");
```

Compile error:

```
error[E0599]: no method named `get` found for struct `PgRow`
```

### Fix

Add `use sqlx::Row;` at the top of the file:

```rust
use sqlx::{Row, query};

let row = sqlx::query("SELECT id, name FROM users WHERE id = $1")
    .bind(1)
    .fetch_one(&pool)
    .await?;
let id: i64 = row.get("id");
```

---

## Issue 6 — Nested transaction `tx.begin()` requires trait import

| Field        | Value                                                                 |
| ------------ | --------------------------------------------------------------------- |
| **Location** | `SKILL.md` (Transactions) and `references/transactions-migrations.md` |
| **Severity** | Compile error                                                         |

### Problem

The skill shows:

```rust
let mut tx = pool.begin().await?;
let mut nested = tx.begin().await?;
```

`Transaction::begin()` comes from the `Acquire` trait. Without importing `Acquire`, the compiler sees only the trait's associated function, not a method, leading to:

```
error[E0599]: no method named `begin` found for struct `Transaction<'_, Sqlite>`
```

### Reproduction

```rust
let mut tx = pool.begin().await?;
let mut nested = tx.begin().await?; // fails
```

### Fix

Import `Acquire`:

```rust
use sqlx::Acquire;

let mut tx = pool.begin().await?;
let mut nested = tx.begin().await?; // now works
```

---

## Issue 7 — `query_scalar!` aggregate nullability is database-specific

| Field        | Value                                             |
| ------------ | ------------------------------------------------- |
| **Location** | `SKILL.md`, `references/compile-time-checking.md` |
| **Severity** | Compile error on PostgreSQL                       |

### Problem

The skill shows:

```rust
let count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
    .fetch_one(&pool)
    .await?;
```

On **SQLite** this compiles successfully because sqlx infers `COUNT(*)` as non-null. On **PostgreSQL** the macro infers it as `Option<i64>`, producing:

```
error[E0308]: `?` operator has incompatible types
  expected `i64`, found `Option<i64>`
```

### Reproduction

```rust
// On PostgreSQL
let count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
    .fetch_one(&pool)
    .await?;
```

### Fix

Use `Option<i64>` (or use a type override to force non-null):

```rust
let count: i64 = sqlx::query_scalar!("SELECT COUNT(*) as \"count!: i64\" FROM users")
    .fetch_one(&pool)
    .await?;

// Or accept Option
let count: Option<i64> = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
    .fetch_one(&pool)
    .await?;
```

---

## Issue 8 — Missing `PgConnectOptions::connect()` is not async

| Field        | Value                                                           |
| ------------ | --------------------------------------------------------------- |
| **Location** | `references/connection-pooling.md` (PostgreSQL Connect Options) |
| **Severity** | Compile error                                                   |

### Problem

The skill shows:

```rust
let options = PgConnectOptions::new()
    ...
    .connect();

let pool = PgPoolOptions::new()
    .connect_with(options)
    .await?;
```

`PgConnectOptions::new()...connect()` is **not** a method on `PgConnectOptions`. The code should build options first, then pass them to `PgPoolOptions::connect_with(options)` (which itself is async).

### Reproduction

```rust
let options = PgConnectOptions::new()
    .host("localhost")
    .connect(); // <-- does not exist
```

### Fix

Simply remove `.connect()` and pass the options struct to `connect_with(...)`:

```rust
let options = PgConnectOptions::new()
    .host("localhost")
    .port(5432)
    .username("user")
    .password("password")
    .database("mydb");

let pool = PgPoolOptions::new()
    .connect_with(options)
    .await?;
```
