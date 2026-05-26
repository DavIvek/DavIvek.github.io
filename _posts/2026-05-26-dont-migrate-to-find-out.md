---
title: You don't have to migrate to find out if graph queries fit your data
date: 2026-05-26 10:00:00 +0200
categories: [Blog Posts]
tags: [graph databases, postgres, gql, memgql, federation]
description: A 30-minute way to test it against the database you already run.
---


## The three-self-join smell

If you've worked with a Postgres database for any length of time, you've probably written a query that looked like this:

```sql
SELECT c.email
FROM customers c
JOIN customer_referrals r1 ON r1.referrer_id = c.id
JOIN customers r ON r.id = r1.referee_id
JOIN customer_referrals r2 ON r2.referrer_id = r.id
JOIN customers r3 ON r3.id = r2.referee_id
WHERE c.id = $1;
```

"Find everyone two referral hops away from this user."

Or this:

```sql
WITH RECURSIVE chain AS (
    SELECT id, parent_id, 1 AS depth
    FROM incidents WHERE id = $1
    UNION ALL
    SELECT i.id, i.parent_id, c.depth + 1
    FROM incidents i JOIN chain c ON i.id = c.parent_id
    WHERE c.depth < 10
)
SELECT * FROM chain;
```

"Walk this incident chain back to the root cause."

These are graph-shaped questions. You write them as SQL because that's the language your database speaks. Recursive CTEs and chains of self-joins are how relational systems express traversal, and they work, until they don't. Performance gets weird past two or three hops. Readability craters. Your codebase fills up with SQL idioms that hide the business logic. And the moment someone asks "who else is connected to this node within N hops?", a question a graph database answers in one line, you start wondering if you're using the wrong tool.

The standard advice when you reach this point is: **use a graph database**. Then comes the migration.

## The migration tax

Adopting a graph database isn't free, and most teams know it. The bill, roughly:

- **An ETL pipeline.** Your facts live in Postgres. The graph database needs its own copy. Now you have two sources of truth and a sync job to maintain.
- **A dual-schema problem.** Your ORM, your dashboards, your analytics, they all still want SQL. Your graph-traversal queries want Cypher or GQL. Every change to your domain model has to land in both places.
- **A new operational surface.** Backup strategy, on-call rotation, monitoring, capacity planning, version upgrades, all for a second database.

There's a second, less obvious tax: **the cost of finding out whether a graph database actually helps your workload has historically been roughly the cost of committing to one**. Most engineering leads, looking at that bill, quietly decide it isn't worth a maybe. They keep writing recursive CTEs.

## What changed

The premise of this post is that the cost of finding out has collapsed. Federated GQL engines now let you point a standard graph query interface at your existing relational database without copying data anywhere. You connect to a Bolt port and write GQL queries. Behind that port, the engine translates `MATCH (a:Package)-[:DEPENDS_ON]->(b)` into the same SQL JOINs you would have written by hand, except it's the engine writing them, not you.

