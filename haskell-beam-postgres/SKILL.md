---
name: haskell-beam-postgres
description: Expert knowledge for writing correct Haskell Beam queries against PostgreSQL. Use this skill any time the user asks about beam, beam-postgres, beam-core, beam queries, beam schemas, beam migrations, beam table definitions, beam relationships, beam aggregations, beam window functions, or any Haskell database code using the Beam library. Also use whenever the user mentions DatabaseEntity, Columnar, QExpr, runBeamPostgres, runSelectReturningList, all_, filter_, guard_, aggregate_, leftJoin_, related_, references_, insertReturning, beam-migrate, or any other beam DSL combinator. Always consult this skill before writing any Beam code — even for "simple" queries, as many subtle pitfalls exist around SqlBool vs Bool, nullable fields, and join performance that require expert handling.
---

# Haskell Beam + PostgreSQL Expert Skill

You are an expert in `beam-core` and `beam-postgres`. Your goal is to produce
**correct, compiling, performant** Beam queries every time. When in doubt about
an API detail, use the fetch tool to look it up before writing code.

---

## Quick Reference — Authoritative Sources

Always fetch from these when you need precise types, signatures, or new APIs:

| Resource | URL |
|---|---|
| beam-core Hackage | `https://hackage.haskell.org/package/beam-core` |
| beam-postgres Hackage | `https://hackage.haskell.org/package/beam-postgres` |
| Beam user guide | `https://haskell-beam.github.io/beam/` |
| Beam tutorial 1 (schema, insert, select) | `https://haskell-beam.github.io/beam/tutorials/tutorial1/` |
| Beam tutorial 2 (joins, lenses) | `https://haskell-beam.github.io/beam/tutorials/tutorial2/` |
| Beam tutorial 3 (aggregates, subqueries) | `https://haskell-beam.github.io/beam/tutorials/tutorial3/` |
| Expressions guide | `https://haskell-beam.github.io/beam/user-guide/expressions/` |
| Relationships guide | `https://haskell-beam.github.io/beam/user-guide/queries/relationships/` |
| beam-postgres backend guide | `https://haskell-beam.github.io/beam/user-guide/backends/beam-postgres/` |
| beam-core Database.Beam.Query docs | `https://hackage.haskell.org/package/beam-core-0.10.4.0/docs/Database-Beam-Query.html` |
| beam-postgres Database.Beam.Postgres docs | `https://hackage.haskell.org/package/beam-postgres/docs/Database-Beam-Postgres.html` |

**If the user's question involves an API you are not 100% certain about, fetch
the relevant URL above before writing code.** This is especially important for:
- Postgres-specific functions (`pgArrayAgg`, `pgSelectWith`, `pgNubBy_`, locking)
- Window functions and CTEs
- beam-migrate APIs
- Any function added after beam-core 0.9

---

## Required GHC Extensions

```haskell
{-# LANGUAGE DeriveGeneric          #-}
{-# LANGUAGE FlexibleInstances      #-}
{-# LANGUAGE MultiParamTypeClasses  #-}
{-# LANGUAGE OverloadedStrings      #-}
{-# LANGUAGE StandaloneDeriving     #-}
{-# LANGUAGE TypeFamilies           #-}
{-# LANGUAGE TypeSynonymInstances   #-}
-- Optional but common:
{-# LANGUAGE GADTs                  #-}
{-# LANGUAGE ScopedTypeVariables    #-}
```

---

## Table & Schema Definition Pattern

