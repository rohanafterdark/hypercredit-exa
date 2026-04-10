# beam-postgres Extensions Reference

Always verify against:
- `https://haskell-beam.github.io/beam/user-guide/backends/beam-postgres/`
- `https://hackage.haskell.org/package/beam-postgres/docs/Database-Beam-Postgres.html`

## Imports

```haskell
import Database.Beam.Postgres
import qualified Database.Beam.Postgres as Pg
import Database.Beam.Postgres.Full  -- pgSelectWith, insertOnConflict, etc.
```

---

## DISTINCT ON — pgNubBy_

```haskell
-- Get one arbitrary user per email domain
Pg.pgNubBy_ (\u -> _userEmail u) $ all_ (_blogUsers blogDb)

-- More complex: one post per author (arbitrary selection)
Pg.pgNubBy_ (\p -> _postAuthor p) $ all_ (_blogPosts blogDb)
```

Generated SQL:
```sql
SELECT DISTINCT ON ("t0"."email") ...
FROM "users" AS "t0"
```

---

## Full-Text Search

```haskell
import Database.Beam.Postgres (TsVector, TsQuery, english, toTsVector, toTsQuery, (@@))

-- Check if document matches query
filter_ (\doc ->
    Pg.toTsVector (Just Pg.english) (_docContent doc)
      @@ Pg.toTsQuery (Just Pg.english) (val_ "haskell & beam"))
  $ all_ (myDb ^. myDocs)

-- Store tsvector in a column (use TsVector type)
data DocumentT f = Document
  { _docId      :: Columnar f Int64
  , _docContent :: Columnar f Text
  , _docVector  :: Columnar f TsVector   -- TSVECTOR column
  } deriving Generic
```

---

## Array Support

```haskell
import Database.Beam.Postgres (array_, arrayOf_, Vector)
import qualified Data.Vector as V

-- Column with array type
data MyT f = My
  { _myTags :: Columnar f (V.Vector Text)  -- TEXT[] column
  } deriving Generic

-- Construct array literal
val_ (V.fromList ["haskell", "beam"])  :: QExpr Postgres s (V.Vector Text)

-- Array aggregation
aggregate_ (\(user, post) ->
    (group_ (_userId user), Pg.pgArrayAgg (_postTitle post)))
  $ do ...

-- Array upper/lower bounds
Pg.arrayUpper_ myArray (val_ 1)
```

---

## JSON Support

```haskell
import Database.Beam.Postgres (PgJSON(..), PgJSONB(..))

data ConfigT f = Config
  { _configData :: Columnar f (PgJSONB Value)  -- JSONB column
  } deriving Generic
```

---

## Upsert / ON CONFLICT (beam-postgres Full module)

```haskell
import Database.Beam.Postgres.Full

-- INSERT ... ON CONFLICT DO NOTHING
runInsert $ Pg.insert (_blogUsers blogDb)
  (insertValues [user1])
  (Pg.onConflictDefault Pg.anyConflict Pg.onConflictDoNothing)

-- INSERT ... ON CONFLICT DO UPDATE (upsert)
runInsert $ Pg.insert (_blogUsers blogDb)
  (insertExpressions [User default_ (val_ "Alice") (val_ "alice@example.com") nothing_])
  (Pg.onConflict (Pg.conflictingFields primaryKey)
    (Pg.onConflictUpdateSet (\u _ ->
        _userEmail u <-. val_ "alice_new@example.com")))
```

---

## Row Locking (SELECT ... FOR UPDATE/SHARE)

```haskell
import qualified Database.Beam.Postgres as Pg

-- Lock all tables FOR SHARE
Pg.lockingAllTablesFor_ Pg.PgSelectLockingStrengthShare Nothing $
  filter_ (\u -> _userId u ==. val_ 1) $ all_ (_blogUsers blogDb)

-- Lock all tables FOR UPDATE, SKIP LOCKED
Pg.lockingAllTablesFor_ Pg.PgSelectLockingStrengthUpdate
  (Just Pg.PgSelectLockingNoWaitOrSkip)
  $ all_ (_blogUsers blogDb)

-- Lock specific tables only
Pg.lockingFor_ Pg.PgSelectLockingStrengthUpdate Nothing $ do
  user   <- locked_ (_blogUsers blogDb)
  post   <- all_ (_blogPosts blogDb)
  guard_ (_postAuthor post `references_` fst user)
  pure (snd user, post)
-- Only users table is locked
```

---

## CTEs / WITH Queries — pgSelectWith

```haskell
import Database.Beam.Postgres (pgSelectWith, selecting, reuse)

-- Standard selectWith (top-level only)
result <- runSelectReturningList $ selectWith $ do
  activePosts <- selecting $
    filter_ (\p -> _postPublished p ==. val_ True) $
    all_ (_blogPosts blogDb)
  pure $ reuse activePosts

-- pgSelectWith: CTEs can appear in JOINs (Postgres extension)
result <- runSelectReturningList $ select $
  pgSelectWith $ do
    topUsers <- selecting $
      limit_ 10 $ orderBy_ (\u -> desc_ (_userId u)) $
      all_ (_blogUsers blogDb)
    pure $ do
      user <- reuse topUsers
      post <- join_' (_blogPosts blogDb)
                (\p -> _postAuthor p ==?. pk user)
      pure (user, post)

-- Recursive CTE
result <- runSelectReturningList $ select $
  pgSelectWith $ do
    rec orgChart <- selecting $
          (fmap (\e -> (e, val_ (0 :: Int32))) $
            filter_ (isNull_ . _empManager) $ all_ (_blogEmployees blogDb))
          `unionAll_`
          (do
            (emp, level) <- reuse orgChart
            report <- filter_' (\e -> _empManager e ==?. just_ (pk emp)) $
                        all_ (_blogEmployees blogDb)
            pure (report, level + 1))
    pure $ reuse orgChart
```

---

## Streaming with Conduit

```haskell
import Database.Beam.Postgres.Conduit
import Conduit

-- Stream results without loading all into memory
runConduit $
  streamingRunSelect conn (select $ all_ (_blogPosts blogDb))
  .| mapC (\post -> _postTitle post)
  .| sinkList
```

---

## Returning Clauses

```haskell
-- INSERT … RETURNING
newPosts <- runInsertReturningList $
  insertReturning (_blogPosts blogDb) $
    insertExpressions [Post default_ (val_ "Hello") (UserId 1)]

-- UPDATE … RETURNING
updatedUsers <- runUpdateReturningList $
  updateReturning (_blogUsers blogDb)
    (\u -> _userBio u <-. val_ (Just "Updated"))
    (\u -> _userId u ==. val_ 1)
    id  -- projection: return full row

-- DELETE … RETURNING
deletedPosts <- runDeleteReturningList $
  deleteReturning (_blogPosts blogDb)
    (\p -> _postAuthor p ==?. val_ (UserId 42))
    id
```

---

## Money and Serial Types

```haskell
import Database.Beam.Postgres (money, serial, smallserial, bigserial, SqlMoney(..))

data PriceT f = Price
  { _priceId     :: Columnar f (SqlSerial Int32)  -- SERIAL
  , _priceAmount :: Columnar f SqlMoney           -- MONEY
  } deriving Generic
```

---

## Range Types

```haskell
import Database.Beam.Postgres.PgSpecific
-- Int4Range, Int8Range, NumRange, TsRange, TsTzRange, DateRange
```

Fetch full docs when using ranges:
`https://hackage.haskell.org/package/beam-postgres/docs/Database-Beam-Postgres.html`