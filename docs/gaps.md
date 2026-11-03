# ormbaga — gaps

## G1 — ~~No `Vec<struct>` for migration registry~~ — closed

**Shipped 2026-08-14.** `Migration { version, name, up, down }` in
`MigrationSet.items`. `migrate_add` / `migrate_len` unchanged;
`migrate_at` returns one row.

## G2 — By-value `OrmDb` threading

**Symptom.** Every call returns updated `db`; forget rebind → stale connection state.

**Workaround.** `db = orm_db_q(r)` / `orm_db_exec(r)` / `migrate_db(r)` —
always rebind (documented).

**Verdict.** Same as pgbaga G2.

## G2b — OrmQuery / OrmExec L3 — **shipped (B1)**

**Shipped.** `OrmExec` / `OrmQuery` / `OrmCount` / `MigrateResult` are sum
enums; helpers `orm_ok`, `orm_ok_q`, `orm_nrows`, `migrate_is_ok`, …
Apps and tests use helpers (no `.ok` fields on the enum).

## G3 — ~~No parameterized queries~~ — **CLOSED**

**Was.** ORM built SQL strings only.

**Now.** `orm_find_*` / `orm_where_eq_*` / `orm_insert_*` / update / delete use
pgbaga Extended Query (`$1`). Legacy `*_lit` paths remain for trusted SQL.

**Verdict.** Closed.

## G4 — No reflection / model macros

**Symptom.** Cannot map `struct User { … }` ↔ row automatically.

**Workaround.** Dynamic cells + `orm_cell_by(name)`.

**Verdict.** Acceptable for P0; codegen tool could come later (Prisma-like) as external binary.

## G5 — Migrations only embedded

**Symptom.** No `migrations/*.sql` directory walker in std.

**Workaround.** `schema.baga` registry; can `read_file` fixed paths manually.

**Verdict.** Add file-based loader when `std/os` has readdir (or list paths in set).

## G6 — ~~Multi-line string SQL awkward~~ — **CLOSED**

**Was.** Assumed Baga string literals were single-line.

**Now.** The lexer accepts raw newlines inside `"..."` (no rejection path in
`lex_string`); `migrate_ensure_table` ships multi-line DDL and it round-trips
through Simple Query unchanged. `\n` escapes remain available.

**Verdict.** Closed (2026-08-08).

## G7 — ~~`RETURNING` not used for insert→id~~ — **CLOSED**

**Now.** `orm_insert_returning_strs` / `orm_insert_returning_lits` +
`sql_insert_returning_p`. Covered in `tests/orm_test.baga`.

## G9 — SQL builders nested concat / empty idents — **closed 2026-08-24**

`sql_lit` / `sql_ident` / `sql_ident_q` / `sql_ident_list` / `sql_join`
use `bufbaga`. Empty or NUL names → `""`. `sql_i64` is `int_to_str`.

## Closed / fine

- goose history table + transactional up/down verified in `tests/orm_test.baga`.
- Quote escaping for `O'Hara` / `a'b` verified.
- ON DELETE CASCADE through ORM delete verified.

## Closed recently (2026-08-07)

- `orm_all_page` / `orm_where_eq_*_page` + `sql_page` / `sql_offset`.
- `orm_ping` for readiness checks.
- Package layered under `sql/`, `session/`, `migrate/`, `pool/`.

## G8 — boilaDB dialect vs Postgres sample schema

**Symptom.** Full Postgres sample migrations (`BIGSERIAL`, `REFERENCES`,
`DEFAULT NOW()`, double-quoted idents) fail on boilaDB.

**Workaround.** `orm_connect_boila_env` + `ormbaga_boila_migrations()`;
`sql_ident` emits bare names when safe; history DDL is dual-backend.
Apps that need FK/SERIAL stay on real Postgres.

**Verdict.** Expected; boilabaga documents the subset.

## G10 — Migration hardening round 2 — **closed 2026-08-24**

- **Staged up.** `migrate_up_steps(db, set, steps)` applies at most N
  pending versions (0 = all); `migrate_up` delegates to it.
- **Out-of-order guard.** A pending version below the highest applied one
  aborts `migrate_up` (`out-of-order migration version N`) — sets edited
  after deploy cannot silently reorder history.
- **Pending report.** `migrate_pending(db, set)` returns count + next
  version in one history fetch; readiness checks use `count == 0`.