```haskell
import Database.Beam
import Database.Beam.Postgres
import GHC.Generics (Generic)

-- Convention: data type ends in 'T', constructor drops 'T'
data UserT f = User
  { _userId        :: Columnar f Int64          -- NOT NULL
  , _userName      :: Columnar f Text
  , _userEmail     :: Columnar f Text
  , _userBio       :: Columnar f (Maybe Text)   -- NULLable
  } deriving (Generic)

type User = UserT Identity
type UserId = PrimaryKey UserT Identity

instance Beamable UserT

instance Table UserT where
  data PrimaryKey UserT f = UserId (Columnar f Int64) deriving (Generic)
  primaryKey = UserId . _userId

instance Beamable (PrimaryKey UserT)

-- Foreign key reference — embed PrimaryKey of referenced table
data PostT f = Post
  { _postId      :: Columnar f Int64
  , _postTitle   :: Columnar f Text
  , _postAuthor  :: PrimaryKey UserT f          -- FK → users
  } deriving (Generic)

instance Beamable PostT
instance Table PostT where
  data PrimaryKey PostT f = PostId (Columnar f Int64) deriving (Generic)
  primaryKey = PostId . _postId
instance Beamable (PrimaryKey PostT)

-- NULLABLE FK: use Nullable to wrap the functor
data CommentT f = Comment
  { _commentId     :: Columnar f Int64
  , _commentParent :: PrimaryKey CommentT (Nullable f)  -- optional self-ref
  } deriving (Generic)
instance Beamable CommentT
instance Table CommentT where
  data PrimaryKey CommentT f = CommentId (Columnar f Int64) deriving (Generic)
  primaryKey = CommentId . _commentId
instance Beamable (PrimaryKey CommentT)

-- Database descriptor
data BlogDb f = BlogDb
  { _blogUsers    :: f (TableEntity UserT)
  , _blogPosts    :: f (TableEntity PostT)
  , _blogComments :: f (TableEntity CommentT)
  } deriving (Generic)

instance Database Postgres BlogDb

blogDb :: DatabaseSettings Postgres BlogDb
blogDb = defaultDbSettings `withDbModification`
  dbModification
    { _blogUsers = modifyTableFields tableModification
        { _userEmail = fieldNamed "email_address" }   -- override column name
    }
```

### Column naming rules
- Beam derives SQL column names from Haskell field names by stripping the
  leading underscore and converting camelCase to snake_case.
- Embedded FK columns are prefixed with the field name: `_postAuthor` with
  `UserId (Columnar f Int64)` generates column `post_author__id`.
- Override with `fieldNamed` inside `modifyTableFields`.

---

## Lenses (tableLenses)

```haskell
-- Derive lenses without Template Haskell
User (LensFor userId) (LensFor userName) (LensFor userEmail) (LensFor userBio)
  = tableLenses

-- Usage in queries:
filter_ (\u -> u ^. userEmail ==. val_ "alice@example.com") $
  all_ (_blogUsers blogDb)
```

---

## Running Queries (beam-postgres)

```haskell
import Database.Beam.Postgres (runBeamPostgres, runBeamPostgresDebug)

-- Production
result <- runBeamPostgres conn $ do
  runSelectReturningList $ select query

-- Debug (logs SQL to stdout)
result <- runBeamPostgresDebug putStrLn conn $ do
  runSelectReturningList $ select query

-- Transactions: use postgresql-simple directly
import qualified Database.PostgreSQL.Simple as Pg
Pg.withTransaction conn $ do
  runBeamPostgres conn action1
  runBeamPostgres conn action2
```

---

## SELECT Queries

```haskell
-- All rows
all_ (_blogUsers blogDb)

-- Filter
filter_ (\u -> _userName u ==. val_ "Alice") $
  all_ (_blogUsers blogDb)

-- Limit / offset
limit_ 10 . offset_ 20 $ all_ (_blogUsers blogDb)

-- Order
orderBy_ (\u -> desc_ (_userName u)) $ all_ (_blogUsers blogDb)

-- Compound ordering
orderBy_ (\u -> (asc_ (_userName u), desc_ (_userId u))) $ ...

-- Distinct (standard)
nub_ $ all_ (_blogUsers blogDb)

-- Run
users <- runBeamPostgres conn $ runSelectReturningList $ select $
  filter_ (\u -> _userName u ==. val_ "Alice") $
  all_ (_blogUsers blogDb)
```

---

## Joins — CRITICAL CORRECTNESS RULES

See `references/joins.md` for full examples. Key rules:

### Inner join (monadic — generates cross join + WHERE)
```haskell
do
  user <- all_ (_blogUsers blogDb)
  post <- all_ (_blogPosts blogDb)
  guard_ (_postAuthor post `references_` user)
  pure (user, post)
```
> ⚠️ Monadic style generates `CROSS JOIN ... WHERE`. Postgres may not use
> indexes. For large tables use explicit join functions below.

### Inner join (explicit — generates proper JOIN ... ON)
```haskell
do
  user <- all_ (_blogUsers blogDb)
  post <- join_' (_blogPosts blogDb)
            (\p -> _postAuthor p ==?. pk user)
  pure (user, post)
```

### LEFT JOIN — use `leftJoin_'` with `SqlBool` for index-friendly SQL
```haskell
do
  user  <- all_ (_blogUsers blogDb)
  mPost <- leftJoin_' (all_ (_blogPosts blogDb))
              (\p -> _postAuthor p ==?. pk user)
  pure (user, mPost)
-- mPost is Nullable: access fields with ^. or pattern match on Maybe
```

