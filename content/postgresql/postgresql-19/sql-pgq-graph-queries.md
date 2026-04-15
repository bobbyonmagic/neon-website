---
title: 'PostgreSQL 19 SQL/PGQ Property Graph Queries'
page_title: 'PostgreSQL 19 SQL/PGQ - Graph Queries on Existing Tables'
page_description: 'Learn how to use PostgreSQL 19 SQL/PGQ to define property graphs over existing tables and query relationships with graph pattern matching, no migration or extensions required.'
ogImage: ''
updatedOn: '2026-04-15T00:00:00+00:00'
enableTableOfContents: true
previousLink:
  title: 'PostgreSQL 19 New Features'
  slug: 'postgresql-19-new-features'
nextLink:
  title: 'PostgreSQL 19 ON CONFLICT DO SELECT'
  slug: 'postgresql-19/on-conflict-do-select'
---

**Summary**: PostgreSQL 19 adds SQL/PGQ (Property Graph Queries) based on the SQL:2023 standard. You can define graph structures over your existing relational tables and query them with pattern matching syntax - no new storage engine, no extensions, no data migration.

## Introduction to SQL/PGQ

Graph databases like Neo4j have a clear advantage when querying relationships: "find friends of friends" or "detect circular dependencies" reads naturally in a graph query language. But running a separate graph database alongside PostgreSQL means data duplication, sync complexity, and another system to manage.

PostgreSQL 19 brings graph query capabilities directly into SQL with the SQL/PGQ standard (ISO/IEC 9075-16:2023). The key design decision: property graphs are defined as views over existing relational tables. Your data stays where it is. You just tell PostgreSQL which tables represent nodes and which represent edges.

## Defining a Property Graph

A property graph has two components: vertex tables (nodes) and edge tables (relationships). You create a graph definition that maps these to your existing tables.

### Sample Tables

```sql
-- These are regular PostgreSQL tables
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT,
    joined_at DATE DEFAULT CURRENT_DATE
);

CREATE TABLE follows (
    id SERIAL PRIMARY KEY,
    follower_id INT NOT NULL REFERENCES users(id),
    followed_id INT NOT NULL REFERENCES users(id),
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    author_id INT NOT NULL REFERENCES users(id),
    title TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE likes (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id),
    post_id INT NOT NULL REFERENCES posts(id),
    created_at TIMESTAMP DEFAULT now()
);

-- Insert some data
INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com'),
    ('Diana', 'diana@example.com');

INSERT INTO follows (follower_id, followed_id) VALUES
    (1, 2), (1, 3), (2, 3), (3, 4), (4, 1);

INSERT INTO posts (author_id, title) VALUES
    (1, 'Getting Started with PostgreSQL'),
    (2, 'Graph Queries in SQL'),
    (3, 'Why I Switched from Neo4j');

INSERT INTO likes (user_id, post_id) VALUES
    (2, 1), (3, 1), (4, 2), (1, 3), (2, 3);
```

### Creating the Graph

```sql
CREATE PROPERTY GRAPH social_graph
  VERTEX TABLES (
    users LABEL person
      PROPERTIES (id, name, email, joined_at),
    posts LABEL post
      PROPERTIES (id, title, created_at)
  )
  EDGE TABLES (
    follows
      SOURCE KEY (follower_id) REFERENCES users (id)
      DESTINATION KEY (followed_id) REFERENCES users (id)
      LABEL follows
      PROPERTIES (created_at),
    likes
      SOURCE KEY (user_id) REFERENCES users (id)
      DESTINATION KEY (post_id) REFERENCES posts (id)
      LABEL liked
      PROPERTIES (created_at)
  );
```

This does not copy any data. It creates a graph definition (similar to a view) that PostgreSQL uses to translate graph queries into relational operations on your existing tables.

## Querying with GRAPH_TABLE and MATCH

The `GRAPH_TABLE` function takes a graph name and a `MATCH` pattern, and returns a result set:

### Find Who Alice Follows

```sql
SELECT * FROM GRAPH_TABLE (social_graph
    MATCH (a IS person WHERE a.name = 'Alice')
          -[f IS follows]->(b IS person)
    COLUMNS (b.name AS followed_name)
);
```

```
 followed_name
---------------
 Bob
 Charlie
```

The arrow syntax `-[f IS follows]->` means a directed edge of type `follows`. The `(a IS person)` matches nodes with the `person` label.

### Find Friends of Friends

```sql
SELECT * FROM GRAPH_TABLE (social_graph
    MATCH (a IS person WHERE a.name = 'Alice')
          -[IS follows]->(b IS person)
          -[IS follows]->(c IS person)
    COLUMNS (
        b.name AS friend,
        c.name AS friend_of_friend
    )
);
```

```
 friend  | friend_of_friend
---------+------------------
 Bob     | Charlie
 Charlie | Diana
```

Multi-hop patterns are expressed by chaining edges. Each arrow represents one hop.

### Find Who Liked Alice's Posts

```sql
SELECT * FROM GRAPH_TABLE (social_graph
    MATCH (author IS person WHERE author.name = 'Alice')
          <-[IS liked]-(liker IS person)
          ,
          (author)-[wrote IS liked]->(p IS post)
    COLUMNS (
        liker.name AS liked_by,
        p.title AS post_title
    )
);
```

The `<-[IS liked]-` syntax reverses the edge direction, matching incoming relationships.

### Combining Graph Queries with Regular SQL

Since `GRAPH_TABLE` returns a regular result set, you can use it anywhere you would use a subquery or table:

