# Public Transit Pathfinding and Bottleneck Analysis
### Using Neo4j Graph Data Science (GDS) with the London Underground

**Authors:** Lucia Shumba, Alwyn Munatsi, Lance Midzi  
**Course:** Spring 2026 Capstone — Final Project  
**Instructor:** James Topor

---

## Overview

This project applies graph-theoretic methods to model and analyse the London Underground transit network using Neo4j's Graph Data Science (GDS) library. A property graph was constructed from three open-source CSV datasets covering **80 stations**, **11 lines**, and **338 route connections**, then interrogated using four canonical graph algorithms to reveal structural insights about network performance, resilience, and efficiency.

---

## Key Findings

- **Green Park** is the single most critical station in the network — its composite criticality score is eight times higher than the network average, with three major line corridors converging at a single interchange point.
- All **five highest-centrality stations** are located in Zone 1, meaning outer-zone passengers have no structural fallback when a central hub fails.
- **32 redundant track segments**, concentrated in Zones 1 and 2, provide accidental resilience built up over 160 years of layered infrastructure.
- The Louvain algorithm detected **16 natural communities** whose boundaries closely mirror TfL's own operational corridor structure — discovered without prior knowledge of lines or zones.
- A targeted resilience simulation confirms that closure of Green Park or Waterloo would impose journey-time penalties of **4–15 minutes** per affected journey, with no single interchange capable of absorbing all converging passenger flows.

---

## Project Structure

```
Logistics Pathfinding Capstone/
├── Logistics Pathfinding Project for London Underground Network.ipynb   # Main analysis notebook
├── London Undeground Transit Network Intelligence.pptx                  # Presentation slides
└── README.md
```

The notebook is organised into five parts:

| Part | Title | Description |
|------|-------|-------------|
| 1 | Data Ingestion | Schema setup, loading stations, lines, connections, and SERVED_BY relationships from CSV files into Neo4j |
| 2 | Exploratory Data Analysis | Node/relationship counts, fare zone distribution, degree analysis, average travel times, interchange stations, network diameter, and single points of failure |
| 3 | Analytical Cypher Queries | Zone boundary crossings, terminal stations, parallel/redundant track segments |
| 4 | GDS Algorithm Application | Dijkstra's shortest path, Betweenness Centrality, PageRank, and Louvain community detection |
| 5 | Network Resilience Simulation | Cascading failure analysis and detour cost assessment for the top 5 critical stations |

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| **Neo4j Desktop** | Graph database for storing and querying the London Underground network |
| **Neo4j GDS Library** | Graph Data Science algorithms (Dijkstra, Betweenness Centrality, PageRank, Louvain) |
| **Cypher** | Query language used throughout for data ingestion, EDA, and analysis |
| **Python** (`neo4j` driver) | Programmatic interaction with Neo4j |
| **Jupyter Notebook** | Interactive analysis and documentation environment |

---

## Graph Data Model

```
(:Station {id, name, latitude, longitude, zone})
    -[:CONNECTION {line_id, time}]->
(:Station)

(:Station)
    -[:SERVED_BY]->
(:Line {id, name, colour, stripe})
```

- **80 Station nodes** with geographic coordinates and fare zone
- **11 Line nodes** representing each Tube line
- **CONNECTION** relationships with travel time weights
- **SERVED_BY** relationships linking stations to the lines that serve them

---

## Algorithms Applied

### Dijkstra's Weighted Shortest Path
Finds the minimum travel-time route between any two stations, accounting for the `time` weight on CONNECTION relationships.

### Betweenness Centrality
Measures how frequently a station appears on shortest paths between all other station pairs — identifying structural bottlenecks and critical transfer hubs.

### PageRank
Measures the relative importance of each station based on the importance of stations connected to it, capturing influence propagation through the network.

### Louvain Community Detection
Partitions the network into natural clusters by maximising modularity, revealing 16 operationally coherent geographic communities without using line or zone labels as input.

---

## Data Sources

The graph is built from three CSV files loaded into Neo4j's import directory:

| File | Contents |
|------|---------|
| `london_stations.csv` | Station ID, name, latitude, longitude, fare zone |
| `london_lines.csv` | Line ID, name, colour, stripe |
| `london_connections.csv` | Station pairs, line ID, travel time (minutes) |

---

## Setup & Usage

### Prerequisites

- Neo4j Desktop 2.x with the GDS plugin installed
- Python 3.x with the `neo4j` driver (`pip install neo4j`)
- Jupyter Notebook or JupyterLab

### Running the Analysis

1. Start Neo4j Desktop and create a new database with the GDS library enabled.
2. Place the three CSV files in Neo4j's `import` directory.
3. Open the notebook and execute cells sequentially from Part 1 through Part 5.
4. Part 1 cells use Cypher directly; Parts 4–5 use the Python neo4j driver to call GDS procedures.

> **Note:** Cells in this notebook contain both Cypher queries (run directly in Neo4j Browser) and Python code (run via the Jupyter kernel). Follow the in-notebook instructions for which environment to use for each cell.
