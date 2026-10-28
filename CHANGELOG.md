## Unreleased

Requires iyi 0.19.1: a `Database` shared by tasks on every worker needs
`spawn_parallel` (0.19.0), and the `@[Share]` defs and `SharedProc` that check
the blocks it keeps (0.19.1).

* A `Database` and its `Pool` are `Share`, so one kept in a constant may be
  used by parallel tasks on every worker, with no `Share` error and no
  warning from the compiler (db#1). v0.3.0's pool kept its bookkeeping with
  no lock, and two tasks on different workers were handed one connection:
  one read the other's rows, or the runtime panicked with "two fibers
  reading one fd". The bookkeeping is behind a `Mutex` now, and a
  connection goes in and out of it in a slot that is `Share` on trust.
* A checkout that finds the pool full parks, and its worker runs other
  tasks. A release hands its resource, or the room it left, to the checkout
  that has waited longest, which a later checkout no longer takes first.
* `Pool#close` and `Database#close` close the idle resources, and each one a
  task holds when the task releases it, where they closed it under the task.
* **Breaking:** the blocks a pool or database keeps run on whichever worker
  needs them, several at once, so the compiler checks each where it is
  written, as it checks a parallel task's: `Pool.new`'s, `Database.new`'s
  and `Database.open`'s factory, and `setup_connection`'s block, may capture
  only `Share` values. A block that counts in a captured local, or keeps
  what it made in a captured array, is refused: count in an `Atomic`, keep
  it behind a `Mutex`.
* `Db.open` calls the driver's `ConnectionBuilder#build` from any worker, on
  trust: a builder holds what the URI said and only reads it.
* `Pool#stats` reads atomics, and takes no lock.
* A pool knows a resource given back by its `object_id`: a resource is a
  class's instance.
* The driver registry is behind a lock, so `Db.open` may run on any worker.
* **Breaking:** `Statement.def_around_query_or_exec`'s hooks run on every
  worker, so a hook may capture only `Share` values. A statement learns
  there are none, the common case, from an atomic count, without a lock.
* **Breaking:** `Db::Core::LOG` is a `SharedLogger`: std's logger for "db",
  which is not `Share`, behind a lock, with its level in an atomic beside
  it, so a statement with debug off takes no lock. Its level is read off
  std's logger when the first statement asks; set it after that with
  `Db::Core::LOG.level =`. A backend is set on std's `Log.for("db")`.
* The dummy driver (`db/testing/dummy_driver`) may be used from every
  worker: its counts are atomics, and what it keeps for the tests is behind
  a lock.
* `db/parallel_test.iyi` runs db#1's program, twenty times, and the pool's
  other cases on twelve workers; `manual/pool_concurrency.iyi` runs its
  tasks with `spawn_parallel`.

## v0.3.0

* `Database#close` and `Pool#close` close every resource. They closed every
  other one, as crystal-db's do, and left the rest open.
* A factory that panics no longer leaves the pool counting a resource in
  flight.
* A waiting checkout is woken when a resource leaves the pool, so it can
  build a new one, and waits `checkout_timeout` in all, however often it is
  woken. A `checkout_timeout` or `retry_delay` too long for a timer waits as
  long as one can, where it panicked.
* A closed pool answers "the pool is closed" to every checkout, the waiting
  ones too, where it built a new resource.
* `Database#setup_connection` takes each idle connection out of the pool
  while it sets it up, so a statement the setup runs does not release it,
  and closes the ones the setup fails on.
* The statement cache builds a new statement in place of one its user
  closed.
* `query` with a block, `query_one`, `query_one?`, `query_all`,
  `query_each`, `scalar` and `Db.connect` with a block answer the error
  closing what they opened, unless they answer an error already, as
  crystal-db raised it from its `ensure`.
* A `transaction` block that answers `Rollback` answers the error of a
  rollback that fails, where it answered `nil`.

## v0.2.0

Requires iyi 0.18.0, whose fixes for iyilang/iyi#121 to #128 this port no
longer works around.

* The entry point is the package's root module, as crystal-db's is `DB`:
  `import db` and `Db.open`, `Db.connect`, `Db.register_driver`,
  `Db.driver_class` and `Db::VERSION`, where they were `import db/db/db`
  and `Db::Db.open`. The other modules are where they were.
* `Db.open` and `Db.connect` take a `URI | String`, as crystal-db's do.
* `Db.register_driver` takes a `Driver.class`.
* A driver's `read` overload for a nilable type can be written `def read(t
  : Int32?.class)`.
* The "Executing query" debug entry carries the query and its arguments as
  the entry's `query` and `args` data, as crystal-db's does, where they were
  written into the message.

## v0.1.1

The modules move from `iyi_db/` to `db/`, so their namespace is `Db::...`
when the package is required. A package's modules are named under the
package's name, `db`, and v0.1.0's `iyi_db/core` became `Db::IyiDb::Core`
in a program that required it: the names `derive db_serializable`,
`mapping` and the README wrote (`::IyiDb::...`, `IyiDb::Db.open`) did not
resolve there.

* `IyiDb::Db.open` is `Db::Db.open`; `import db/iyi_db/core::*` is
  `import db/db/core::*`.

## v0.1.0

The first release of iyi-db: crystal-db 0.15.0 ported to iyi, in iyi, with no
Crystal code left.

* `IyiDb::Db.open`, `.connect` and `.register_driver` (module `iyi_db/db`) are
  crystal-db's `DB.open`, `DB.connect` and `DB.register_driver`.
* Every fallible call answers `T | DBError`; crystal-db's exception classes are
  the `DBError` hierarchy in `iyi_db/errors`.
* Query arguments and read values are `DBValue`: the scalars, `Time`, `Span`,
  `Bytes`, arrays of them, and `CustomValue` for a driver's own types.
* Several columns are read as a `Tuple(...)` or `NamedTuple(...)` type, in
  `ResultSet#read` and in `as:`.
* `derive db_serializable`, `derive db_serializable_non_strict` and `@[DBField]`
  replace `include DB::Serializable`, `DB::Serializable::NonStrict` and
  `@[DB::Field]`; `mapping` is `DB.mapping`.
* A `transaction` block left early rolls back; answering `Rollback.new` rolls
  back and answers `nil`.
* `Statement.def_around_query_or_exec` is a hook registry, and
  `CustomValue#to_log` replaces `DB::MetadataValueConverter`.
* `Pool#prefill` and `Database.open` open the initial connections; the
  constructors open nothing.
* `DriverSpecs` (crystal-db's `db/spec`) runs the shared driver specs.
* `Connection::Options.from_http_params` reads the `prepared_statements_cache`
  default from `prepared_statements_cache`, not `prepared_statements`.

### Notes for driver implementors

* Driver methods answer their errors: `ConnectionBuilder#build`,
  `Connection#build_prepared_statement`, `#build_unprepared_statement`,
  `Statement#perform_query`, `#perform_exec`, `ResultSet#move_next` and `#read`
  return `... | DBError`, and `do_close` returns `Nil | DBError`.
* A transaction subclass implements `start`, `perform_commit` and
  `perform_rollback`; `Connection#create_transaction` and
  `TopLevelTransaction#create_save_point_transaction(parent, name)` build the
  driver's own transaction types without doing IO.
* A driver's own value types inherit `CustomValue`.

crystal-db's history before the port is in its
[CHANGELOG](https://github.com/crystal-lang/crystal-db/blob/master/CHANGELOG.md).
