# PostgreSQL vs SQLite Architecture Comparison

Although both Postgre and SQLite are SQL databases the design, purpose and scale at which both of these operate is very different. Hence, I think it is important to point out that although we are comparing the two, they are fundamentally not interchangable for most production workloads.

PostgreSQL is a database server designed for sclability, reliability and complicated concurrent usecases. On the other hand, SQLite is a serverless DB build specifically to replace embedded config files(txt etc.) with a structured and SQL queryable system.

## PostgreSQL

PostgreSQL or simply Postgres is a client-server relational database system. It runs as a background server process(daemon) on a host machine and listens for client connections over TCP. Clients communicate with the server using PostgreSQL's wire protocol, sending SQL queries and commands to store, retrieve, and manage data.

Designed for concurrency and scale, Postgres is ideal for multi-user, concurrent and high thoughput systems where despite the high volume volume of read/write operations, data integretity and availability are of fundamental importance.

In most production use cases, Postgres lives in a seprate machine and clients connect to it using the network.

## SQLite

SQLite as also evident from its name is meant to be light weight. Unlike Postgres, SQLite is

1. an embedded database, it lives on the same machine as the application. 
2. a library, not a server. The application uses `sqlite` to manage a single file in the file system, which holds all data.

As is evident form the above points, SQLite is not meant for highly concurrent and durable usecases usecases. In most production systems where SQLite is used for persistent configs, states and history, where deploying a full database server will be inefficient, unstable and in some cases not always available(offline access).

## Architectural Differences

### Access Model

Another fundamental difference between SQLite and PostgreSQL is their access model.

PostgreSQL is designed as a database server. The database server typically runs as a separate process, often on a different machine, and applications communicate with it over a network connection. This architecture enables multiple clients to concurrently access and modify shared data, making PostgreSQL well-suited for multi-user and distributed systems.

However, because data access depends on communication with the database server, network latency and connectivity become part of the critical path. If a client loses connectivity to the database server, it can no longer directly access or modify data unless additional caching or replication mechanisms have been implemented.

SQLite has a different access model. The database engine is embedded directly within the application process, and the database itself is stored as a local file. Queries are executed through local function calls rather than network requests. As a result, applications can continue to access data even when completely offline, making SQLite particularly attractive for mobile applications, desktop software, embedded systems, and edge deployments where reliable network connectivity cannot be assumed.

This difference reflects the primary design goals of the two systems: PostgreSQL prioritizes shared access, concurrency, and centralized data management, while SQLite prioritizes simplicity, locality, and offline accessibility.

**Example — how a connection looks in each system:**

```python
# SQLite: no network, no daemon, just open a local file
import sqlite3
conn = sqlite3.connect("/app/data/local.db")   # opens (or creates) a file

# PostgreSQL: connect over TCP to a running server
import psycopg2
conn = psycopg2.connect(
    host="db.internal",
    port=5432,
    dbname="myapp",
    user="appuser",
    password="secret"
)
```

With SQLite the application _is_ the database engine. With PostgreSQL the application is a client that sends SQL over a socket — if `db.internal` is unreachable, every query fails.

### Concurrency

SQLite is a file as a database (or database as a file), there is no central server to manage concurrent access to the same file, hence it used a file-lock. If, one writer process has acquired a lock on the file, another writer cannot write to the same file at the same time. SQLite hence allows on the multiple readers on but only one writer at a time.

PostgreSQL has a central server that receives all client connections and acts as a central coordinator for database access. Because every read and write request passes through the server, PostgreSQL can manage locks, transactions, snapshots, and concurrent access in a much more sophisticated manner.

Postgres uses MVCC(Multi Version Concurrency Control), when multiple clients modify the resource, Postgres create different snapshots for each when they start. The server coordinates row locks, visibility and ordering of transactions. 

Every row in Postgres contains metadata

```jsx
xmin: transaction that create this row version
xmax: transaction that deleted or modified that the row version
```

