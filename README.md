# ormbaga

**Table ORM + versioned migrations** for Baga, on top of
[`../pgbaga`](../pgbaga/README.md). One `OrmDb` type, three backends:
**PostgreSQL** and **boilaDB** over the PG wire, and **rocksbaga** (KV, RESP2)
over [`../drockb`](../drockb/README.md) — `ORM_BACKEND=drock`. The KV path keeps
the same table API and the same `OrmQuery` accessors, so an app written against
Postgres reads rows identically; only raw SQL does not translate (see
[`docs/gaps.md`](docs/gaps.md) G12).

This repository is the package. The compiler, `std`, `pgbaga`, and
`boilabaga` stay in the baga language monorepo. Check this tree out as
`app-product/ormbaga` there (git submodule) so path deps and
`-I app-product` keep working.

## Checkout

Inside a baga language clone:

```bash
git submodule update --init --recursive
# or, first time from a fresh baga tree without the submodule recorded:
git clone git@github.com:bagalang/ormbaga.git app-product/ormbaga
```

`sandak.toml` keeps path deps (`../../std`, `../pgbaga`, `../boilabaga`).
`fmrbaga`, `oauthbaga`, `apps/api`, and `apps/registry` still depend on
this path. Tests (`orm_test`, `orm_boila_test`) stay in baga.

