# beam-migrate Reference

Always verify current API at:
`https://hackage.haskell.org/package/beam-migrate`

## Setup

```cabal
-- cabal.project or package.yaml
build-depends:
    beam-core
  , beam-postgres
  , beam-migrate
```

```haskell
import Database.Beam.Migrate
import Database.Beam.Postgres.Migrate (migrationBackend)
```

---

## Checked Database Settings

```haskell
-- Use defaultMigratableDbSettings instead of defaultDbSettings
-- when you want migration support
blogDbChecked :: CheckedDatabaseSettings Postgres BlogDb
blogDbChecked = defaultMigratableDbSettings `withDbModification`
  dbModification
    { _blogUsers = modifyTableFields tableModification
        { _userEmail = fieldNamed "email_address" }
    }

-- Extract plain settings from checked settings
blogDb :: DatabaseSettings Postgres BlogDb
blogDb = unCheckDatabase blogDbChecked
```

---

## Define Migrations

```haskell
import Database.Beam.Migrate.Simple
import Database.Beam.Postgres.Migrate

-- Initial migration: create all tables
initialSetup :: Migration Postgres (CheckedDatabaseSettings Postgres BlogDb)
initialSetup = do
  users <- createTable "users" $
    User
      { _userId    = field "id"    int notNull unique
      , _userName  = field "name"  (varchar (Just 100)) notNull
      , _userEmail = field "email_address" (varchar (Just 200)) notNull unique
      , _userBio   = field "bio"   (maybeType text)
      }
  posts <- createTable "posts" $
    Post
      { _postId     = field "id"    int notNull unique
      , _postTitle  = field "title" (varchar (Just 500)) notNull
      , _postAuthor = UserId (field "author_id" int notNull)
      }
  pure $ BlogDb (unmovedTable users) (unmovedTable posts)
```

---

## Run Migrations

```haskell
import Database.Beam.Migrate.Simple (autoMigrate, checkSchema)

-- Auto-migrate: apply all pending migrations
conn <- connectPostgreSQL connStr
withTransaction conn $
  runBeamPostgresDebug putStrLn conn $
    autoMigrate migrationBackend blogDbChecked

-- Check schema matches (dry run)
withTransaction conn $
  runBeamPostgres conn $
    checkSchema migrationBackend blogDbChecked >>= \case
      Left errs -> mapM_ print errs
      Right ()  -> putStrLn "Schema OK"
```

---

## Generate Schema from Existing Database

```bash
# Use the beam-migrate CLI tool
beam-migrate --backend postgres --conn "host=localhost dbname=mydb" \
  generate-schema > Schema.hs
```

---

## Data Types in Migrations

```haskell
-- Common beam-migrate data type constructors
int        :: DataType Postgres Int32
bigint     :: DataType Postgres Int64
smallint   :: DataType Postgres Int16
text       :: DataType Postgres Text
varchar    :: Maybe Word -> DataType Postgres Text
boolean    :: DataType Postgres Bool
double     :: DataType Postgres Double
timestamp  :: DataType Postgres LocalTime
timestamptz :: DataType Postgres UTCTime
date       :: DataType Postgres Day
bytea      :: DataType Postgres ByteString
uuid       :: DataType Postgres UUID  -- requires beam-postgres

-- Nullable
maybeType  :: DataType Postgres a -> DataType Postgres (Maybe a)

-- Serial (auto-increment)
serial     :: DataType PgDataTypeSyntax (SqlSerial Int32)
bigserial  :: DataType PgDataTypeSyntax (SqlSerial Int64)
```

---

## Constraints

```haskell
-- Field-level constraints
notNull   :: FieldCheck
unique    :: FieldCheck
primaryKey :: FieldCheck

-- Example composite primary key
createTable "post_tags" $
  PostTag
    { _ptPost = PostId (field "post_id" int (references (_blogPosts blogDb) notNull))
    , _ptTag  = TagId  (field "tag_id"  int (references (_blogTags  blogDb) notNull))
    }
```