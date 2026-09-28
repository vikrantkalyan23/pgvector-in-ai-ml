# pgvector

Tested setup: **PostgreSQL 18.x** (EDB installer) · **pgvector v0.8.6** · **pgAdmin 4** · macOS

---

## Part 1: What is pgvector?

**pgvector** is an open-source PostgreSQL extension that adds a `vector` data type and similarity-search operators to Postgres. It lets you store **embeddings** (arrays of numbers produced by AI models) right next to your normal relational data, and search them by "closeness" using plain SQL.

**Key ideas**

- **Embedding:** a list of numbers (for example 1536 floats) that represents the meaning of text, an image, or other data.
- **Similarity search:** given a query embedding, find the stored embeddings that are nearest to it.
- **Distance operators:**

| Operator | Meaning | Typical use |
|---|---|---|
| `<->` | Euclidean (L2) distance | General purpose |
| `<=>` | Cosine distance | Text embeddings (most common) |
| `<#>` | Negative inner product | Normalized embeddings |

- **Indexes:** `HNSW` (better recall/speed, slower build) and `IVFFlat` (faster build, needs tuning).

## Part 2: Use cases

- **Semantic search:** search docs, notes, or products by meaning, not just keywords.
- **RAG (Retrieval-Augmented Generation):** fetch relevant chunks to give an LLM context.
- **Recommendations:** "similar items" or "users like you".
- **Duplicate / near-duplicate detection:** find similar tickets, articles, or images.
- **Chatbots with memory:** store and retrieve past conversation snippets.
- **Anomaly detection and clustering** on vector features.

## Part 3: Why use pgvector?

- You already use (or can easily adopt) PostgreSQL, so there is **no new database to run**.
- Vectors live **alongside relational data**, so you can filter, join, and use transactions in one query.
- Full Postgres features apply: ACID, backups, replication, point-in-time recovery, roles/permissions.
- Works with any language that has a Postgres client.
- Widely available on managed services (RDS, Supabase, Neon, and others).

## Part 4: Pros and cons

| Pros | Cons |
|---|---|
| Free and open source | Not built for billions of vectors; scaling is bound by Postgres |
| One system for relational + vector data | Fewer vector-specific features than dedicated engines |
| SQL filters, joins, and transactions work with vectors | Index builds and large indexes can be memory- and time-heavy |
| Familiar tooling (psql, pgAdmin, ORMs) | Approximate indexes need tuning for recall vs. speed |
| Easy to start, easy to back up | High-dimension vectors use lots of storage |
| Backed by a large, active community | Limited built-in hybrid (keyword + vector) ranking compared to search engines |

**Rule of thumb:** pgvector is an excellent default for apps up to millions of vectors, especially if you are already on Postgres. Consider a dedicated vector DB when you need very large scale, very low latency at huge volume, or specialised features.

---

## Part 5: Installation on Mac (step by step)

### Step 0: Find out how Postgres was installed

```bash
which psql
```

| Path contains | Install method | Go to |
|---|---|---|
| `/Library/PostgreSQL/18` | EDB / postgresql.org installer | **Path B** (build from source) |
| `/opt/homebrew` or `/usr/local` | Homebrew | **Path A** |
| `/Applications/Postgres.app` | Postgres.app | Already includes pgvector; skip to Part 6 |

> Note: `brew install pgvector` only works for a **Homebrew-installed** Postgres. It does **not** install into an EDB-installer Postgres.

### Step 1: Put Postgres tools on your PATH (EDB installer)

```bash
export PATH="/Library/PostgreSQL/18/bin:$PATH"
psql --version
```

To make it permanent, add the `export PATH=...` line to `~/.zshrc`.

---

### Path A: Homebrew Postgres

```bash
brew install pgvector
brew list pgvector      # confirm it installed
```

Then jump to **Part 6**.

---

### Path B: EDB installer Postgres (build from source)

**B1. Install Xcode Command Line Tools** (skip if already installed)

```bash
xcode-select --install
```

**B2. Point to the correct `pg_config`**

```bash
export PG_CONFIG=/Library/PostgreSQL/18/bin/pg_config
$PG_CONFIG --version      # should print PostgreSQL 18.x
```

**B3. Clone, build, and install pgvector**

```bash
cd ~/Downloads
git clone --branch v0.8.6 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo --preserve-env=PG_CONFIG make install
```

If this succeeds, go to **Part 6**. If it fails, see the troubleshooting fix below.

**B4. Troubleshooting: "no such sysroot directory" / `inttypes.h not found`**