Architecture: [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Layers

| Path | Role |
|------|------|
| `sql/sql.baga` | pure quote/escape/SQL builders + `$1` placeholders |
| `session/db.baga` | `OrmDb` + backend избор (PG wire или drock) |
| `session/result.baga` | `OrmQuery` / `OrmExec` / `OrmCount` + accessor-ите |
| `session/orm.baga` | table CRUD + transactions, разклонени по backend |
| `session/kv*.baga` | table слойът върху rocksbaga: ключове, запис, sort, JOIN, транзакция, миграции |
| `migrate/migrate.baga` | migration registry + `up` / `down` / `status` |
| `migrate/schema.baga` | **sample only** migrations (Postgres + boila) — apps own schema |
| `pool/pool.baga` | shared session pool (sync/batch) |
| `examples/demo.baga` | migrate + CRUD demo |

Root `*.baga` files re-export these paths for stable imports.

## Migrations

```sql
CREATE TABLE IF NOT EXISTS baga_schema_migrations (
  version    BIGINT PRIMARY KEY,
  name       TEXT NOT NULL,
  applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```baga
let mut set = migrate_set_new()
set = migrate_add(set, 20260803001, "create_users", up_sql, down_sql)
// Migration { version, name, up, down } — one Vec, not four
// …
let r = migrate_up(db, set)?          // pending only; each in a transaction.
                                      // duplicate-object (42710/42P07/42701)
                                      // records the version and continues
                                      // (DDL that committed without history)
db = migrate_db(r)                    // always rebind (by-value structs)
let n = migrate_applied_n(r)          // how many versions ran this call
let s = migrate_up_steps(db, set, 2)? // staged: at most 2 pending (0 = all)
let d = migrate_down(db, set, 1)?     // rollback last N
db = migrate_db(d)

let cur = migrate_current(db)?          // highest applied version (0 = empty)
db = orm_db_c(cur)
let pend = migrate_pending(db, set)?    // count + next version (0/0 = at head)
db = pend.db
let st = migrate_status_text(db, set)?  // "ver name [applied|pending]" lines
db = st.db                              // report keeps your session
```

Registry is **embedded** (parallel `Vec`s — Baga has no `Vec<struct>`).
Duplicate versions in a set are rejected before any SQL runs; applied
versions are fetched once per `migrate_up` and checked in memory.
**Fail-loud:** a pending version below the highest applied one (set edited
after deploy) aborts `migrate_up` as out-of-order instead of corrupting
history.

## ORM API

```baga
let db = orm_connect_env()?   // PG* env, default baga_orm / bagatest
// or boilaDB (PG wire, boilabaga defaults :6575):
// let db = orm_connect_boila_env()?
// or ORM_BACKEND=boila|pg:
// let db = orm_connect_auto_env()?
// or wrap any PgConn:  let db = orm_from_conn(pg_or_boila_conn)

let cols = vec_new(); vec_push(cols, "email"); vec_push(cols, "name")
let vals = vec_new(); vec_push(vals, "a@b.c"); vec_push(vals, "Ada")

// INSERT … RETURNING * — id in one round-trip
let ins = orm_insert_returning_strs(db, "users", cols, vals)?
db = orm_db_q(ins)
let id = orm_cell_by_i64(ins, 0, "id")

let q = orm_where_eq_str(db, "users", "email", "a@b.c")?
db = orm_db_q(q)

let f = orm_find_i64(db, "users", "id", id)?
let page = orm_all_page(db, "users", "id", 0, 20, 0)?  // order, limit, offset
let u = orm_update_by_id(db, "users", id, set_cols, set_lits)?
let d = orm_delete_by_id(db, "users", id)?
let c = orm_count(db, "users")?
let p = orm_ping(db)?

// NULL-capable paths (nullable columns):
// orm_insert_full / orm_insert_returning_full(db, table, cols, vals, nulls)
// orm_update_by_id_full(db, table, id, set_cols, set_vals, set_nulls)
// nulls[i] == 1 binds SQL NULL; length mismatch is a fail-loud error.
```

Always rebind `db = result.db` (by-value structs — same as pgbaga).

Raw escape hatch: `orm_exec` / `orm_query` / `orm_begin` / `orm_commit` / `orm_rollback`.

## Setup (Postgres)

```bash
psql -h 127.0.0.1 -U postgres -c "CREATE USER bagatest PASSWORD 'pas+123';"
psql -h 127.0.0.1 -U postgres -c "CREATE DATABASE baga_orm OWNER bagatest;"
psql -h 127.0.0.1 -U postgres -d baga_orm -c "GRANT ALL ON SCHEMA public TO bagatest;"
```

Defaults: `PGHOST=127.0.0.1` `PGUSER=bagatest` `PGPASSWORD=pas+123` `PGDATABASE=baga_orm`.

## Setup (boilaDB)

```bash
# terminal 1
./baga -I . -I app-product app-product/boilaDB/tools/serve_pg.baga
# terminal 2 — ORM over PG wire
./baga -I . -I app-product tests/orm_boila_test.baga
./baga -I . -I app-product app-product/ormbaga/examples/demo_boila.baga
```

Use `ormbaga_boila_migrations()` (BoilaSQL: explicit `id`, no SERIAL/FK) or
your own set that stays inside the boila subset. See boilabaga dialect notes.

Env: `ORM_BACKEND=boila` makes `orm_connect_auto_env` / `orm_pool_from_auto_env`
target boilaDB; default is Postgres (`PG*`).

## Setup (rocksbaga)

```bash
# terminal 1 — KV сървърът (RESP2, без автентикация)
cd ../..   # корена на baga монорепото
LSMPATH=/tmp/ormkv LSMPORT=16579 ./baga -I . -I app-product app-product/rocksbaga/tools/serve.baga

# terminal 2
export ORM_BACKEND=drock DROCK_HOST=127.0.0.1 DROCK_PORT=16579
./baga -I . -I app-product tests/orm_drock_test.baga
```

`orm_connect_drock_env()` взима адреса от `DROCK_HOST`/`DROCK_PORT`; `fmr_run` на
`fmrbaga` прави същото при `ORM_BACKEND=drock`. Редът е JSON под
`kv:t:<table>:<id>`, вторичните индекси са `kv:t:<table>:ix:<col>:<val>`, а PK
идва от `kv:t:<table>:seq`.

## Run

```bash
./baga app-product/ormbaga/demo.baga
./baga tests/orm_test.baga              # real Postgres
./baga tests/orm_boila_test.baga        # boilaDB PG wire
```

## Parameterized queries (default)

`orm_find_*`, `orm_where_eq_*`, `orm_insert_strs`, `orm_insert_returning_strs`,
`orm_insert_full`, `orm_update_by_id`, `orm_update_by_id_full`,
`orm_delete_by_id` use **Extended Query** (`$1..$n`) via
pgbaga — values are bound, not concatenated.

Legacy `orm_*_lit` / `sql_lit` paths remain for migrations and trusted SQL.

## Pool + prepared statements

```baga
let pool = orm_pool_from_env(4)?            // Postgres
// let pool = orm_pool_from_boila_env(4)?   // boilaDB
// let pool = orm_pool_from_auto_env(4)?    // ORM_BACKEND=
let lease = orm_pool_acquire(pool)?
// ... use lease.db ...
orm_pool_release(lease)?
orm_pool_close(pool)?

orm_prepare(db, "user_by_id", "SELECT * FROM users WHERE id=$1", 1)?
orm_query_prepared_str(db, "user_by_id", ids)?
```

Broken pooled sessions (server restarts, idle kill) **reconnect on acquire** —
the pool keeps its connect params and self-heals before checkout.

HTTP servers should prefer **`FMR_WORKERS=N`** (one DB per worker).

## Honesty

- Prefer parameterized APIs; avoid hand-building SQL with user data.
- No model codegen — rows are dynamic cells (`orm_cell_by`).
- No associations API yet (`has_many` is a `where` on FK).
- No auto-migrate — prefer explicit versioned migrations.
- boilaDB has no SERIAL/FK/DEFAULT — use `ormbaga_boila_migrations` or own subset.

## Path to framework

```
fmrbaga routes → ormbaga session → pgbaga → PostgreSQL
                     │               ↑ migrations at boot
                     └→ boilabaga → pgbaga → boilaDB :6575
```

## License

[MIT](LICENSE) — Copyright (c) 2026 Dim Gigov.