Example:

Lets assume a table with one column(Name) which stores the name of the user.

| id | Name |
| --- | --- |
| 1 | Alice |

for the row `where id=1` . There are two transctions

1. T69: Starts by reading the the row.
2. T169: Starts after T1 and modifies the row. (Alice → Bob)

Old Row version `name = Alice`

```jsx
xmin = 69
xmax = 169
```

New Row version `name = Bob`

```jsx
xmin = 169
xmax = NULL
```

Transaction 69 created the row version name is Alice and transaction 169 has modified it. For Bob, that was created by Tx. 169, it has not been modified by any transaction so the xmax is NULL. 

When a transaction reads data, PostgreSQL uses the transaction's snapshot together with the `xmin` and `xmax` values to determine which row versions are visible. Older transactions may continue to see the Alice version, while newer transactions see the Bob version. This allows readers and writers to operate concurrently without blocking each other, allowing for thousands of concurrent clients.

**Example — concurrent writers in SQLite vs PostgreSQL:**

```bash
# Shell 1
sqlite3 mydb.db "BEGIN EXCLUSIVE; UPDATE users SET name='Bob' WHERE id=1;"

# Shell 2
sqlite3 mydb.db "UPDATE users SET name='Charlie' WHERE id=1;"
# Runtime error: database is locked
```

Infact, `Database is locked` is one of the most frequently seen error for write-heavy services working on a SQLite database.

SQLite uses a single file-level write lock. The second writer gets an immediate error (or waits up to `busy_timeout` milliseconds if configured).

```sql
-- PostgreSQL: both transactions run simultaneously, no blocking on reads
-- Session 1
BEGIN;
SELECT name FROM users WHERE id = 1;  -- sees 'Alice'

-- Session 2 (concurrently)
BEGIN;
UPDATE users SET name = 'Bob' WHERE id = 1;
COMMIT;  -- succeeds without waiting for Session 1

-- Back in Session 1 — still sees 'Alice' due to its snapshot
SELECT name FROM users WHERE id = 1;  -- still 'Alice'
COMMIT;
```

PostgreSQL's MVCC lets both sessions proceed. Session 1 keeps reading its own consistent snapshot even after Session 2 commits.

### Data Types

SQLite is dyanmically typed model, each column has a type affinity rather than a strict column type. While columns may be declared with types such as `INTEGER`, `TEXT`, or `REAL`, these declarations act as recommendations rather than strict constraints (similar to type hinting in python). 

The latest version of sqlite although does support strict types, but the default dynamic typing can cause unexpected behavioural errors and weaker data validation.

**Example — SQLite's type affinity silently accepting wrong data:**

```sql
-- SQLite
CREATE TABLE products (id INTEGER, price REAL, name TEXT);
INSERT INTO products VALUES ('hello', 'world', 42);  -- no error will be raised here.
SELECT typeof(id), typeof(price), typeof(name) FROM products;
-- text | text | integer : SQLite stored what you gave, not what you declared

-- PostgreSQL
CREATE TABLE products (id INTEGER, price REAL, name TEXT);
INSERT INTO products VALUES ('hello', 'world', 42);
-- ERROR:  invalid input syntax for type integer: "hello"
```

In SQLite, a column declared `INTEGER` stores the string `'hello'`. In PostgreSQL, the schema is enforced — the insert fails with a clear error.

Postgres on the other hand, is strictly statically typed, with a very large set of supported data types including support for optimal read/wite with JSONB and arrays as well.

### Stored Procedures

PostgreSQL supports stored procedures and user-defined functions that execute directly inside the database server.

Stored procedures reduce application-side logic, improve performance by minimizing network round trips, and allow business rules to be enforced directly within the database.

SQLite does not provide native stored procedures. Complex business logic must typically be implemented in the application layer.

**Example — applying a discount in PostgreSQL (server-side) vs SQLite (application-side):**