Cause: the Postgres build config points to an SDK folder (for example `MacOSX15.sdk`) that no longer exists after a Command Line Tools update.

1. See which SDKs you actually have:

```bash
ls /Library/Developer/CommandLineTools/SDKs/
```

2. Clean the failed build:

```bash
cd ~/Downloads/pgvector
make clean
```

3. **Fix option 1: symlink the missing SDK name** (replace `MacOSX26.sdk` with the SDK name from step 1):

```bash
sudo ln -s /Library/Developer/CommandLineTools/SDKs/MacOSX26.sdk \
           /Library/Developer/CommandLineTools/SDKs/MacOSX15.sdk
export PG_CONFIG=/Library/PostgreSQL/18/bin/pg_config
make
sudo --preserve-env=PG_CONFIG make install
```

4. **Fix option 2: override the sysroot instead of symlinking**

```bash
make clean
make CPPFLAGS="-isysroot $(xcrun --show-sdk-path)"
sudo --preserve-env=PG_CONFIG make install
```

---

## Part 6: Enable pgvector in a database

pgvector is now *installed on the server*, but each database must **enable** it separately.

### Step 1: (Optional) Restart Postgres

Usually not required, since `CREATE EXTENSION` picks up new extension files. Restart only if something does not appear:

```bash
sudo -u postgres /Library/PostgreSQL/18/bin/pg_ctl restart \
  -D /Library/PostgreSQL/18/data -m fast
```

### Step 2: Check that the extension is available

Run this **inside psql or pgAdmin's Query Tool** (it is SQL, not a shell command):

```sql
SELECT * FROM pg_available_extensions WHERE name = 'vector';
```

You should see one row with `name = vector`. If it returns nothing, the install in Part 5 did not complete or went to a different Postgres.

### Step 3: Enable it via pgAdmin 4

**Option A: Query Tool (simplest)**

1. Open pgAdmin 4 and connect to your server.
2. Expand **Servers → your server → Databases** and click your target database.
3. Open **Tools → Query Tool**.
4. Run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

**Option B: Point-and-click**

1. Expand your database → **Extensions**.
2. Right-click **Extensions → Create → Extension…**
3. Choose `vector` in the **Name** dropdown.
4. Click **Save**.

**Option C: psql**

```bash
psql -U postgres -d your_database -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Step 4: Verify

```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```

One row means it is enabled. Quick functional test:

```sql
CREATE TABLE test_vec (id serial PRIMARY KEY, embedding vector(3));
INSERT INTO test_vec (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');
SELECT id, embedding <-> '[1,2,3]' AS distance
FROM test_vec
ORDER BY distance
LIMIT 5;
```

### Step 5: Add an index (for real data)

```sql
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);
```

> The extension must be enabled **per database**. If you create a new database later, run `CREATE EXTENSION vector;` there too.

---

## Part 7: Comparison with other popular vector databases

> Features, licensing, and pricing change often. Treat this table as a high-level guide and check each project's docs before deciding.

| # | Database | Type | Open source? | Hosting | Index types | Best for | Main drawback |
|---|---|---|---|---|---|---|---|
| 1 | **pgvector** | Postgres extension | Yes (PostgreSQL license) | Self-host or any managed Postgres | HNSW, IVFFlat | Apps already on Postgres; SQL + vector in one place | Scale ceiling for very large workloads |
| 2 | **Pinecone** | Managed vector DB | No | Fully managed cloud | Proprietary ANN | Zero-ops production search | Vendor lock-in, cost, no self-hosting |
| 3 | **Milvus** | Dedicated vector DB | Yes (Apache 2.0) | Self-host or Zilliz Cloud | HNSW, IVF, DiskANN, others | Very large scale (billions of vectors) | Operational complexity |
| 4 | **Weaviate** | Dedicated vector DB | Yes (BSD-3) | Self-host or managed cloud | HNSW (+ flat) | Built-in hybrid search and integrations | Can be memory-hungry; more concepts to learn |
| 5 | **Qdrant** | Dedicated vector DB (Rust) | Yes (Apache 2.0) | Self-host or managed cloud | HNSW | Fast filtered search, strong performance | Smaller ecosystem than Postgres |
| 6 | **Chroma** | Lightweight vector DB | Yes (Apache 2.0) | Embedded/local, or cloud | HNSW | Prototyping, small apps, notebooks | Less proven at large scale |
| 7 | **Elasticsearch / OpenSearch** | Search engine | Varies (OpenSearch is Apache 2.0) | Self-host or managed | HNSW | Hybrid keyword + vector search | Heavy, resource-intensive to run |
| 8 | **Redis (vector search)** | In-memory data store | Varies by version/licence | Self-host or managed | HNSW, FLAT | Ultra-low latency, caching + vectors | RAM cost for large datasets |
| 9 | **MongoDB Atlas Vector Search** | Document DB feature | No (Atlas service) | Primarily Atlas cloud | HNSW-based | Teams already on MongoDB | Tied to Atlas; limited outside Mongo ecosystem |
| 10 | **FAISS** | Library (not a DB) | Yes (MIT) | Embedded in your code | Flat, IVF, HNSW, PQ, GPU | Research, custom pipelines, GPU search | No built-in storage, CRUD, or filtering |

### How to choose

- **Already on Postgres, up to millions of vectors:** pgvector.
- **Want zero infrastructure work:** Pinecone or a managed option.
- **Massive scale or heavy performance needs:** Milvus or Qdrant.
- **Need strong keyword + vector hybrid search:** Elasticsearch/OpenSearch or Weaviate.
- **Quick prototype on a laptop:** Chroma or FAISS.

---

## Quick command cheat sheet

```bash
# 1. Tools on PATH
export PATH="/Library/PostgreSQL/18/bin:$PATH"
export PG_CONFIG=/Library/PostgreSQL/18/bin/pg_config

