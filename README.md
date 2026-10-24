# mezze-sqlite

The SQLite JDBC driver, as a Mezze package.

```toml
[dependencies]
mezze_sqlite = "github:mezze-lang/mezze-sqlite@v0.1.0"
```

Then `mez sync`, and the standard library's performer works — you import
nothing from this package:

```mezze
use std::effects::db::{ Db, SqliteDb, sql, BindParams, value_at }

let main = {} -> perform SqliteDb { path = "app.db" } in
  Db.run { q = sql { text = "select name from users where id = :id" }
             .bind { params = { id = 7 } } }
```

## Why this exists

`std::effects::db` already has `SqliteDb`. SQLite is the one flavour the
standard library knows a performer for, because an embedded database with no
server to run is worth that.

What the standard library cannot carry is the **driver**, because a JDBC driver
for SQLite is not pure Java — it carries compiled native libraries, 24
platform/arch pairs and 24.5 MB of them. Bundling a subset into the runtime jar
was tried, and it is worse than it looks:

- everyone pays the weight, whether they open a database or not;
- anyone outside the chosen subset — FreeBSD, musl/Alpine, 32-bit, riscv — gets
  a native-library failure instead of a clear missing-driver message;
- `mez build --native` has to **strip** it from the image classpath, and that
  strip is why `Db` silently did not work in a native binary at all.

`__native_db_open` takes any JDBC URL, so a driver is an ordinary package
dependency. That is how `mezze-postgres` works, and there is no reason for
SQLite to be the exception.

## Without this package

`perform SqliteDb { path = … }` compiles. The first operation answers
`Unavailable`, naming the cure:

> no JDBC driver for this URL is on the classpath. The `Db` effect and its mock
> performers ship with the stdlib, but a DRIVER does not — the JDK provides the
> `java.sql` API only. Add the driver package to your Mezze.toml
> `[dependencies]`.

`MockDb`, `MockDbWith` and `FailingDb` need no driver, so a route's data path
stays testable without this.