```sql
-- Find users followed by more than 2 people
SELECT followed_name, count(*) AS follower_count
FROM GRAPH_TABLE (social_graph
    MATCH (a IS person)-[IS follows]->(b IS person)
    COLUMNS (b.name AS followed_name)
)
GROUP BY followed_name
HAVING count(*) > 1
ORDER BY follower_count DESC;
```

This is one of the key advantages over a separate graph database: graph queries compose naturally with aggregation, window functions, CTEs, and everything else in SQL.

## How It Works Under the Hood

When you run a `GRAPH_TABLE` query, PostgreSQL's rewriter translates the graph pattern into standard relational operations (joins, filters, projections) on the underlying tables. The execution plan is a normal PostgreSQL plan - the optimizer can use indexes, parallel queries, and all the usual strategies.

You can verify this with `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM GRAPH_TABLE (social_graph
    MATCH (a IS person WHERE a.name = 'Alice')
          -[IS follows]->(b IS person)
    COLUMNS (b.name AS followed_name)
);
```

The plan will show joins between the `users` and `follows` tables, using whatever indexes are available. There is no special graph execution engine.

## Managing Property Graphs

### Listing Graphs

```sql
-- In psql
\dG

-- Or query the catalog
SELECT * FROM pg_property_graph;
```

### Altering a Graph

```sql
-- Add a new edge type
ALTER PROPERTY GRAPH social_graph
  ADD EDGE TABLE messages
    SOURCE KEY (sender_id) REFERENCES users (id)
    DESTINATION KEY (receiver_id) REFERENCES users (id)
    LABEL sent_message;
```

### Dropping a Graph

```sql
DROP PROPERTY GRAPH social_graph;
```

Dropping a graph only removes the graph definition. The underlying tables are not affected.

## Practical Use Cases

### Dependency Tracking

```sql
CREATE TABLE packages (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    version TEXT
);

CREATE TABLE depends_on (
    id SERIAL PRIMARY KEY,
    package_id INT REFERENCES packages(id),
    dependency_id INT REFERENCES packages(id)
);

CREATE PROPERTY GRAPH dep_graph
  VERTEX TABLES (packages LABEL pkg PROPERTIES (id, name, version))
  EDGE TABLES (
    depends_on
      SOURCE KEY (package_id) REFERENCES packages(id)
      DESTINATION KEY (dependency_id) REFERENCES packages(id)
      LABEL depends
  );

-- Find direct dependencies
SELECT * FROM GRAPH_TABLE (dep_graph
    MATCH (a IS pkg WHERE a.name = 'my-app')
          -[IS depends]->(b IS pkg)
    COLUMNS (b.name AS dependency, b.version)
);
```

### Organization Hierarchy

```sql
-- Find who reports to whom (2 levels)
SELECT * FROM GRAPH_TABLE (org_graph
    MATCH (emp IS employee)
          -[IS reports_to]->(mgr IS employee)
          -[IS reports_to]->(dir IS employee)
    COLUMNS (
        emp.name AS employee,
        mgr.name AS manager,
        dir.name AS director
    )
);
```

### Fraud Detection

```sql
-- Find accounts that share the same phone number as a flagged account
SELECT * FROM GRAPH_TABLE (fraud_graph
    MATCH (flagged IS account WHERE flagged.status = 'flagged')
          -[IS uses]->(phone IS phone_number)
          <-[IS uses]-(other IS account)
    COLUMNS (
        flagged.id AS flagged_account,
        phone.number,
        other.id AS connected_account
    )
);
```

## Current Limitations

The initial SQL/PGQ implementation in PostgreSQL 19 has some intentional limitations:

**No variable-length paths**: You cannot write patterns like `-[IS follows]->+` (one or more hops) or `-[IS follows]->{2,5}` (2 to 5 hops). Each hop must be explicitly written. This means recursive traversals (shortest path, transitive closure) still require recursive CTEs.

**No quantified path patterns**: The `*`, `+`, and `{m,n}` quantifiers are planned for a follow-up patch but are not included in the initial release.

**Fixed depth only**: Queries like "find all paths between A and B regardless of length" are not possible with SQL/PGQ alone in PostgreSQL 19. Use recursive CTEs for those.

For fixed-depth relationship queries (friend-of-friend, 2-3 hop patterns, direct dependency lookups), SQL/PGQ works well today. Variable-length paths are expected in a future PostgreSQL release.

## SQL/PGQ vs. Other Graph Solutions

| | SQL/PGQ (PG 19) | Apache AGE | Neo4j |
|---|---|---|---|
| Language | SQL standard (ISO 2023) | Cypher via extension | Cypher |
| Storage | Existing tables | Extension storage | Native graph DB |
| Variable-length paths | Not yet | Yes | Yes |
| Shortest path | Not yet | Limited | Yes |
| Installation | Built-in | Extension required | Separate database |
| Standards-based | Yes (SQL:2023) | No | No |
| Indexes | Uses existing PG indexes | Own indexes | Own indexes |
| Full SQL support | Yes (joins, CTEs, window functions) | Limited SQL interop | No SQL |

The biggest advantage of SQL/PGQ: it works on your existing tables with your existing indexes. No data migration, no extension installation, no separate system to maintain.

## Summary

SQL/PGQ is one of the most significant features in PostgreSQL 19. It brings standards-based graph query capabilities to PostgreSQL without requiring data migration, extensions, or a separate database. The initial implementation covers fixed-depth pattern matching, which handles many common graph query use cases. Variable-length path support is expected in a future release. The feature was committed on March 16, 2026 by Peter Eisentraut and Ashutosh Bapat.
