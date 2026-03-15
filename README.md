# Beam

[![Build status](https://github.com/haskell-beam/beam/workflows/Build/badge.svg)](https://github.com/haskell-beam/beam/actions)
[![Hackage](https://img.shields.io/hackage/v/beam-core.svg)](https://hackage.haskell.org/package/beam-core)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A type-safe, non-TH Haskell relational database library and ORM.

> If you use beam commercially, please consider a [donation](https://liberapay.com/tathougies) to support the project.

---

## Overview

Beam is a Haskell interface to relational databases that uses the type system to verify queries are correct before they ever reach the database. Queries are written in a natural monadic style and cover SQL92 as well as significant portions of SQL99, SQL2003, and SQL2008.

Key design principles:

- **Type-safe** — invalid queries are caught at compile time
- **No Template Haskell** — schema definitions use standard Haskell generics
- **Modular** — core functionality is separate from backend-specific extensions
- **Non-opinionated** — no connection or transaction management; use the backend's native library

---

## Packages

| Package | Version | Description |
|---|---|---|
| `beam-core` | 0.9.0.0 | Core query and schema DSL |
| `beam-postgres` | 0.5.0.0 | PostgreSQL backend (via `postgresql-simple`) |
| `beam-sqlite` | 0.5.0.0 | SQLite backend (via `sqlite-simple`) |
| `beam-migrate` | 0.5.0.0 | Schema migration support |
| `beam-migrate-cli` | — | CLI tool for running migrations |

Third-party backends:
- [`beam-mysql`](https://github.com/tathougies/beam-mysql)
- [`beam-firebird`](https://github.com/gibranrosa/beam-firebird)

---

## Installation

Add beam to your `cabal` or `stack` project:

```cabal
-- cabal
build-depends:
    beam-core    >= 0.9,
    beam-postgres >= 0.5
```

```yaml
# stack
extra-deps:
  - beam-core-0.9.0.0
  - beam-postgres-0.5.0.0
```

---

## Quick Start

### Define a schema

```haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE OverloadedStrings #-}

import Database.Beam
import GHC.Generics

data UserT f = User
  { _userId    :: Columnar f Int
  , _userEmail :: Columnar f Text
  , _userName  :: Columnar f Text
  } deriving (Generic, Beamable)

type User = UserT Identity
deriving instance Show User

instance Table UserT where
  data PrimaryKey UserT f = UserId (Columnar f Int) deriving (Generic, Beamable)
  primaryKey = UserId . _userId

data AppDb f = AppDb
  { appUsers :: f (TableEntity UserT)
  } deriving (Generic, Database be)

appDb :: DatabaseSettings be AppDb
appDb = defaultDbSettings
```

### Query

```haskell
import Database.Beam.Postgres

-- Fetch all users
getAllUsers :: Pg [User]
getAllUsers = runSelectReturningList $ select $ all_ (appUsers appDb)

-- Lookup by ID
getUserById :: Int -> Pg (Maybe User)
getUserById uid = runSelectReturningOne $ select $
  filter_ (\u -> _userId u ==. val_ uid) $
  all_ (appUsers appDb)
```

### Insert

```haskell
insertUser :: Pg ()
insertUser = runInsert $ insert (appUsers appDb) $
  insertValues [ User 1 "alice@example.com" "Alice" ]
```

---

## Backends

### beam-postgres

Backed by [`postgresql-simple`](https://hackage.haskell.org/package/postgresql-simple). Connection and transaction management is handled directly via `postgresql-simple` — beam just handles query construction and result mapping.

```haskell
import Database.Beam.Postgres
import Database.PostgreSQL.Simple

main :: IO ()
main = do
  conn <- connect defaultConnectInfo { connectDatabase = "mydb" }
  users <- runBeamPostgres conn getAllUsers
  print users
```

### beam-sqlite

Backed by [`sqlite-simple`](https://hackage.haskell.org/package/sqlite-simple).

```haskell
import Database.Beam.Sqlite
import Database.SQLite.Simple

main :: IO ()
main = do
  conn <- open "mydb.sqlite"
  users <- runBeamSqlite conn getAllUsers
  print users
```

---

## Migrations

`beam-migrate` provides schema migration tooling with type-level schema tracking. Use `beam-migrate-cli` to apply migrations from the command line.

See the [migrations guide](https://haskell-beam.github.io/beam/schema-guide/migrations/) for details.

---

## Testing

`beam-core` has in-depth unit tests covering query generation against an idealized ANSI SQL backend. The `beam-postgres` and `beam-sqlite` backends are tested through the documentation — every code example is executed against a live database when building docs.

Run core tests:

```bash
cabal test beam-core
```

---

## Documentation

Full user guide: https://haskell-beam.github.io/beam

### Building docs locally

```bash
# Using the provided script (requires Python + mkdocs)
./build-docs.sh

# Or with Nix
nix develop
./build-docs.sh
```

To build docs for a single backend only, edit `mkdocs.yml` and set `enabled_backends`:

```yaml
- docs.markdown.beam_query:
    enabled_backends:
      - beam-sqlite
```

---

## Contributing

Contributions are welcome. Please open an issue or pull request on [GitHub](https://github.com/haskell-beam/beam).

See [CONTRIBUTORS](./CONTRIBUTORS) for the list of contributors.

---

## Community

- Mailing list: [beam-discussion on Google Groups](https://groups.google.com/forum/#!forum/beam-discussion)
- IRC: `#haskell-beam` on Libera.Chat
- Compatibility matrix: [haskell-beam.github.io/beam/about/compatibility](https://haskell-beam.github.io/beam/about/compatibility/)

---

## License

MIT — see [LICENSE](./beam-core/LICENSE)