```sql
-- PostgreSQL: logic lives inside the database
CREATE OR REPLACE FUNCTION apply_discount(product_id INT, pct NUMERIC)
RETURNS VOID AS $$
BEGIN
    UPDATE products
    SET price = price * (1 - pct / 100)
    WHERE id = product_id;
END;
$$ LANGUAGE plpgsql;

-- Call it from any client with a single round trip
SELECT apply_discount(42, 10);
```

```python
# SQLite: logic must live in the application
def apply_discount(conn, product_id, pct):
    cursor = conn.cursor()
    cursor.execute("SELECT price FROM products WHERE id = ?", (product_id,))
    row = cursor.fetchone()               # round trip 1: read
    new_price = row[0] * (1 - pct / 100)
    cursor.execute(                       # round trip 2: write
        "UPDATE products SET price = ? WHERE id = ?",
        (new_price, product_id)
    )
    conn.commit()
```

The SQLite version requires two separate statements and a Python calculation in between. In a high-frequency workload the extra application-layer hop adds up.

### JSONB Support

One of PostgreSQL's most powerful features is `JSONB` (Binary JSON).

Unlike plain JSON storage, `JSONB` is stored in a binary representation and supports indexing for fast lookups.

SQLite provides JSON functions through the JSON1 extension but stores JSON as ordinary text and lacks PostgreSQL's deep integration and indexing capabilities.

**Example — querying JSON fields and indexing them:**

```sql
-- PostgreSQL: JSONB with a GIN index for fast key lookups
CREATE TABLE events (
    id   SERIAL PRIMARY KEY,
    data JSONB
);
INSERT INTO events (data) VALUES
    ('{"user": "alice", "action": "login",  "region": "us-east"}'),
    ('{"user": "bob",   "action": "purchase","region": "eu-west"}');

-- Create a GIN index so key lookups don't require a full scan
CREATE INDEX idx_events_data ON events USING GIN (data);

-- This query hits the index
SELECT data->>'user' FROM events WHERE data @> '{"action": "login"}';
-- Result: alice

-- SQLite: JSON stored as TEXT, no indexing on keys
CREATE TABLE events (id INTEGER PRIMARY KEY, data TEXT);
INSERT INTO events (data) VALUES
    ('{"user": "alice", "action": "login"}');

-- SQLite must parse every row's text to evaluate the condition
SELECT json_extract(data, '$.user')
FROM events
WHERE json_extract(data, '$.action') = 'login';
-- always does a full table scan
```

The `@>` operator in PostgreSQL uses the GIN index to find matching documents in sub-millisecond time even with millions of rows. SQLite's `json_extract` in a WHERE clause always scans every row.

### Array Support

PostgreSQL supports native array types.

This is particularly useful for tags, permissions, categories, and other multi-valued attributes.

SQLite has no native array type and typically models such data using JSON or separate relational tables.

**Example — storing and querying tags:**

```sql
-- PostgreSQL: native array column
CREATE TABLE articles (
    id    SERIAL PRIMARY KEY,
    title TEXT,
    tags  TEXT[]
);
INSERT INTO articles (title, tags) VALUES
    ('Intro to SQL',  ARRAY['database', 'sql', 'beginner']),
    ('Advanced MVCC', ARRAY['database', 'postgres', 'internals']);

-- Find all articles tagged 'database'
SELECT title FROM articles WHERE 'database' = ANY(tags);

-- GIN index
CREATE INDEX idx_tags ON articles USING GIN (tags);
```

```sql
-- SQLite workaround 1: store as comma-separated text
INSERT INTO articles (title, tags) VALUES ('Intro to SQL', 'database,sql,beginner');
SELECT title FROM articles WHERE ',' || tags || ',' LIKE '%,database,%';
-- slow and error-prone

-- SQLite workaround 2: separate join table
CREATE TABLE article_tags (article_id INTEGER, tag TEXT);
INSERT INTO article_tags VALUES (1, 'database'), (1, 'sql'), (1, 'beginner');
SELECT DISTINCT a.title FROM articles a
JOIN article_tags t ON t.article_id = a.id
WHERE t.tag = 'database';
```

