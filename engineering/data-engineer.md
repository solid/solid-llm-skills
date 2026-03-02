# Full Stack Data Engineer Skill

You are an expert data engineer proficient in PostgreSQL, Scala, MongoDB, Supabase, SQL, row-level security (RLS), and field-level security. You design reliable data pipelines, schemas, and access control systems.

---

## PostgreSQL

### Schema Design

```sql
-- Use explicit schemas for separation of concerns
CREATE SCHEMA app;
CREATE SCHEMA audit;

-- Prefer UUID primary keys for distributed systems
CREATE TABLE app.users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Foreign key with cascading delete
CREATE TABLE app.posts (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES app.users(id) ON DELETE CASCADE,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partial index for common filtered queries
CREATE INDEX idx_posts_user_id ON app.posts(user_id);
CREATE INDEX idx_users_email_lower ON app.users(lower(email)); -- case-insensitive lookup
```

### Common Query Patterns

```sql
-- Pagination with keyset (cursor-based) — faster than OFFSET for large tables
SELECT id, title, created_at
FROM app.posts
WHERE user_id = $1
  AND created_at < $2  -- cursor
ORDER BY created_at DESC
LIMIT 20;

-- Upsert
INSERT INTO app.users (id, email, name)
VALUES ($1, $2, $3)
ON CONFLICT (email) DO UPDATE
  SET name = EXCLUDED.name,
      updated_at = now();

-- Window function: rank posts per user
SELECT
  user_id,
  title,
  created_at,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM app.posts;

-- JSON aggregation
SELECT
  u.id,
  u.name,
  json_agg(json_build_object('id', p.id, 'title', p.title) ORDER BY p.created_at DESC) AS posts
FROM app.users u
LEFT JOIN app.posts p ON p.user_id = u.id
GROUP BY u.id, u.name;
```

### Performance

- Use `EXPLAIN ANALYZE` to diagnose slow queries
- Add indexes on columns used in `WHERE`, `JOIN`, and `ORDER BY`
- Use `VACUUM ANALYZE` regularly (or enable autovacuum)
- Connection pooling: use **PgBouncer** (transaction mode) or **RDS Proxy**
- Avoid `SELECT *` in application queries — specify columns

---

## Row-Level Security (RLS)

RLS enforces data access rules at the database level — even if application code is buggy, users cannot see each other's data.

```sql
-- Enable RLS on a table
ALTER TABLE app.posts ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their own posts
CREATE POLICY posts_select_own ON app.posts
  FOR SELECT
  USING (user_id = current_setting('app.current_user_id')::UUID);

-- Policy: users can only insert posts for themselves
CREATE POLICY posts_insert_own ON app.posts
  FOR INSERT
  WITH CHECK (user_id = current_setting('app.current_user_id')::UUID);

-- Policy: users can only update/delete their own posts
CREATE POLICY posts_modify_own ON app.posts
  FOR UPDATE USING (user_id = current_setting('app.current_user_id')::UUID);

CREATE POLICY posts_delete_own ON app.posts
  FOR DELETE USING (user_id = current_setting('app.current_user_id')::UUID);
```

```sql
-- Set the current user context before running queries
-- (done by application layer per request)
SET LOCAL app.current_user_id = 'abc123-...';
SELECT * FROM app.posts;  -- automatically filtered
```

### RLS with Supabase

Supabase uses PostgreSQL RLS with `auth.uid()` as the JWT subject:

```sql
-- Supabase RLS: user can read their own data
CREATE POLICY "Users read own posts" ON posts
  FOR SELECT USING (auth.uid() = user_id);

-- Allow public read of published posts
CREATE POLICY "Public reads published" ON posts
  FOR SELECT USING (published = true);
```

---

## Field-Level Security

Field-level security (FLS) restricts access to specific columns.

### Approach 1: Views

```sql
-- Create a view that omits sensitive columns
CREATE VIEW app.users_public AS
  SELECT id, name, created_at  -- omit email, password_hash, etc.
  FROM app.users;

GRANT SELECT ON app.users_public TO app_role;
REVOKE SELECT ON app.users FROM app_role;  -- deny access to base table
```

### Approach 2: Column Privileges

```sql
-- Grant column-level SELECT only for non-sensitive columns
GRANT SELECT (id, name, created_at) ON app.users TO limited_role;
-- email column not granted — SELECT on it will fail
```

### Approach 3: Application Layer

- Use DTO projections in your ORM/query layer to strip sensitive fields
- Annotate fields with sensitivity metadata and enforce in middleware

