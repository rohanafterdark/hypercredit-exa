# Beam Join Patterns — Complete Reference

## Table of Contents
1. [The Two Styles of Inner Join](#inner-joins)
2. [LEFT JOIN — The Right Way](#left-join)
3. [related_ and relatedBy_](#related)
4. [Many-to-Many Joins](#many-to-many)
5. [Outer Joins](#outer-joins)
6. [Subselect Joins](#subselect)
7. [Performance Rules](#performance)

---

## 1. Inner Joins {#inner-joins}

### Monadic style (generates CROSS JOIN + WHERE)

```haskell
-- Generates: SELECT ... FROM users, posts WHERE posts.author_id = users.id
do
  user <- all_ (_blogUsers blogDb)
  post <- all_ (_blogPosts blogDb)
  guard_ (_postAuthor post `references_` user)
  pure (user, post)
```

> **Warning**: Postgres may not use indexes on the FK column in this form.
> For large tables, use explicit join syntax.

### Explicit INNER JOIN (join_ / join_')

```haskell
-- join_ expects Bool condition
do
  user <- all_ (_blogUsers blogDb)
  post <- join_ (_blogPosts blogDb)
            (\p -> isTrue_ (_postAuthor p ==?. pk user))
  pure (user, post)

-- join_' expects SqlBool condition — PREFERRED for Postgres index usage
do
  user <- all_ (_blogUsers blogDb)
  post <- join_' (_blogPosts blogDb)
            (\p -> _postAuthor p ==?. pk user)
  pure (user, post)
```

Generated SQL (join_'):
```sql
SELECT u.id, u.name, u.email, p.id, p.title, p.author_id
FROM users AS u
INNER JOIN posts AS p ON p.author_id = u.id
```

---

## 2. LEFT JOIN {#left-join}

### leftJoin_' (CORRECT — index-friendly)

```haskell
do
  user  <- all_ (_blogUsers blogDb)
  mPost <- leftJoin_' (all_ (_blogPosts blogDb))
              (\p -> _postAuthor p ==?. pk user)
  pure (user, mPost)
```

After a `leftJoin_'`, the right-side table is **nullable**. Access fields via:
```haskell
-- Pattern: use maybe_
maybe_ (val_ 0) (\p -> _postId p) mPost

-- Or use isNothing_ / isJust_
guard_ (isNothing_ (mPost ^. postId))
```

### leftJoin_ (with Bool — may hurt performance)

```haskell
-- This generates ON (IS TRUE (p.author_id = u.id)), breaking index usage
mPost <- leftJoin_ (all_ (_blogPosts blogDb))
           (\p -> _postAuthor p `references_` user)
```

**Rule of thumb**: Always use `leftJoin_'` + `==?.` in Postgres for any join
condition that involves a column comparison.

### Three-way LEFT JOIN example

```haskell
allUsersPostsComments <- runSelectReturningList $ select $ do
  user    <- all_ (_blogUsers blogDb)
  mPost   <- leftJoin_' (all_ (_blogPosts blogDb))
                (\p -> _postAuthor p ==?. pk user)
  mComment <- leftJoin_' (all_ (_blogComments blogDb))
                (\c -> _commentPost c ==?. pk mPost)
  pure (user, mPost, mComment)
```

---

## 3. related_ and relatedBy_ {#related}

```haskell
-- related_: follow an FK from a child to its parent (INNER JOIN)
do
  post   <- all_ (_blogPosts blogDb)
  author <- related_ (_blogUsers blogDb) (_postAuthor post)
  pure (post, author)
-- Generates proper INNER JOIN

-- relatedBy_: join with custom Bool condition
do
  post   <- all_ (_blogPosts blogDb)
  editor <- relatedBy_ (_blogUsers blogDb)
              (\u -> _userRole u ==. val_ "editor")
  pure (post, editor)

-- relatedBy_': SqlBool version (preferred)
do
  post   <- all_ (_blogPosts blogDb)
  editor <- relatedBy_' (_blogUsers blogDb)
              (\u -> _userRole u ==?. val_ "editor")
  pure (post, editor)
```

---

## 4. Many-to-Many {#many-to-many}

Manually join through the association table:

```haskell
data PostTagT f = PostTag
  { _ptPost :: PrimaryKey PostT f
  , _ptTag  :: PrimaryKey TagT f
  } deriving Generic

instance Beamable PostTagT
instance Table PostTagT where
  data PrimaryKey PostTagT f = PostTagId
    (PrimaryKey PostT f) (PrimaryKey TagT f) deriving Generic
  primaryKey pt = PostTagId (_ptPost pt) (_ptTag pt)
instance Beamable (PrimaryKey PostTagT)

-- Query: posts with their tags
postsWithTags <- runSelectReturningList $ select $ do
  post    <- all_ (_blogPosts blogDb)
  postTag <- join_' (_blogPostTags blogDb)
               (\pt -> _ptPost pt ==?. pk post)
  tag     <- related_ (_blogTags blogDb) (_ptTag postTag)
  pure (post, tag)
```

Using `manyToMany_`:
```haskell
-- Defined in Database.Beam.Query
postsAndTags <- manyToMany_ (_blogPostTags blogDb)
                  _ptPost _ptTag
                  (all_ (_blogPosts blogDb))
                  (all_ (_blogTags blogDb))
```

---

## 5. Outer Joins (Full Outer) {#outer-joins}

Only on backends that support it (Postgres does):

```haskell
import Database.Beam.Query (outerJoin_')

(mUser, mPost) <- outerJoin_'
  (all_ (_blogUsers blogDb))
  (all_ (_blogPosts blogDb))
  (\u p -> pk p ==?. _postAuthor u)
```

---

## 6. Subselect Joins {#subselect}

Use `subselect_` to scope an inner query, then join against it:

```haskell
do
  user <- all_ (_blogUsers blogDb)
  (authorPk, postCount) <- subselect_ $
    aggregate_ (\(apk, _p) -> (group_ apk, as_ @Int32 countAll_)) $ do
      u <- all_ (_blogUsers blogDb)
      p <- leftJoin_' (all_ (_blogPosts blogDb))
             (\p -> _postAuthor p ==?. pk u)
      pure (pk u, p)
  guard_ (authorPk `references_` user)
  pure (user, postCount)
```

---

## 7. Performance Rules for Postgres {#performance}

1. **Always use `leftJoin_'`** with `==?.` for LEFT JOINs. Using `leftJoin_`
   with `==.` wraps the condition in `IS TRUE (...)`, preventing index use.

2. **Use explicit `join_'`** over monadic joins for large tables. Monadic style
   generates `CROSS JOIN + WHERE` which may not use indexes.

3. **`references_`** is fine for inner join conditions (`guard_` context) on
   non-nullable FKs — it generates a direct equality.

4. **`==?.` in ON clauses** generates clean `a.col = b.col` SQL that Postgres
   indexes can match.

5. **Avoid wrapping join conditions** in `isTrue_`, `sqlBool_`, etc. in ON
   clause position — this prevents index use.