These specialized indexing mechanisms allow PostgreSQL to efficiently support a wide range of workloads, including full-text search, document storage, geospatial applications, analytics, and time-series processing. SQLite provides a much simpler indexing model and lacks native equivalents for many of these advanced index types.

### Indexing

On a high level, both SQLite and Postgres use B-Tree indexes, yet their indexing architectures are fundamentally different.

SQLite stores both tables and indexes directly inside a single database file. An index is implemented as a separate B-Tree structure within the file. SQLite traverses the index B-Tree to locate the matching entry and then follows a pointer to retrieve the corresponding row from the table B-Tree. Because SQLite is designed as an embedded database, index management is relatively simple. There is no shared buffer pool, background maintenance process, or sophisticated index access framework. Index pages are read directly from the database file and cached by SQLite's pager layer.

PostgreSQL treats indexes as independent storage structures managed by the database server. Unlike SQLite, PostgreSQL tables are stored as heaps rather than clustered B-Trees. A B-Tree index entry contains: Indexed Value + Tuple Identifier (TID). The TID points to the physical location of the row inside the heap.

**Example — seeing the index lookup in action with EXPLAIN:**

```sql
-- PostgreSQL
CREATE TABLE orders (id SERIAL PRIMARY KEY, customer_id INT, total NUMERIC);
CREATE INDEX idx_customer ON orders (customer_id);

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

```
Index Scan using idx_customer on orders  (cost=0.29..8.31 rows=1 width=36)
                                         (actual time=0.021..0.023 rows=3 loops=1)
  Index Cond: (customer_id = 42)
Planning Time: 0.1 ms
Execution Time: 0.05 ms
```

PostgreSQL walks the B-Tree index to find the matching TIDs, then fetches only those rows from the heap. Without the index it would show `Seq Scan` and visit every row.

```sql
-- SQLite equivalent
CREATE TABLE orders (id INTEGER PRIMARY KEY, customer_id INTEGER, total REAL);
CREATE INDEX idx_customer ON orders (customer_id);

EXPLAIN QUERY PLAN SELECT * FROM orders WHERE customer_id = 42;
```

```
QUERY PLAN
------------------------------------------------------------------
SEARCH orders USING INDEX idx_customer (customer_id=?)
```

SQLite's planner confirms the index is used, but note the output is much simpler — there are no cost estimates, no actual vs planned rows, and no timing. PostgreSQL's planner collects table statistics (via `ANALYZE`) and uses them to pick the optimal plan among multiple candidates.

### Security and Access Control

Both SQLite and Postgres provide a way for security models and access control features, but again, they differ in implementation. SQLite uses a simple approach, since the DB is essentially a file, access to that file is controlled by the operating system. SQLite does not support built in user accounts, roles, privileges or similar authentication or authorization mechanisms. Since in most cases the DB will be owned by a single application and is not exposed to external clients, this is not an issue. Multiple mobile application, browsers, and IoT devices will manage their own DB files.

Postgres on the other hand has builtin support for multi-user environments and provides a complete authentication and authorization framework

- User accounts and roles
- Fine-grained privileges and access control
- Role inheritance
- Row-Level Security (RLS)
- SSL/TLS encrypted client connections
- Authentication methods such as passwords, certificates, LDAP, Kerberos, and OAuth integrations
- Auditing and monitoring capabilities

These features allow PostgreSQL to safely serve multiple users and applications simultaneously while enforcing security boundaries at the database level.

**Example — role-based access and Row-Level Security:**

```sql
-- PostgreSQL: create a read-only role for a reporting service
CREATE ROLE reporting_user LOGIN PASSWORD 'secret';
GRANT CONNECT ON DATABASE myapp TO reporting_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_user;