# 2. Build & install pgvector (EDB Postgres)
cd ~/Downloads
git clone --branch v0.8.6 https://github.com/pgvector/pgvector.git
cd pgvector && make
sudo --preserve-env=PG_CONFIG make install
```

```sql
-- 3. Enable in your database (pgAdmin Query Tool or psql)
CREATE EXTENSION IF NOT EXISTS vector;
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**Docs:** https://github.com/pgvector/pgvector

---

## Part 8: Use of pgvector on Docker

The pgvector Docker image includes PostgreSQL and the extension files, which makes it useful for local development and repeatable deployments.

### Start with Docker Compose

Create a `docker-compose.yml` file:

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg18-trixie
    container_name: pgvector-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: vectordb
    ports:
      - "5432:5432"
    volumes:
      - pgvector_data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d vectordb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgvector_data:
```

Start PostgreSQL and connect to it:

```bash
docker compose up -d
docker compose exec postgres psql -U postgres -d vectordb
```

Enable pgvector in the database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
  id bigserial PRIMARY KEY,
  content text NOT NULL,
  embedding vector(3) NOT NULL
);

INSERT INTO documents (content, embedding) VALUES
  ('Postgres vector search', '[1,2,3]'),
  ('AI similarity search', '[1,1,1]');

SELECT id, content, embedding <=> '[1,2,3]' AS cosine_distance
FROM documents
ORDER BY cosine_distance
LIMIT 5;
```

Use the dimension required by your embedding model in a real application, such as `vector(1536)`, instead of the small demonstration dimension.

### Production-level steps

1. **Pin image and model versions.** Use an explicit pgvector image version instead of `latest`, and pin the embedding model so vector dimensions and semantics do not change unexpectedly.
2. **Use durable storage.** Mount `/var/lib/postgresql` for PostgreSQL 18 images and verify that data survives container replacement.
3. **Protect credentials.** Store passwords in Docker secrets or a cloud secret manager, never directly in a committed Compose file.
4. **Restrict network access.** Keep PostgreSQL on a private network and do not publish port `5432` to the public internet.
5. **Design the schema for filtering.** Add tenant, metadata, and timestamp columns, then create normal B-tree or GIN indexes for frequently used filters.
6. **Create the vector index after bulk loading.** For example:

   ```sql
   CREATE INDEX documents_embedding_hnsw_idx
   ON documents
   USING hnsw (embedding vector_cosine_ops);
   ```

7. **Match the index to the distance operator.** Use `vector_cosine_ops` with `<=>`, `vector_l2_ops` with `<->`, or `vector_ip_ops` with `<#>`.
8. **Measure query behavior.** Check production-like queries with `EXPLAIN (ANALYZE, BUFFERS)` and tune HNSW recall with `hnsw.ef_search` when needed.
9. **Plan capacity and maintenance.** Monitor CPU, memory, disk I/O, table size, index size, autovacuum, query latency, and connection-pool usage.
10. **Back up and test recovery.** Configure regular backups, point-in-time recovery, restore drills, and replication or high availability for critical workloads.
11. **Prefer managed PostgreSQL for critical systems.** A provider with pgvector support can handle patching, backups, monitoring, failover, and storage growth more reliably than a single Docker host.
