# Beam Troubleshooting Guide

## Table of Contents
1. [Type Error Patterns](#type-errors)
2. [Runtime / SQL Errors](#runtime)
3. [Performance Issues](#performance)
4. [Nullable / Maybe Pitfalls](#nullable)
5. [Schema Definition Mistakes](#schema)

---

## 1. Type Error Patterns {#type-errors}

### "Couldn't match type 'Bool' with 'SqlBool'"

**Cause**: Mixing `Bool`-context functions with `SqlBool`-producing operators.

```haskell
-- WRONG
leftJoin_ (...) (\p -> _postAuthor p ==?. pk user)
-- ==?. returns SqlBool, leftJoin_ expects Bool

-- RIGHT
leftJoin_' (...) (\p -> _postAuthor p ==?. pk user)
-- leftJoin_' accepts SqlBool
```

**Full mapping**:
| Bool version | SqlBool version |
|---|---|
| `guard_` | `guard_'` |
| `filter_` | `filter_'` |
| `leftJoin_` | `leftJoin_'` |
| `join_` | `join_'` |
| `relatedBy_` | `relatedBy_'` |
| `==.` | `==?.` |
| `/=.` | `/=?.` |
| `&&.` | `&&?.` |
| `\|\|.` | `\|\|?.` |
| `not_` | `sqlNot_` |

### "Ambiguous type variable 'a0' arising from use of 'countAll_'"

**Cause**: GHC can't infer the return type of the aggregate.

```haskell
-- WRONG
aggregate_ (\u -> (group_ (_userName u), countAll_)) ...

-- RIGHT: annotate with as_
aggregate_ (\u -> (group_ (_userName u), as_ @Int32 countAll_)) ...
-- or
aggregate_ (\u -> (group_ (_userName u), as_ @Int64 countAll_)) ...
```

### "No instance for (Beamable (PrimaryKey FooT))"

**Cause**: Missing `instance Beamable (PrimaryKey FooT)`.

```haskell
-- You need BOTH:
instance Beamable FooT
instance Beamable (PrimaryKey FooT)   -- often forgotten!
```

### "Couldn't match ... QNested s ... with ... s ..."

**Cause**: A QExpr from an outer scope is being used inside a `subselect_` or
similar scope boundary.

```haskell
-- WRONG: user from outer scope used inside subselect
user <- all_ (_blogUsers blogDb)
(email, count) <- subselect_ $
  aggregate_ (\(e, _) -> (group_ e, countAll_)) $ do
    u <- all_ (_blogUsers blogDb)
    guard_ (u `references_` user)   -- ERROR: user escapes scope
    ...

-- RIGHT: filter after subselect using guard_
user <- all_ (_blogUsers blogDb)
(email, count) <- subselect_ $ aggregate_ ... $ all_ (_blogUsers blogDb)
guard_ (email `references_` user)   -- filter in outer scope
```

### "Could not deduce (HasTableEquality be (PrimaryKey FooT))"

**Cause**: The PrimaryKey type doesn't have an equality instance for the
backend. Usually happens with composite PKs that aren't properly defined.

Check that:
1. `PrimaryKey FooT` derives `Generic`
2. `instance Beamable (PrimaryKey FooT)` exists
3. All fields in the PK are types that beam can compare

### Type error in `references_` usage

```haskell
-- references_ signature:
-- references_ :: PrimaryKey t (QGenExpr ...) -> t (QGenExpr ...) -> QGenExpr ... Bool

-- WRONG: passing whole table when PK expected
guard_ (_postAuthor post `references_` pk user)  -- pk user gives PK, not table

-- CORRECT:
guard_ (_postAuthor post `references_` user)
-- _postAuthor post :: PrimaryKey UserT (QExpr ...)
-- user             :: UserT (QExpr ...)
-- references_ compares the FK against the table's PK
```

---

## 2. Runtime / SQL Errors {#runtime}

### SqlError: duplicate key / unique constraint

```haskell
-- Catch postgresql-simple SqlError:
import Database.PostgreSQL.Simple (SqlError(..))
import Control.Exception (catch)

result `catch` \(err :: SqlError) -> do
  putStrLn $ "SQL error: " <> show err
```

### Seeing the generated SQL

Always use debug mode during development:
```haskell
runBeamPostgresDebug putStrLn conn $ ...
-- prints SQL and parameters to stdout
```

Or build the SQL without running:
```haskell
import Database.Beam.Backend.SQL.BeamExtensions
-- dumpSqlSelect :: ... -- check hackage for current API
```

### Wrong column names / table names

If queries run but return wrong data or fail with "column not found":
1. Check `defaultDbSettings` vs your actual DB schema — beam derives names from
   Haskell field names (strips `_`, converts camelCase to snake_case)
2. Use `fieldNamed` to override:
   ```haskell
   blogDb = defaultDbSettings `withDbModification` dbModification
     { _blogUsers = modifyTableFields tableModification
         { _userEmail = fieldNamed "email_address" }
     }
   ```
3. Use `runBeamPostgresDebug` to see what SQL is actually generated.

---

## 3. Performance Issues {#performance}

### Seq scan instead of index scan on JOIN

**Cause**: Using `==.` or wrapping condition in `isTrue_` in ON clause.

```haskell
-- SLOW: generates ON (IS TRUE (p.author_id = u.id))
leftJoin_ (...) (\p -> isTrue_ (_postAuthor p ==?. pk user))

-- FAST: generates ON p.author_id = u.id
leftJoin_' (...) (\p -> _postAuthor p ==?. pk user)
```

### Seq scan on filtered JOIN (monadic style)

```haskell
-- SLOW: CROSS JOIN + WHERE
do
  u <- all_ (_blogUsers blogDb)
  p <- all_ (_blogPosts blogDb)
  guard_ (_postAuthor p `references_` u)

-- FAST: explicit INNER JOIN
do
  u <- all_ (_blogUsers blogDb)
  p <- join_' (_blogPosts blogDb) (\p -> _postAuthor p ==?. pk u)
```

### N+1 query problem

Beam queries are single SQL statements — you won't get N+1 from joins. But if
you're calling `runBeamPostgres` inside a `mapM`, you will:

```haskell
-- BAD: N+1
users <- runBeamPostgres conn $ runSelectReturningList $ select $ all_ (_blogUsers blogDb)
postsPerUser <- forM users $ \user ->
  runBeamPostgres conn $ runSelectReturningList $ select $
    filter_ (\p -> _postAuthor p `references_` user) $ all_ (_blogPosts blogDb)

-- GOOD: single join query
usersAndPosts <- runBeamPostgres conn $ runSelectReturningList $ select $ do
  user <- all_ (_blogUsers blogDb)
  post <- join_' (_blogPosts blogDb) (\p -> _postAuthor p ==?. pk user)
  pure (user, post)
```

---

## 4. Nullable / Maybe Pitfalls {#nullable}

### Double Maybe after LEFT JOIN through nullable FK

When a table with a nullable FK is introduced via `leftJoin_`, beam wraps the
already-Maybe column in another Maybe:

```haskell
-- ShippingInfoT has nullable FK: _orderShipping :: PrimaryKey ShippingInfoT (Nullable f)
-- After leftJoin_, _orderShipping becomes: Maybe (Maybe (PrimaryKey ShippingInfoT Identity))
```

This is expected beam behavior. Handle with nested `maybe_`:
```haskell
maybe_ (val_ 0) (\order ->
  maybe_ (val_ 0) (\shippingId -> shippingId) (_orderShipping order))
  mOrder
```

### NULL comparison with ==. generates CASE WHEN

```haskell
-- This generates: CASE WHEN x IS NULL AND y IS NULL THEN TRUE
--                     WHEN x IS NULL OR y IS NULL THEN FALSE
--                     ELSE x = y END
-- (correct Haskell semantics but slow)
_userBio user ==. val_ Nothing

-- Better: use isNothing_ / isJust_
isNothing_ (_userBio user)
isJust_    (_userBio user)
```

### Comparing nullable FK with justRef / nothingRef

```haskell
-- For Nullable FK field:
data CommentT f = Comment
  { _commentParent :: PrimaryKey CommentT (Nullable f)
  }

-- In query:
guard_ (_commentParent comment ==. nothingRef)    -- IS NULL check
guard_ (_commentParent comment ==. justRef (pk parent))  -- equality with Just
```

---

## 5. Schema Definition Mistakes {#schema}

### Missing `deriving Generic` on PrimaryKey

```haskell
instance Table UserT where
  data PrimaryKey UserT f = UserId (Columnar f Int64) deriving Generic  -- MUST have Generic
  primaryKey = UserId . _userId
```

### Using `Auto` (removed in newer beam)

`Auto` was removed. Use `SqlSerial` from beam-postgres:

```haskell
-- OLD (doesn't work in beam >= 0.9)
data UserT f = User { _userId :: Columnar f (Auto Int64) }

-- NEW
data UserT f = User { _userId :: Columnar f (SqlSerial Int64) }
-- or just use Int64 and `default_` / `insertExpressions`
```

### Column naming: embedded FKs get double underscore

```haskell
data PostT f = Post
  { _postAuthor :: PrimaryKey UserT f }
  -- _postAuthor contains UserId (Columnar f Int64)
  -- Generated column name: "post_author__id"  (note double underscore)
  -- Rename with: fieldNamed "author_id"
```