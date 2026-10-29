# iyi-db

The common database interface for [iyi](https://iyi-lang.com): a port of
[crystal-db](https://github.com/crystal-lang/crystal-db) 0.15.0, written in iyi
with no Crystal code and no Crystal standard library underneath. A driver for a
specific database is needed to access one.

It keeps crystal-db's shape - `open` a URI, a pooled `Database`, `exec`,
`scalar` and `query` with positional or `args:` arguments, `ResultSet#read`,
nested transactions, a connection pool configured from the URI, object
mapping, and the shared specs a driver runs - and changes what iyi's rules
require: errors are values, types are closed, and tasks are structured.

## Requirements

iyi 0.19.1 or later: a `Database` shared by tasks on every worker needs
`spawn_parallel` (0.19.0), and the `@[Share]` defs and `SharedProc` that
check the blocks it keeps (0.19.1).

## Installation

iyi-db is an iyi module. Add it to a project under the short name `db`:

```sh
iyi get github.com/iyilang/db --as db
```

which writes this line to `iyi.mod`:

```
require github.com/iyilang/db v0.3.0 as db
```

Files then `import db` for the root module, `Db.open` and the driver
registry, and its other modules as `db/db/...`: the short name, then the
module's path in the package.

## Usage

```iyi
module app

import db
import db/db/core::*
import db/db/errors::{DBError}
import db/db/value::{DBValue}
import sqlite3/sqlite3 # a driver: importing it registers "sqlite3"

def contacts : Nil | DBError
  Db.open("sqlite3:./file.db") do |db|
    db.exec("create table contacts (name text, age integer)")!
    db.exec("insert into contacts values (?, ?)", "John Doe", 30)!

    args = [] of DBValue
    args << "Sarah"
    args << 33
    db.exec("insert into contacts values (?, ?)", args: args)!

    puts "max age:"
    puts db.scalar("select max(age) from contacts")! # => 33

    puts "contacts:"
    db.query("select name, age from contacts order by age desc") do |rs|
      puts "#{rs.column_name(0)} (#{rs.column_name(1)})"
      # => name (age)
      rs.each do
        puts "#{rs.read(String)!} (#{rs.read(Int32)!})"
        # => Sarah (33)
        # => John Doe (30)
      end!
    end!
  end
end

case contacts
in Nil     then puts "done"
in DBError then puts "failed: #{it.message}"
end
```

The example uses SQLite, where `?` marks an argument; PostgreSQL writes `$1`,
`$2`, ... iyi-db does not interpret the statements.

Several columns are read as a tuple or named tuple *type*:
`rs.read(Tuple(String, Int32))`, `db.query_all(sql, as: NamedTuple(name:
String, age: Int32))`.

The pool is configured from the URI's query string: `initial_pool_size`
(default 1), `max_pool_size` (0, unlimited), `max_idle_pool_size` (1),
`checkout_timeout` (5.0 seconds), `retry_attempts` (1) and `retry_delay` (0.2
seconds). `prepared_statements` (true) and `prepared_statements_cache` (true)
choose how statements run.

### On every worker

A `Database` is `Share`, so one opened once and kept in a constant serves
parallel tasks (`spawn_parallel`) on every worker, a web server's handlers
among them, with no `Share` error and no warning from the compiler:

```iyi
DATABASE = Db.open(ENV["DATABASE_URL"]).or_panic

def ask(n : Int64) : Int64
  DATABASE.query_one("SELECT $1::int8", n, as: Int64).or_panic
end

group do |g|
  32.times { |t| g.spawn_parallel { ask(t.to_i64); nil } }
end
```

The pool hands each connection to one task at a time, decides which one
under a `Mutex`, and parks a checkout that finds none free - the task, not
its worker - until a release hands it one. The blocks a database keeps run
on whichever worker opens a connection or runs a statement, several at
once, so the compiler checks each where it is written as it checks a
parallel task's: the factory given to `Pool.new`, `Database.new` and
`Database.open`, `setup_connection`'s block and
`Statement.def_around_query_or_exec`'s hooks may capture only `Share`
values. Count in an `Atomic`, keep things behind a `Mutex`.

`close` closes the idle connections at once, and each one a task holds
when the task releases it.

### Reading rows into objects

```iyi
import db/db/serializable::{db_serializable, DBField}

class Employee
  property title : String

  @[DBField(key: "firstname")]
  property name : String?

  derive db_serializable
end

employees = db.query_all("SELECT title, firstname FROM employees", as: Employee)!
employees[0].title # => "Manager"
```

`derive db_serializable_non_strict` passes over columns the type has no
property for. `Employee.from_db_row(rs)` reads the current row and
`Employee.from_rs(rs)` every row. `mapping({...})`, crystal-db's deprecated
`DB.mapping`, is there for code written against it.

### Logging and instrumentation

Every statement logs `Executing query` at debug level to the std/log logger
`"db"`, with the query and its arguments as the entry's data:
`Executing query -- query="select name from contacts where age > ?" args=[18]`.
`Db::Core::LOG` is that logger behind a lock, with its level in an atomic
beside it, so a statement with debug off takes no lock. Its level is read
off std's logger when the first statement asks, and set after that with
`Db::Core::LOG.level = Severity::Debug`; a level given to std's loggers
directly once it has been read is not seen. A backend is set on std's
`Log.for("db")`.
`Statement.def_around_query_or_exec { |stmt, args, run| ...; run.call; ... }`
wraps every query a driver runs.

## Modules

| module | what it holds |
|---|---|
| `db` | `Db.open`, `.connect`, `.register_driver`, `.driver_class`, `VERSION` |
| `db/core` | `Driver`, `ConnectionBuilder`, `Database`, `Connection`, `Statement`, `ResultSet`, transactions, `PoolStatement`s, the `QueryMethods`/`SessionMethods`/`StatementMethods`/`BeginTransaction`/`Disposable`/`ConnectionContext` traits, `LOG` and its `SharedLogger` |
| `db/errors` | `DBError` and its subclasses |
| `db/value` | `DBValue`, `CustomValue`, `Any`, `ExecResult` |
| `db/pooling` | `Pool(T)`, usable for any resource with a `close` |
| `db/query_params` | `fetch_bool`, `fetch_int`, `fetch_float` for URI options |
| `db/serializable` | `derive db_serializable`, `DBField`, `mapping` |
| `db/spec` | `DriverSpecs`, and `describe`/`it`/`expect_error` for tests |

## What is different from crystal-db

Each difference follows from a rule of iyi (SPEC.md); the API is otherwise
crystal-db's.

| crystal-db | iyi-db | why |
|---|---|---|
| `DB.open(uri)` raises | `Db.open(uri) : Database \| DBError` - every fallible call answers its error, `!` hands it on | III.1: errors are values |
| `DB::Error`, `DB::MappingException`, ... | `DBError`, `MappingError`, ... - the same hierarchy as classes implementing `Error`; `ArgumentError` is a `DBError` too | III.1 |
| `rescue`/`ensure` around `close` | `close : Nil \| DBError`, and `defer` | III.1.4 |
| `raise DB::Rollback.new` in a `transaction` block | the block answers `Rollback.new` (the transaction answers `nil`); answering any other error rolls back and answers it | no exceptions |
| `return` from a `transaction` block commits | leaving the block early (`return`, `break`, `!`, a panic) rolls back; `tx.commit` first to commit | `!` is a `return`: the two cannot be told apart, and rolling back cannot lose data |
| `DB::Any` arguments, anything else by driver overloads | `DBValue`: the scalars, `Time`, `Span`, `Bytes`, arrays of them, and `CustomValue`, which a driver's own value classes inherit | closed types (R-3), and an artifact cannot name its importers' types (R-1) |
| `rs.read(String, Int32)`, `as: {String, Int32}` | `rs.read(Tuple(String, Int32))`, `as: Tuple(String, Int32)` (and `NamedTuple(...)`) | R-2: a signature has to name what it returns |
| `include DB::Serializable`, `@[DB::Field]`, `DB::Serializable::NonStrict`, `T.new(rs)` | `derive db_serializable`, `@[DBField]`, `derive db_serializable_non_strict`, `T.from_db_row(rs)` | II.4: derives |
| reopening `DB::Statement` with `def_around_query_or_exec` | `Statement.def_around_query_or_exec { \|stmt, args, run\| ... }`, a hook registry; drivers may still override `around_query_or_exec` | R-3: no open classes |
| `DB::MetadataValueConverter.arg_to_log` overloads | `CustomValue#to_log` | R-3 |
| `Pool.new` and `Database.new` open the initial connections | `Pool#prefill` and `Database.open(...)` do; the constructors do no IO (a constructor has no error to answer) | III.1 |
| `TopLevelTransaction.new` runs `BEGIN` | `Transaction#start` runs `BEGIN`/`SAVEPOINT`; `perform_commit`/`perform_rollback` are what a transaction subclass implements | III.1 |
| `Pool` guards itself with a `Mutex` | so does iyi-db's, around its bookkeeping, since tasks run on every worker (`spawn_parallel`); a connection goes in and out of the lock in a slot that is `Share` on trust, as no connection is `Share` | III.4.4, III.4.13 |
| a pool's factory, `setup_connection`'s block and `def_around_query_or_exec`'s hooks may capture anything | they run on whichever worker needs them, so they may capture only `Share` values, checked where they are written (`@[Share]`) | III.4.4: data races are a compile error |
| `disconnect!`-style names | no `!` in names | III.1.7 |
| `DB::DriverSpecs.run do ... end` (implicit receiver) | `DriverSpecs.run do \|specs\| specs.... end` | no `with ... yield` |
| `Connection::Options.from_http_params` read the statement cache default from `prepared_statements` | it reads `prepared_statements_cache` | a crystal-db bug, fixed |
| `Pool#close` closes every other resource, as closing one takes it out of the list being walked; and the ones out, under the tasks using them | it closes every idle one, and each one out when it is released | a crystal-db bug, fixed; a task's connection is not closed under it |
| a checkout waiting on a full pool is woken only when a resource goes idle, and each wake-up waits `checkout_timeout` again | a release hands its resource, or the room it left, to the checkout that has waited longest, and a wait lasts `checkout_timeout` in all | a crystal-db bug, fixed |
| a closed pool builds a new resource for the next checkout | a checkout of a closed pool, a waiting one too, answers "the pool is closed" | a crystal-db bug, fixed |
| the statement cache hands back a statement its user closed | it builds a new one | a crystal-db bug, fixed |

## Writing a driver

Subclass `Driver` (answer a `ConnectionBuilder` from `connection_builder(uri)`),
`ConnectionBuilder`, `Connection`, `Statement` and `ResultSet` from
`db/core`, and call `Db.register_driver("name", MyDriver)` at the
top level of the driver's module, so importing the driver registers it. The
abstract methods say what to implement; the comments on `Driver`,
`Connection`, `Statement` and `ResultSet` say how. A lost connection answers
`ConnectionLost.new(connection)`, so the pool retries on another one. Values
of the driver's own types are classes inheriting `CustomValue`.

A `Database` that `Db.open` makes calls the driver's `ConnectionBuilder#build`
from whichever worker opens a connection, several at once, and takes that on
trust: a builder holds what the URI said, made once, and `build` only reads
it.

Test a driver with the shared specs:

```iyi
import db/db/spec::{DriverSpecs}

DriverSpecs.run do |specs|
  specs.connection_string "sqlite3:./test.db"
  specs.sample_value "hello", "varchar(25)", "'hello'"
  specs.binding_syntax { |index| "?" }
  # ... the other *_syntax blocks
end
```

`db/driver_specs_test.iyi` runs them against an in-memory driver.

## Development

```sh
iyi test db     # every *_test.iyi
```

The tests are crystal-db's specs, ported; `db/testing/dummy_driver.iyi` is
its dummy driver, safe to use from every worker. `db/parallel_test.iyi` runs
db#1's program and the pool's other cases on twelve workers. Every test also
builds from artifacts (`iyi build --emit-iyimod DIR` then `--use-iyimod
DIR`). `manual/` holds the two programs crystal-db runs by hand:
`iyi run manual/pool_concurrency.iyi -- --duration=3`, whose tasks run on
every worker, and `iyi run manual/load.iyi`.

`iyi check` a program, not a module that only queries: a block passed to
`Database#query` is typed through the driver's `Statement`, and a module
with no driver in it has none, so iyi 0.18.0 reports "can't infer block
return type" there. A program that imports a driver checks clean.

## Contributors

- [bcardiff](https://github.com/bcardiff) Brian J. Cardiff - creator and
  maintainer of crystal-db