-- Row-Level Security: each user only sees their own rows
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_orders ON orders
    USING (owner_id = current_user_id());
```

```bash
# SQLite: security is entirely the OS's job
ls -l myapp.db
# -rw-r--r-- 1 appuser appgroup  myapp.db

# Anyone with read access to the file can read every row
sqlite3 myapp.db "SELECT * FROM orders;"
```

With SQLite you secure the file at the OS level — there's no concept of database users, roles, or row visibility policies.

### Scaling, Partitions and Sharding

The design and implementation differences between SQLite and Postgres are most evident when it comes to scaling.  

SQLite being embedded is limited to the resources provided to the application by the host machine, it scales with the application. While the file can be copied, backed up, or synchronized to other systems, SQLite does not provide built-in support for distributed databases, replication, clustering, or sharding.

#### Data Partitions

SQLite does not support data partitions natively, it needs to be implemented at the application level.

PostgreSQL supports table partitioning, allowing a large logical table to be split into smaller physical partitions.

For example: Partitions can then be created for different date ranges, 

```jsx
CREATE TABLE orders ( 
	id BIGINT,
	order_date DATE 
) 
PARTITION BY RANGE (order_date);
```

Postgres will now store orders with similar order_date physically adjacently, leading to faster query execution. Applications continue querying a single logical table while PostgreSQL automatically routes data to the appropriate partition.

#### Data Sharding

SQLite does not provide built-in sharding support.

PostgreSQL does not provide automatic sharding in its core database engine. However, its architecture is significantly more suitable for distributed deployments and can be extended using solutions such as Citus or application-level sharding strategies.

In a typical sharded PostgreSQL deployment, a coordinator node receives client requests and determines which shard contains the required data. The coordinator then forwards the query to the appropriate shard nodes, aggregates the results if necessary, and returns the final response to the client.

### Logging, Replication and Recovery

The architectural differences between SQLite and PostgreSQL are particularly evident in how they handle durability, replication, and failure recovery.

#### Logging

Both databases use Write-Ahead Logging (WAL) to ensure durability, but their implementations serve different purposes.

SQLite uses a WAL file primarily to improve concurrency and crash recovery. When WAL mode is enabled, write operations are first recorded in the WAL file before being merged into the main database file. This allows readers and writers to operate concurrently while ensuring that committed transactions can be recovered after a crash.

PostgreSQL also uses Write-Ahead Logging, but as a central component of its architecture. Before any change is written to a data page, a corresponding WAL record is written to disk. These WAL records contain sufficient information to replay database operations and reconstruct a consistent state after a failure.

#### Replication

SQLite does not provide native replication capabilities. Since the database is simply a local file, replication must be implemented externally through file synchronization, application-level logic, or third-party solutions.

PostgreSQL provides built-in replication support. As transactions are committed on the primary server, WAL records are streamed to standby servers, which replay the changes and maintain an up-to-date copy of the database.

#### Recovery

SQLite recovery is relatively straightforward. After a crash, SQLite examines the WAL or rollback journal and replays or rolls back incomplete operations to restore database consistency.

When PostgreSQL starts after an unexpected shutdown:

1. It locates the most recent checkpoint.
2. It scans WAL records generated after that checkpoint.
3. It replays all committed changes that may not yet have reached the data files.
4. It discards incomplete or uncommitted transactions.

Both SQLite and PostgreSQL are ACID-compliant and can recover from power failures, operating system crashes, and unexpected shutdowns using their respective journaling mechanisms.

The primary difference lies in the scope of their recovery systems. SQLite's journaling is designed to restore a single database file to a consistent state after a crash, whereas PostgreSQL's Write-Ahead Logging (WAL) serves as the foundation for crash recovery, replication, WAL archiving, and point-in-time recovery (PITR). As a result, PostgreSQL's recovery infrastructure is considerably more sophisticated and better suited for high-availability and enterprise deployments.
