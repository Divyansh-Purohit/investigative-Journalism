# PEPs & Offshore Shell Companies: Knowledge Graph Pipeline

Knowledge graph for investigative journalism that merges the ICIJ Offshore Leaks database with OpenSanctions Politically Exposed Persons data and consolidated sanctions/warrants records to expose multi-hop financial connections between public officials and offshore shell companies.

Built as a group project for the **Knowledge Engineering** course. 



**Client role:** Anti-corruption NGO / investigative journalism group auditing conflicts of interest and hidden wealth among global public officials.

**Why a knowledge graph?** Relational databases struggle with the deep, recursive connections between public figures and secretive offshore shell companies. A knowledge graph that merges public entity data with offshore financial leaks allows efficient pattern-matching to expose these hidden networks.


## Research Questions

1. Which publicly known politicians or government officials are directly or indirectly connected to offshore financial entities within **three degrees of separation**?
2. What are the most common **intermediary nodes** (e.g., specific law firms, wealth managers, or banks) that serve as the structural bridge between politically exposed persons and multiple offshore shell companies?


## Data Sources

| # | Dataset | Description | Source |
|---|---------|-------------|--------|
| 1 | **ICIJ Offshore Leaks Database** | Structured nodes and edges detailing the ownership and management of offshore companies, trusts, and funds across five major leaks: Panama Papers (2016), Pandora Papers (2021), Paradise Papers (2017–18), Bahamas Leaks (2016), and Offshore Leaks (2013). Contains 800k+ entities, 750k+ officers, 25k intermediaries, and 3.3M+ relationships. | https://offshoreleaks.icij.org |
| 2 | **OpenSanctions PEP Export (Wikidata-derived)** | ~696k Politically Exposed Persons with political roles, countries of office, family links, and associate relationships, exported in the FollowTheMoney model from Wikidata via OpenSanctions. | https://www.opensanctions.org |
| 3 | **OpenSanctions Consolidated Sanctions** | 70,673 entries of sanctioned persons and organisations across UN, EU, OFAC, and other international sanctions programmes. | https://www.opensanctions.org |
| 4 | **OpenSanctions Warrants & Criminal Entities** | 248,413 entries including Interpol Red Notices and US SAM debarments. | https://www.opensanctions.org |

---

## Graph Statistics

| Metric | Value |
|--------|-------|
| Total nodes (full graph) | 2,107,703 |
| Total edges (enriched, directed) | 2,903,986 |
| Officer nodes | 771,315 |
| Entity nodes | 814,344 |
| Intermediary nodes | 25,629 |
| Position nodes (OpenSanctions) | 91,180 |
| PEP-flagged officer nodes | 73,436 |
| Sanctioned officer nodes | 942 |
| Wanted officer nodes | 1,234 |
| Dual-risk nodes (PEP + sanctioned) | 581 |
| 3-hop PEP subgraph size | 522,919 nodes |

---

## Pipeline Overview

```
ICIJ Offshore Leaks CSVs          OpenSanctions PEP Export
        │                                    │
        ▼                                    ▼
  Load & clean nodes            Load persons, positions,
  and relationships             occupancy, family, associates
        │                                    │
        └──────────────┬─────────────────────┘
                       ▼
            Entity Resolution
        (name similarity 55% +
         country overlap 45%)
         → 73,436 PEP-matched officers
                       │
                       ▼
            Knowledge Graph (NetworkX DiGraph)
            2M+ nodes · 2.9M+ edges
                       │
              ┌────────┴─────────┐
              ▼                  ▼
     OpenSanctions          OpenSanctions
     Consolidated           Warrants &
     Sanctions              Criminal Entities
     (is_sanctioned)        (is_wanted)
              │                  │
              └────────┬─────────┘
                       ▼
            3-Hop PEP Subgraph
            (522,919 nodes)
                       │
              ┌────────┴──────────┐
              ▼                   ▼
           Neo4j              Betweenness
         (Cypher              Centrality
          queries)            (NetworkX, k=1000)
```

---

## Key Findings

**RQ1: PEP Connections:** High-confidence PEP-matched officers appear as direct (1-hop) officers of tax-haven shell companies across Panama Papers, Pandora Papers, and Offshore Leaks simultaneously, confirming persistent multi-leak offshore exposure for identified politicians.

**RQ2: Top Bridge Intermediaries (by betweenness centrality on the 3-hop subgraph):**