[MemGQL](https://memgraph.com/blog/memgql-zero) is one such engine. It speaks the ISO/IEC 39075 GQL standard, accepts a small JSON file that maps your tables to graph nodes and edges, and exposes everything over the standard Bolt protocol. Any Bolt client connects without modification.

**The price of asking "do graph queries fit my workload?" is now an afternoon, not a quarter.**

## What this looks like in practice

Let's walk through a complete example: a CVE just dropped against `cool-parser-3`, a third-party library. You need to figure out which of your packages are affected (directly or transitively), how deep the dependency chain goes, and which teams need to be paged tonight. **Everything lives in Postgres**, no graph database anywhere in the stack. You reach all of it through one MemGQL Bolt port.

### 1. The Postgres schema

Two entity tables and two junction tables, exactly the kind of schema a real internal inventory looks like:

```sql
CREATE TABLE packages (
    id INTEGER PRIMARY KEY, name TEXT, ecosystem TEXT,
    maintainer_email TEXT, team TEXT, in_production INTEGER
);
CREATE TABLE cves (
    id INTEGER PRIMARY KEY, cve_id TEXT, severity TEXT,
    description TEXT, published_at TEXT
);
-- Junction tables that back the graph edges.
CREATE TABLE dependencies (id INTEGER PRIMARY KEY, package_id INTEGER, depends_on_id INTEGER);
CREATE TABLE cve_affects  (id INTEGER PRIMARY KEY, cve_id INTEGER,     package_id INTEGER);
```

`dependencies` will be the `(:Package)-[:DEPENDS_ON]->(:Package)` edge; `cve_affects` will be `(:Cve)-[:AFFECTS]->(:Package)`. The seeded data has ten packages with `cool-parser-3` at the root of a small dependency tree, plus one CVE row that points at it.

### 2. The graph mapping

A short JSON file tells MemGQL how to read those tables as a graph. One block per node label, one per edge type:

```json
{
  "nodes": [
    { "label": "Package", "table": "packages", "id_column": "id",
      "properties": { "name": "name", "team": "team",
                      "in_production": "in_production",
                      "maintainer_email": "maintainer_email" } },
    { "label": "Cve", "table": "cves", "id_column": "id",
      "properties": { "cve_id": "cve_id", "severity": "severity",
                      "description": "description" } }
  ],
  "edges": [
    { "rel_type": "DEPENDS_ON", "table": "dependencies", "id_column": "id",
      "source_column": "package_id", "target_column": "depends_on_id",
      "source_label": "Package", "target_label": "Package" },
    { "rel_type": "AFFECTS", "table": "cve_affects", "id_column": "id",
      "source_column": "cve_id", "target_column": "package_id",
      "source_label": "Cve", "target_label": "Package" }
  ]
}
```

Nothing about this file is MemGQL-specific philosophy, it's just a declaration: "these tables are nodes; these junction tables are edges." Every node and edge declares its identity column via `id_column`; for edges, that column is what the engine carries through variable-length traversal to enforce trail semantics (no edge visited twice in the same path). Real schemas need maybe ten more lines than this; the principle scales.

### 3. Starting MemGQL

One `docker run` brings up MemGQL pointed at your existing Postgres, with the mapping mounted in:

```bash
docker run -d --name memgql -p 7688:7688 \
    -e CONNECTOR_TYPE=postgres \
    -e POSTGRES_URL="host=<your-pg-host> user=<user> password=<pw> dbname=<db>" \
    -e MAPPING_FILE=/data/supply.json \
    -e BOLT_LISTEN_ADDR=0.0.0.0:7688 \
    -v "$PWD/mappings/supply.json:/data/supply.json" \
    memgraph/memgql:latest
```

That's the entire catalog setup. Three env vars do the work.

### 4. Connecting

`mgconsole` is the standard interactive client for Bolt-protocol servers. Point it at MemGQL and you're at a GQL prompt against your Postgres data:

```bash
mgconsole --host localhost --port 7688
```

That's it. No driver to install, no dual ORM, no second connection pool. From here, every query that walks the dependency graph gets to use `MATCH` instead of `WITH RECURSIVE`.

### 5. The triage

Two queries answer the whole question. **First, look up the CVE itself**:

```cypher
MATCH (c:Cve WHERE c.cve_id = 'CVE-2026-0042')-[:AFFECTS]->(p:Package)
RETURN c.severity, c.description, p.name;
```

Result (a single row):

```
severity | description                                                  | name
"high"   | "Remote code execution via crafted input in cool-parser-3..." | "cool-parser-3"
```

**Then, walk the dependency graph 1–3 hops from the affected package, in one query**:

```cypher
MATCH (p:Package) (-[:DEPENDS_ON]->()){1,3}
      (target:Package WHERE target.name = 'cool-parser-3')
RETURN p.name, p.team, p.in_production, p.maintainer_email
ORDER BY p.name;
```

Result (six rows: `json-helper` and `request-logger` at 1 hop, `auth-middleware` and `our-api-server` at 2, `mobile-client` and `our-web-frontend` at 3):

```
name                | team           | in_production | maintainer_email
"auth-middleware"   | "platform"     | 1             | "platform@acme.io"
"json-helper"       | "shared-libs"  | 1             | "shared-libs@acme.io"
"mobile-client"     | "mobile"       | 1             | "mobile-team@acme.io"
"our-api-server"    | "platform"     | 1             | "platform@acme.io"
"our-web-frontend"  | "web"          | 1             | "web-team@acme.io"
"request-logger"    | "ops"          | 0             | "ops-team@acme.io"
```

That's the full triage in two queries, against data that never left Postgres: five production packages, four teams to page (platform, shared-libs, web, mobile).

## MemGQL is more than a Postgres front-end

We just walked through a Postgres example because it's the most common starting point. The same engine speaks to nine backends today, and the workflow is identical for all of them. Swap `CONNECTOR_TYPE` and the rest of the setup (mapping JSON, Bolt port, GQL queries) doesn't change:

- **Relational**: PostgreSQL, MySQL, Oracle, DuckDB
- **Analytical / lakehouse**: ClickHouse, Apache Iceberg (via Trino), Apache Pinot
- **Graph passthrough**: Memgraph, Neo4j

### Multi mode

The single-connector launch shown above is the simplest path: one container, one backend, one mapping. There's also a **multi mode**, where a single MemGQL process holds several backends at once and you manage them with MemGQL DSL statements at the prompt:

```cypher
ADD MAPPING  inventory FROM 'inventory.json';
ADD MAPPING  events    FROM 'events.json';
ADD CONNECTOR pg    TYPE postgres   URI '...' MAPPING inventory;
ADD CONNECTOR ch    TYPE clickhouse URI '...' MAPPING events;
CONNECT pg AS pg_conn;
CONNECT ch AS ch_conn;

USE CONNECTION pg_conn MATCH (p:Package) RETURN p;
USE CONNECTION ch_conn MATCH (e:Event)   RETURN e;
```

One Bolt endpoint, multiple databases behind it, each addressed by name.

### Coming soon: cross-backend queries

The obvious follow-up is "can I write one query that touches both?". Imagine joining your live Postgres orders with a ClickHouse clickstream in a single `MATCH`, with no ETL job between them: MemGQL splits the work, ships sub-queries to each backend, and joins the results inside the engine.

That's exactly the work currently in flight, and it lands as the federation layer in the next release. Stay tuned.
