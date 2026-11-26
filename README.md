# Public Transit Pathfinding & Bottleneck Analysis: London Underground on Neo4j Graph Data Science

[![Graph DB](https://img.shields.io/badge/Graph%20DB-Neo4j-4581C3?logo=neo4j)](https://neo4j.com/)
[![GDS](https://img.shields.io/badge/Algorithms-Neo4j%20GDS-018BFF?logo=neo4j)](https://neo4j.com/docs/graph-data-science/)
[![Query](https://img.shields.io/badge/Query-Cypher-008CC1)](https://neo4j.com/docs/cypher-manual/)
[![Notebook](https://img.shields.io/badge/Analysis-Jupyter-F37626?logo=jupyter)](https://jupyter.org/)

A graph analytics project that models the **London Underground** as a property graph in **Neo4j** and uses the **Graph Data Science (GDS)** library to answer transit-planning questions: what is the fastest route between two stations, which stations are structural bottlenecks, how the network naturally clusters, and what happens to journey times when a critical interchange closes. The graph is built from three open CSV datasets (**80 stations, 11 lines, 338 route connections**) and analysed with **Dijkstra**, **Betweenness Centrality**, **PageRank** and **Louvain**, followed by a targeted resilience simulation.

**Authors:** Lucia Shumba, Alwyn Munatsi, Lance Midzi
**Course:** Spring 2026 Capstone — Final Project · **Instructor:** James Topor

---

## 1. Analytical Architecture & End-to-End Flow

### End-to-End Execution Flow
$$\text{CSV files} \xrightarrow{\text{LOAD CSV}} \text{Neo4j property graph} \xrightarrow{\text{Cypher}} \text{EDA + analytical queries} \xrightarrow{\text{gds.graph.project}} \text{In-memory projections} \xrightarrow{\text{GDS algorithms}} \text{Scores written to nodes} \xrightarrow{} \text{Resilience simulation}$$

1. **Source CSVs:** Stations (with coordinates and fare zone), lines (with colours) and station-to-station connections (with travel time in minutes) are placed in Neo4j's `import` directory.
2. **Schema & ingestion (Part 1):** A uniqueness constraint and name index are created, then nodes and relationships are loaded with idempotent `MERGE` statements.
3. **Exploratory analysis (Part 2):** Counts, zone distribution, degree, average travel time per line, interchanges, network diameter and terminal stations.
4. **Analytical Cypher (Part 3):** Zone boundary crossings, terminal-station isolation, clustering coefficients, redundant track segments and fastest access to Zone 1.
5. **GDS algorithms (Part 4):** Undirected, time-weighted projections feed Dijkstra, Betweenness, PageRank and Louvain; scores are written back to `Station` nodes.
6. **Resilience simulation (Part 5):** The top-5 critical stations are removed one at a time to measure the longest affected paths and the minimum detour cost.

---

## 2. Graph Data Model

```text
(:Station {id, name, latitude, longitude, zone})
      -[:CONNECTION {line, time}]->
(:Station)

(:Station) -[:SERVED_BY]-> (:Line {id, name, colour, stripe})
```

| Element | Count / Details | Purpose |
| :--- | :--- | :--- |
| **`Station`** nodes | 80, with coordinates and fare zone | Network locations |
| **`Line`** nodes | 11 Tube lines | Line metadata (name, colour, stripe) |
| **`CONNECTION`** relationships | 338 route connections, stored in both directions, weighted by `time` | Physical track segments |
| **`SERVED_BY`** relationships | Derived from connections | Which lines serve each station (interchange detection) |

Properties written back by GDS: `betweenness_score`, `pagerank_score`, `community_id`.

---

## 3. Methodology Choices

| Question | Technique | Why This Choice |
| :--- | :--- | :--- |
| Fastest route between two stations | **Dijkstra (weighted by `time`)** | Minimises travel minutes, not hop count |
| Which stations are bottlenecks? | **Betweenness Centrality** | Counts how often a station lies on shortest paths between all other pairs |
| Which stations are most influential? | **PageRank** (damping 0.85) | Rewards stations connected to other important stations |
| A single "criticality" ranking | **Normalised Betweenness × normalised PageRank** | Combines flow-through and influence into one score |
| How does the network cluster? | **Louvain (weighted by `time`)** | Finds communities by maximising modularity, with no line or zone labels as input |
| What breaks if a hub closes? | **Constrained `shortestPath` with `NONE(...)`** | Re-routes around a removed station and measures the detour |
| Projection orientation | **Undirected** | Tube segments are traversable both ways |

---

## 4. Part 1: Schema Setup & Data Ingestion

```cypher
// Uniqueness constraint and lookup index
CREATE CONSTRAINT station_id_unique IF NOT EXISTS
FOR (s:Station) REQUIRE s.id IS UNIQUE;

CREATE INDEX station_name_index IF NOT EXISTS
FOR (s:Station) ON (s.name);

// Stations
LOAD CSV WITH HEADERS FROM 'file:///london_stations.csv' AS row
MERGE (s:Station {id: toInteger(row.id)})
SET s.name      = row.name,
    s.latitude  = toFloat(row.latitude),
    s.longitude = toFloat(row.longitude),
    s.zone      = toInteger(row.zone);

// Connections (forward; a second pass loads the reverse direction)
LOAD CSV WITH HEADERS FROM 'file:///london.connections.csv' AS row
MATCH (s1:Station {id: toInteger(row.station1)})
MATCH (s2:Station {id: toInteger(row.station2)})
MERGE (s1)-[r1:CONNECTION {line: toInteger(row.line)}]->(s2)
SET r1.time = toFloat(row.time);

// Which lines serve each station
MATCH (s:Station)-[r:CONNECTION]->()
MATCH (l:Line {id: r.line})
MERGE (s)-[:SERVED_BY]->(l);
```

---

## 5. Part 2 & 3: Exploratory Analysis and Analytical Cypher

| Analysis | What It Reveals |
| :--- | :--- |
| Fare zone distribution, station degree, stations per line | Network shape and density |
| Average / min / max travel time per line | Line-level speed profile |
| Interchange stations (3+ lines) | Major transfer hubs |
| Network diameter via `shortestPath` | Longest minimum journey in hops and minutes |
| Terminal stations (degree 1) and isolation tests | Single points of failure at the network edge |
| Local clustering coefficient proxy | How interconnected each station's neighbours are |
| Station pairs served by more than one line | Redundant / parallel track segments |
| Fastest and slowest access to Zone 1 by outer zone | Accessibility gaps |

---

## 6. Part 4: Graph Data Science Algorithms

### Weighted projection and Dijkstra
```cypher
CALL gds.graph.project(
  'tube-weighted', 'Station',
  { CONNECTION: { type: 'CONNECTION', orientation: 'UNDIRECTED', properties: ['time'] } }
);

MATCH (src:Station {name: 'Euston'}), (dst:Station {name: 'Hammersmith'})
CALL gds.shortestPath.dijkstra.stream('tube-weighted', {
  sourceNode: src, targetNode: dst, relationshipWeightProperty: 'time'
})
YIELD totalCost, nodeIds, costs
RETURN totalCost AS total_minutes,
       [nodeId IN nodeIds | gds.util.asNode(nodeId).name] AS ordered_route;
```
Routes analysed include Euston → Hammersmith, Canary Wharf → Paddington, Putney Bridge → Finsbury Park and single-source Dijkstra from King's Cross St Pancras.

### Betweenness, PageRank and Louvain (write mode)
```cypher
CALL gds.betweenness.write('tube-weighted', { writeProperty: 'betweenness_score' });

CALL gds.pageRank.write('tube-pagerank', {
  dampingFactor: 0.85, tolerance: 0.0000001,
  minIterations: 50, maxIterations: 100,
  writeProperty: 'pagerank_score'
});

CALL gds.louvain.write('tube-louvain', {
  writeProperty: 'community_id', relationshipWeightProperty: 'time'
});
```

### Composite criticality score
```cypher
MATCH (s:Station)
WITH max(s.betweenness_score) AS max_b, max(s.pagerank_score) AS max_p
MATCH (s:Station)
WITH s, s.betweenness_score / max_b AS norm_b, s.pagerank_score / max_p AS norm_p
RETURN s.name AS station, s.zone AS zone,
       round(norm_b * norm_p, 6) AS criticality_score
ORDER BY criticality_score DESC LIMIT 10;
```

---

## 7. Part 5: Network Resilience Simulation

For each of the top-5 betweenness stations (including Green Park, Waterloo, High Street Kensington and King's Cross St Pancras), the notebook finds the longest shortest path that passes through the station, then re-routes the same journey around it:

```cypher
MATCH (s1:Station {name: 'Clapham South'}), (s2:Station {name: 'Wembley Central'})
MATCH path = shortestPath((s1)-[:CONNECTION*]-(s2))
WHERE NONE(n IN nodes(path) WHERE n.name = 'Green Park')
RETURN [n IN nodes(path) | n.name] AS detour_route;
```

---

## 8. Key Findings

| Finding | Detail |
| :--- | :--- |
| **Most critical station** | **Green Park** — composite criticality score eight times the network average, with three major line corridors converging at one interchange |
| **Zone 1 dependence** | All **five** highest-centrality stations are in Zone 1, so outer-zone passengers have no structural fallback when a central hub fails |
| **Accidental resilience** | **32 redundant track segments**, concentrated in Zones 1 and 2 |
| **Natural communities** | Louvain found **16 communities** whose boundaries closely mirror TfL's own operational corridors, without being given lines or zones |
| **Closure impact** | Closing Green Park or Waterloo adds **4–15 minutes** per affected journey; no single interchange can absorb all diverted flows |

---

## 9. Repository Structure

```text
London-Underground-Pathfinding/
├── README.md
├── Logistics Pathfinding Project for London Underground Network.ipynb   <-- Full analysis (Parts 1–5)
└── London Undeground Transit Network Intelligence.pptx                  <-- Presentation slides
```

| Notebook Part | Title | Contents |
| :--- | :--- | :--- |
| 1 | Data Ingestion | Constraints, indexes, stations, lines, connections, `SERVED_BY` |
| 2 | Exploratory Data Analysis | Counts, zones, degree, travel times, interchanges, diameter, single points of failure |
| 3 | Analytical Cypher Queries | Zone crossings, terminal isolation, clustering, redundant segments, Zone 1 access |
| 4 | GDS Algorithm Application | Dijkstra, Betweenness, PageRank, composite criticality, Louvain |
| 5 | Network Resilience Simulation | Adjacent-station tests, cascading failure and detour cost for the top-5 stations |

---

## 10. Data Sources

| File | Contents |
| :--- | :--- |
| `london_stations.csv` | Station ID, name, latitude, longitude, fare zone |
| `london_lines.csv` | Line ID, name, colour, stripe |
| `london.connections.csv` | Station pairs, line ID, travel time (minutes) |

The CSVs are not stored in this repository; place them in Neo4j's `import` directory before running Part 1.

---

## 11. Complete Step-by-Step Reproduction Guide

### Step 1: Prepare Neo4j
1. Install **Neo4j Desktop 2.x** and create a new database.
2. Install the **Graph Data Science** plugin for that database and start it.

### Step 2: Load the Data Files
Copy `london_stations.csv`, `london_lines.csv` and `london.connections.csv` into the database's `import` folder.

### Step 3: Set Up the Notebook Environment
```bash
pip install neo4j jupyter
jupyter notebook
```

### Step 4: Run the Analysis
1. Open `Logistics Pathfinding Project for London Underground Network.ipynb`.
2. Run Part 1 to create the schema and load the graph; confirm 80 stations and 11 lines.
3. Run Parts 2–3 for exploratory and analytical queries.
4. Run Part 4 to project the graph and write algorithm scores back to the nodes (each section drops its projection when done).
5. Run Part 5 for the resilience simulation.

> **Note:** The notebook mixes Cypher cells (run in Neo4j Browser) with Python cells (run via the Jupyter kernel and the `neo4j` driver). Follow the in-notebook instructions for which environment each cell uses.

### Step 5: Review the Presentation
`London Undeground Transit Network Intelligence.pptx` summarises the findings for a non-technical audience.