| Intermediary | Betweenness |
|---|---|
| Mossack Fonseca & Co. (Bahamas) Ltd | 0.2674 |
| Portcullis TrustNet (Samoa) Ltd | 0.0480 |
| Mossack Fonseca & Co. (Singapore) Pte Ltd | 0.0393 |
| Orion House Services (HK) Ltd | 0.0361 |
| Offshore Business Consultant (Int'l) Ltd | 0.0284 |

A betweenness score of 0.267 means 26.7% of all shortest paths in the PEP subgraph pass through Mossack Fonseca (Bahamas) alone.

---

## Repository Structure

```
.
├── project_code.ipynb               # Main pipeline notebook 
├── data/
│   ├── icij/                       # ICIJ Offshore Leaks CSVs (not committed)
│   │   ├── nodes-entities.csv
│   │   ├── nodes-officers.csv
│   │   ├── nodes-intermediaries.csv
│   │   ├── nodes-addresses.csv
│   │   ├── nodes-others.csv
│   │   └── relationships.csv
│   └── wikidata/                   # OpenSanctions PEP export CSVs (not committed)
│       ├── persons.csv
│       ├── positions.csv
│       ├── occupancy.csv
│       ├── associates.csv
│       ├── family.csv
│       └── ownership.csv
├── data_opensanctions/             # OpenSanctions enrichment data (not committed)
│   ├── consolidated sanctions/
│   │   └── targets.simple.csv
│   └── warrants and criminal entities/
│       └── targets.simple.csv
└── output/                                 # Generated on first run — not committed
    ├── fig_icij_distributions.png          # ICIJ node/leak/jurisdiction distribution plots
    ├── pep_matches_v3.csv                  # Entity matching cache (skip re-run on reload)
    ├── pep_matches.csv                     # Enriched matching results with position info
    ├── rq1_pep_offshore_connections.csv    # RQ1 query results (PEP-entity pairs, hop distance)
    └── betweenness_pep_subgraph.csv        # RQ2 betweenness centrality scores (all nodes)
```

---

## Setup

### Requirements

```bash
pip install -r requirements.txt
```

### Data

All input data is bundled in a single archive. Download and extract it into the root of this repository, keeping the directory structure as-is:

**[Download data.zip from Google Drive](https://drive.google.com/file/d/1MwY2ws_jWnu7E-3TgQM-rbk55nJofj6I/view?usp=sharing)**

After extracting, your directory should contain:
```
data/
├── icij/
└── wikidata/
data_opensanctions/
├── consolidated sanctions/
└── warrants and criminal entities/
```

Original data sources for reference:
- **ICIJ data:** https://offshoreleaks.icij.org/pages/database
- **OpenSanctions PEP export:** https://www.opensanctions.org/datasets/wd_peps/
- **OpenSanctions Consolidated Sanctions:** https://www.opensanctions.org/datasets/sanctions/
- **OpenSanctions Warrants:** https://www.opensanctions.org/datasets/interpol_red_notices/

### Neo4j

Start a local Neo4j instance (version 5.x) and update the credentials in the notebook:

```python
NEO4J_URI  = "bolt://localhost:7687"
NEO4J_USER = "neo4j"
NEO4J_PASS = "your_password"
```

### Run

Open `project_code.ipynb` and run all cells in order. On first run, entity matching takes approximately 20–30 minutes with 10 parallel jobs. Subsequent runs load from the `pep_matches_v3.csv` cache automatically.

Betweenness centrality on the 522,919-node subgraph takes approximately 11 minutes on an Apple M4 Pro (k=1000 approximate algorithm).

---

## Graph Schema

```
[OpenSanctions PEP]  ──entity resolution──►  [:Officer:PEP]
                                                    │
                                          :OFFICER_OF│
                                                    ▼
[:Intermediary] ──:INTERMEDIARY_OF──►  [:Entity]  ◄──:SAME_NAME_AS──  [:Entity]
                                            │
                                 :REGISTERED_ADDRESS│
                                            ▼
                                        [:Address]

[:Officer:PEP] ──:HOLDS_POSITION──► [:Position]
[:Officer:PEP] ──:FAMILY_OF──────►  [:Officer]
[:Officer:PEP] ──:ASSOCIATES_WITH── [:Officer]
```

**Node properties of note:**
- `is_pep` — matched to a known politician at high confidence
- `pep_match_score` — combined name + country matching score (0–100)
- `is_tax_haven` — entity or officer in a known secrecy jurisdiction
- `is_sanctioned` — matched to OpenSanctions consolidated sanctions
- `is_wanted` — matched to Interpol Red Notice or US SAM debarment


---

## Authors

Divyansh Purohit
Prathamesh Samal
Arthur Johannes Sliwinski
Naga Dheeraj Mukkara