### related_ (shorthand for FK join)
```haskell
do
  post   <- all_ (_blogPosts blogDb)
  author <- related_ (_blogUsers blogDb) (_postAuthor post)
  pure (post, author)
```

---

## Bool vs SqlBool — THE MOST COMMON SOURCE OF BUGS

| Situation | Use |
|---|---|
| Normal filter on non-nullable | `==.`, `guard_`, `filter_` |
| Filter involving nullable column | `==?.`, `guard_'`, `filter_'` |
| LEFT JOIN condition | **Always** `leftJoin_'` + `==?.` |
| FK equality in join ON clause | `==?.` or `references_` |

```haskell
-- WRONG — wraps comparison in IS TRUE, breaks Postgres index usage
leftJoin_ (...) (\p -> isTrue_ (_postAuthor p ==?. pk user))

-- RIGHT — passes SqlBool directly, Postgres can use indexes
leftJoin_' (...) (\p -> _postAuthor p ==?. pk user)

-- Converting SqlBool to Bool when needed
isTrue_    :: QExpr be s SqlBool -> QExpr be s Bool
isFalse_   :: QExpr be s SqlBool -> QExpr be s Bool
isUnknown_ :: QExpr be s SqlBool -> QExpr be s Bool
unknownAs_ :: Bool -> QExpr be s SqlBool -> QExpr be s Bool
sqlBool_   :: QExpr be s Bool -> QExpr be s SqlBool
```

---

## Aggregations

```haskell
-- COUNT(*) grouped by name
aggregate_ (\u -> (group_ (_userName u), as_ @Int32 countAll_)) $
  all_ (_blogUsers blogDb)

-- SUM with maybe handling (after LEFT JOIN)
aggregate_ (\(user, post) ->
    ( group_ user
    , sum_ (maybe_ 0 (\p -> _postLikes p) post)
    )) $ do
  user  <- all_ (_blogUsers blogDb)
  mPost <- leftJoin_' (all_ (_blogPosts blogDb))
              (\p -> _postAuthor p ==?. pk user)
  pure (user, mPost)

-- Common aggregate functions
countAll_  :: QAgg be s Int32         -- COUNT(*)
count_     :: ... -> QAgg be s Int32  -- COUNT(expr)
sum_       :: Num a => QExpr be s a -> QAgg be s (Maybe a)
avg_       :: ...
min_       :: ...
max_       :: ...
```

---

## DML — INSERT / UPDATE / DELETE

```haskell
-- Insert fixed values
runInsert $ insert (_blogUsers blogDb) $
  insertValues [User 1 "Alice" "alice@example.com" Nothing]

-- Insert with DB-generated PK (default_)
runInsert $ insert (_blogUsers blogDb) $
  insertExpressions [User default_ (val_ "Alice") (val_ "alice@example.com") nothing_]

-- INSERT … RETURNING (Postgres-specific)
newUsers <- runInsertReturningList $
  insertReturning (_blogUsers blogDb) $
    insertExpressions [User default_ (val_ "Bob") (val_ "bob@example.com") nothing_]

-- UPDATE
runUpdate $ update (_blogUsers blogDb)
  (\u -> _userBio u <-. val_ (Just "Hello"))
  (\u -> _userId u ==. val_ 42)

-- UPDATE by saving a whole record
runUpdate $ save (_blogUsers blogDb) updatedUser

-- DELETE
runDelete $ delete (_blogUsers blogDb)
  (\u -> _userId u ==. val_ 42)
```

---

## Postgres-Specific Features

For full details, always fetch:
`https://haskell-beam.github.io/beam/user-guide/backends/beam-postgres/`

### DISTINCT ON
```haskell
import Database.Beam.Postgres (pgNubBy_)

pgNubBy_ (\u -> _userEmail u) $ all_ (_blogUsers blogDb)
```

### Full-Text Search
```haskell
import Database.Beam.Postgres

filter_ (\doc ->
    toTsVector (Just english) (doc ^. docContent)
      @@ toTsQuery (Just english) (val_ "haskell & beam"))
  $ all_ (myDb ^. myDocs)
```

### Row Locking
```haskell
import qualified Database.Beam.Postgres as Pg

Pg.lockingAllTablesFor_ Pg.PgSelectLockingStrengthUpdate Nothing $
  filter_ (\u -> _userId u ==. val_ 1) $ all_ (_blogUsers blogDb)
```