- **NULL-capable typed paths.** `orm_insert_full` /
  `orm_insert_returning_full` / `orm_update_by_id_full` bind SQL NULL per
  column (`nulls[i] == 1`); length mismatch is fail-loud.
- **Pool self-heal.** `OrmPool` keeps its connect params; `acquire` /
  `try_acquire` reconnect broken sessions before checkout (server
  restarts / idle kills no longer hand out dead connections).

Covered live: `tests/orm_test.baga` (Postgres) and
`tests/orm_boila_test.baga` (serve_pg), 2026-08-24.

## G11 — history INSERT lost after DDL (boila) — **closed 2026-08-26**

**Symptom.** `migrate_up` wraps each version in `BEGIN` / DDL / history
`INSERT` / `COMMIT`. The INSERT used Extended Query (`$1`); the DDL used
Simple Query. On boila the extended DML path does not always see the
simple-query txn, so the index/table commits and the history row does
not. Next boot: `CREATE INDEX` → `42710` and the rest of the set never
runs (bagabuch stopped at `index_accounts_number` with 14 history rows
and 90 objects).

**Fix.** History INSERT/DELETE are Simple Query (same path as DDL).
`42710` / `42P07` / `42701` after UP: ROLLBACK, record the version,
continue. Gate: `tests/orm_boila_test.baga` `dup_index_repaired`.

## Live check (2026-08-13)

`tests/orm_boila_test.baga` срещу `serve_pg` — **36/36** (клиент и
сървър с и без `--rc`). `demo_boila` migrate+CRUD OK. Без peer в
`baga-test` остава FAIL (очакван external-peer). Вж.
`bench/boila/results/orm-boila-2026-08-13.md`.

## G12 — drock (rocksbaga) backend: какво не е SQL

**Symptom.** `ORM_BACKEND=drock` дава същия `OrmDb` и същите accessor-и, но
долу няма сесия на SQL двигател. Не нещата, които layer-ът не крие:

- **64 аргумента на команда.** rocksbaga затваря връзката при по-голям брой, а
  мъртва връзка отравя целия worker (503 до рестарт). `kv_mget` / `kv_mset` /
  `kv_raw_del_many` режат на парчета под `KV_CMD_MAX_ARGS()`.
- **COMMIT не е атомен.** `orm_begin` трупа записите в `OrmDb.tx`
  (`session/kv_tx.baga`), `orm_commit` ги изсипва с `MSET`/`DEL`, `orm_rollback`
  ги хвърля. Читателят в рамките на заявката вижда собствените си редове, но
  процес, умрял по средата на изсипването, може да остави частичен buffer.
  Идва си само от write batch в rocksbaga, не от този слой.
- **Няма UNIQUE, няма CHECK, няма суми.** Уникалността я пази приложението
  (upsert или проверка преди insert); `SUM`/`MAX(col)` се правят в модела с
  `bagadecimal` / `kv_max_value`.
- **WHERE е само равенство.** Индексът е `kv:t:<table>:ix:<col>:<val>` за
  стойности до 120 байта; `!=`, `LIKE`, `BETWEEN`, `<` се филтрират в модела
  след GET-а. `ORDER BY` е сортиране в паметта (`kv_sort.baga`), с numeric
  подравняване за `id`/`*_id`.
- **Граница на заявката.** `orm_guard_txn(db)` (извиква се от `fmrbaga` след
  хендлъра) хвърля буфер на транзакция, която action-ът е оставил отворена —
  worker-ът споделя `db`-то си със следващата заявка. Покрито от
  `tests/orm_drock_test.baga` (`guard_closes_abandoned_txn`).
- **Суров SQL е грешка.** `orm_query_params` / `orm_exec_params` / `orm_exec`
  върху drock връщат ясна грешка и **не пипат `db`** — преди да е пипнал,
  `orm_query_from` подменяше живата връзка на работника с мъртва PG-овска.
- **KV миграциите са отделен регистър.** `KvMigration` + `kv_migrate_run`
  (версия под `kv:mig:<table>`), forward-only и идемпотентни; SQL DDL-ът не
  се изпълнява тук.

**Verdict.** Acceptable за product върху KV: моделите се прехищят към
`kv_select_*`, а границите са документирани, не маскирани.
