# Graph Databases

## What it is

A **graph database** stores data as **nodes** (entities), **edges** (relationships), and **properties** (attributes on nodes/edges). It is designed for data that is fundamentally relational in nature — where the connections between entities are as important as the entities themselves.

Graph databases excel at **traversal queries**: "find all friends of friends of Alice who also like Jazz within 3 hops" — queries that explode in complexity and performance cost when modeled as relational JOIN chains.

**Common systems**: Neo4j, Amazon Neptune, ArangoDB, TigerGraph, JanusGraph.

---

## How it works

### Data Model

A graph consists of:
- **Nodes**: represent entities (Person, Movie, Product, Account).
- **Edges** (relationships): directed connections between nodes, each with a type (`:FOLLOWS`, `:PURCHASED`, `:TRANSFERRED_TO`).
- **Properties**: key-value attributes on both nodes and edges.

```
(Alice:Person {name: "Alice", age: 30})
       │
  [:FOLLOWS]
       ▼
(Bob:Person {name: "Bob", age: 28})
       │
  [:FOLLOWS]
       ▼
(Charlie:Person {name: "Charlie", age: 35})

(Alice) -[:REVIEWED {rating: 5}]-> (Inception:Movie {title: "Inception"})
(Bob)   -[:REVIEWED {rating: 4}]-> (Inception:Movie)
```

Relationships are first-class citizens with their own properties — not just foreign keys.

### Cypher Query Language (Neo4j)

Neo4j uses **Cypher**, an ASCII-art-style graph query language:

```cypher
// Create nodes and relationship
CREATE (a:Person {name: "Alice"})-[:FOLLOWS]->(b:Person {name: "Bob"});

// Find all people Alice follows
MATCH (a:Person {name: "Alice"})-[:FOLLOWS]->(b:Person)
RETURN b.name;

// Friends of friends (2 hops)
MATCH (a:Person {name: "Alice"})-[:FOLLOWS*2]->(fof:Person)
WHERE NOT (a)-[:FOLLOWS]->(fof)  // exclude direct friends
RETURN DISTINCT fof.name;

// Shortest path between Alice and Charlie
MATCH path = shortestPath(
  (a:Person {name: "Alice"})-[:FOLLOWS*..6]-(c:Person {name: "Charlie"})
)
RETURN path;

// Common friends of Alice and Bob
MATCH (alice:Person {name: "Alice"})-[:FOLLOWS]->(common),
      (bob:Person {name: "Bob"})-[:FOLLOWS]->(common)
RETURN common.name;
```

The pattern-matching syntax (`(node)-[:RELATIONSHIP]->(other)`) makes traversal queries intuitive.

### Graph Traversal vs SQL Joins

To find friends-of-friends in SQL:

```sql
-- Depth 2 friends in relational DB:
SELECT DISTINCT f2.name
FROM users u
JOIN follows f1 ON f1.follower_id = u.id
JOIN users f  ON f.id = f1.followee_id
JOIN follows f2_rel ON f2_rel.follower_id = f.id
JOIN users f2 ON f2.id = f2_rel.followee_id
WHERE u.name = 'Alice'
  AND f2.id != u.id;
```

For depth 6 or variable-depth traversal, SQL requires recursive CTEs (WITH RECURSIVE) which become both complex and slow at scale.

In a graph database, the traversal follows pointers stored in the graph structure — each hop is an O(1) pointer dereference rather than a join operation that must scan rows.

### Index-Free Adjacency

The key performance advantage of purpose-built graph databases is **index-free adjacency**: each node directly stores physical pointers to its adjacent nodes and edges. Traversal from node to neighbor follows a pointer, not an index lookup.

```
SQL join: hash(user_id) → index lookup → row scan → result [O(log n) per hop]
Graph:    node.next_edge → follow pointer → neighbor node [O(1) per hop]
```

For deep traversals and complex patterns, this is orders of magnitude faster than relational joins.

### Graph Algorithms

Graph databases and graph processing frameworks include built-in algorithms:

| Algorithm | Use case |
|---|---|
| **Shortest Path** (Dijkstra, BFS) | Routing, degrees of separation |
| **PageRank** | Link importance, recommendation scoring |
| **Community Detection** (Louvain) | Group identification, clustering |
| **Betweenness Centrality** | Key node identification in networks |
| **Label Propagation** | Semi-supervised classification |
| **Node Similarity** | Collaborative filtering recommendations |

Neo4j Graph Data Science library, Amazon Neptune Neptune ML.

### Common Use Cases

**Social Networks** — friend recommendations, mutual friends, follow graph:
```cypher
MATCH (user)-[:FOLLOWS]->(f)-[:FOLLOWS]->(fof)
WHERE user.id = $userId AND NOT (user)-[:FOLLOWS]->(fof)
RETURN DISTINCT fof, count(*) AS mutual_friends
ORDER BY mutual_friends DESC LIMIT 10
```

**Fraud Detection** — detecting rings of fraudulent accounts sharing devices, phone numbers, or addresses:
```cypher
MATCH (a:Account)-[:SHARED_DEVICE]->(d:Device)<-[:SHARED_DEVICE]-(b:Account)
WHERE a.flagged = true
RETURN DISTINCT b.account_id, b.status
```

**Knowledge Graphs** — semantic relationships between concepts used in search and AI.

**Access Control / RBAC** — complex permission inheritance through organizational hierarchies.

**Supply Chain** — dependency chains, impact analysis ("if supplier X fails, which products are affected?").

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Natural model for highly-connected data | Not suitable for aggregate/analytical queries |
| Fast variable-depth traversal | Limited horizontal scalability vs Cassandra/Dynamo |
| Index-free adjacency for deep patterns | Relatively small community and ecosystem |
| Intuitive query language for relationships | Not a general-purpose database |
| Built-in graph algorithm library | High memory footprint for large dense graphs |

---

## When to use

Use a graph database when:
- **Relationship depth and patterns** are the primary query paradigm (not simple lookups or aggregations).
- You need **variable-depth traversal** (2+ hops) that would require complex recursive CTEs in SQL.
- Your domain is inherently graph-structured: social networks, recommendation engines, fraud detection, access control, knowledge graphs, logistics networks.
- **Connection patterns matter** as much as entity attributes.

Do not use a graph database as a general-purpose store. Graph databases are a specialized tool for specific relationship-intensive workloads.

---

## Common Pitfalls

- **Using graph DB for non-graph problems**: storing flat, lookup-based data in a graph database adds complexity without benefit. Use graph DB only when traversal is a core query.
- **Ignoring scalability limits**: most graph databases (especially Neo4j community edition) are single-machine. Distributed graph processing at massive scale requires specialized systems (JanusGraph, TigerGraph) or accepting the graph doesn't fit one machine.
- **Super-nodes (hot nodes)**: a node with millions of edges (a celebrity in a social graph, a widely-used category node) becomes expensive to traverse. Implement strategies to limit traversal depth or use edge sampling.
- **Not indexing node properties**: graph databases still need indexes on frequently-queried node properties (`name`, `account_id`). Without them, the query must scan all nodes to find the starting point.
- **Confusing graph DB with property graph in RDBMS**: PostgreSQL's recursive CTE or a graph extension (Apache AGE) can handle limited graph queries. Reserve dedicated graph DBs for genuinely deep, complex traversal workloads.