### CTEs / `WITH` (pgSelectWith)
```haskell
import Database.Beam.Postgres (pgSelectWith)

-- pgSelectWith allows CTEs to appear within joins (unlike selectWith)
result <- pgSelectWith $ do
  activePosts <- selecting $
    filter_ (\p -> _postPublished p ==. val_ True) $
    all_ (_blogPosts blogDb)
  pure $ do
    post <- reuse activePosts
    pure post
```

### pgArrayAgg
```haskell
import Database.Beam.Postgres (pgArrayAgg)

aggregate_ (\(user, post) ->
    (group_ (_userId user), pgArrayAgg (_postTitle post)))
  $ do ...
```

---

## Subqueries

```haskell
-- Use subselect_ to scope a subquery
result <- runSelectReturningList $ select $ do
  user <- all_ (_blogUsers blogDb)
  (email, postCount) <- subselect_ $
    aggregate_ (\(email, post) -> (group_ email, as_ @Int32 countAll_)) $ do
      u <- all_ (_blogUsers blogDb)
      p <- leftJoin_' (all_ (_blogPosts blogDb))
              (\p -> _postAuthor p ==?. pk u)
      pure (pk u, p)
  guard_ (email `references_` user)
  pure (user, postCount)
```

---

## Window Functions

```haskell
withWindow_ (\row -> frame_ (partitionBy_ (_postAuthor row)) noOrder_ noBounds_)
  (\row w -> (row, rowNumber_ `over_` w))
  (all_ (_blogPosts blogDb))
```

---

## Nullable Fields — Handling in Queries

```haskell
-- maybe_ is like Haskell's maybe, but for QExpr
maybe_ (val_ 0) (\bio -> charLength_ bio) (_userBio user)

-- isNothing_ / isJust_
filter_ (isNothing_ . _userBio) $ all_ (_blogUsers blogDb)
filter_ (isJust_    . _userBio) $ all_ (_blogUsers blogDb)

-- fromMaybe_ (provide default)
fromMaybe_ (val_ "") (_userBio user)

-- justRef / nothingRef for nullable FK references
guard_ (_commentParent comment ==. nothingRef)
guard_ (_commentParent comment ==. justRef (pk parent))
```

---

## beam-migrate Cheatsheet

```haskell
import Database.Beam.Migrate

-- Checked database settings carry migration info
blogDbChecked :: CheckedDatabaseSettings Postgres BlogDb
blogDbChecked = defaultMigratableDbSettings

-- Create all tables
createSchema :: Migration Postgres ()
createSchema = do
  users    <- createTable "users"    (User notNull notNull notNull noConstraint)
  posts    <- createTable "posts"    (Post notNull notNull (UserId notNull))
  pure ()
```

---

## Common Compile Errors & Fixes

| Error message fragment | Likely cause | Fix |
|---|---|---|
| `Couldn't match type 'Bool' with 'SqlBool'` | Using `==.` where `==?.` needed | Switch to `==?.` and `leftJoin_'` / `guard_'` |
| `Ambiguous type variable` | Missing `as_ @TypeName` on aggregate | Add `as_ @Int32` or appropriate type annotation |
| `No instance for Beamable` | Missing `instance Beamable` or `deriving Generic` | Add both |
| `Could not deduce (HasTableEquality ...)` | FK type mismatch | Ensure FK field type matches PrimaryKey definition exactly |
| `Couldn't match ... QNested` | Cross-scope QExpr use | Use `subselect_` to scope the inner query |
| `references_` type error | Wrong FK lens | Use `pk table` to get PrimaryKey, pass correct lens |

---

## Using the Fetch Tool for Unknown APIs

When you encounter an unfamiliar beam function, Postgres-specific combinator,
or newer API (beam-core ≥ 0.10), **always fetch the Hackage docs or beam
user guide URL** listed in the Quick Reference table at the top before writing
code. Prefer:
1. The official beam documentation site
2. The Hackage module docs
3. The beam GitHub source (`https://github.com/haskell-beam/beam`)

Do not guess type signatures. Beam's type errors are notoriously cryptic, so
getting the signatures right the first time is essential.

---

## Detailed Reference Files

For more complex scenarios, read these reference files as needed:

- `references/joins.md` — Complete join patterns with generated SQL examples
- `references/postgres-extensions.md` — All Postgres-specific beam-postgres features
- `references/migrations.md` — beam-migrate full workflow
- `references/troubleshooting.md` — Expanded error diagnosis guide