---

## Supabase

Supabase is a Postgres-based BaaS providing auth, storage, realtime, and auto-generated REST/GraphQL APIs.

```bash
pnpm add @supabase/supabase-js
```

```typescript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!);

// Select with filter
const { data, error } = await supabase
  .from("posts")
  .select("id, title, created_at")
  .eq("user_id", userId)
  .order("created_at", { ascending: false })
  .limit(20);

// Insert
const { data: post, error } = await supabase
  .from("posts")
  .insert({ title: "Hello", body: "World", user_id: userId })
  .select()
  .single();

// Realtime subscription
const channel = supabase
  .channel("posts-changes")
  .on("postgres_changes", { event: "INSERT", schema: "public", table: "posts" }, (payload) => {
    console.log("New post:", payload.new);
  })
  .subscribe();
```

### Supabase Best Practices

- Always enable RLS on every table — the anon key is public
- Use service role key only in trusted server-side code
- Use `supabase gen types typescript` to generate TypeScript types from your schema
- Use Supabase migrations (`supabase db diff`, `supabase migration new`) for schema changes

---

## MongoDB

```bash
pnpm add mongodb
```

```typescript
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI!);
const db = client.db("myapp");
const posts = db.collection("posts");

// Insert
const result = await posts.insertOne({
  title: "Hello",
  body: "World",
  userId: new ObjectId(userId),
  createdAt: new Date(),
});

// Find with projection (field-level security equivalent)
const userPosts = await posts
  .find({ userId: new ObjectId(userId) })
  .project({ title: 1, createdAt: 1 })  // only return these fields
  .sort({ createdAt: -1 })
  .limit(20)
  .toArray();

// Update
await posts.updateOne(
  { _id: new ObjectId(postId), userId: new ObjectId(userId) },
  { $set: { title: "Updated" } }
);
```

### Schema Design for MongoDB

- **Embed** related data when it's always read together (1:1, 1:few)
- **Reference** (store ObjectId) when data is large, shared, or independently updated (1:many, many:many)
- Add indexes for all query filters: `posts.createIndex({ userId: 1, createdAt: -1 })`
- Use `$lookup` sparingly — it's a JOIN in a document DB; consider schema design first

---

## Scala (Data Pipelines)

Scala with Apache Spark is the standard for large-scale batch data processing.

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

val spark = SparkSession.builder()
  .appName("UserActivityPipeline")
  .getOrCreate()

// Read from S3
val events = spark.read
  .parquet("s3://my-bucket/events/year=2024/month=03/")

// Transform
val userActivity = events
  .filter(col("event_type") === "page_view")
  .groupBy("user_id", "date")
  .agg(
    count("*").as("page_views"),
    countDistinct("session_id").as("sessions")
  )
  .orderBy(desc("page_views"))

// Write to output
userActivity.write
  .mode("overwrite")
  .partitionBy("date")
  .parquet("s3://my-bucket/aggregates/user-activity/")

spark.stop()
```

### Scala / Spark Best Practices

- Use `DataFrames` / `Datasets` over RDDs for performance and optimisation
- Cache (`df.cache()`) only intermediate results that are reused multiple times
- Partition by the column most commonly used in filters (`date`, `user_id`)
- Monitor Spark UI for skewed partitions and slow stages
- Use `coalesce()` before writing to reduce small file count in output

---

## Data Pipeline Patterns

### ETL vs ELT

| | ETL | ELT |
|-|-----|-----|
| Transform | Before loading | After loading |
| Best for | Structured → structured | Raw → data warehouse |
| Tools | Spark, dbt, custom | BigQuery, Snowflake + dbt |

### Data Quality Checks

```sql
-- Null check
SELECT COUNT(*) FROM app.orders WHERE user_id IS NULL;

-- Duplicate check
SELECT email, COUNT(*) FROM app.users GROUP BY email HAVING COUNT(*) > 1;

-- Referential integrity check
SELECT o.id FROM app.orders o
LEFT JOIN app.users u ON u.id = o.user_id
WHERE u.id IS NULL;
```

---

## Key Links

| Resource | URL |
|----------|-----|
| PostgreSQL Docs | https://www.postgresql.org/docs/ |
| Supabase Docs | https://supabase.com/docs |
| MongoDB Docs | https://www.mongodb.com/docs/ |
| PgBouncer | https://www.pgbouncer.org/ |
| Apache Spark (Scala) | https://spark.apache.org/docs/latest/ |
| dbt | https://docs.getdbt.com/ |
