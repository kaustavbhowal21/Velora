# Velora — Intelligent Route Optimisation Software

> **Velora** is a full-stack employee transportation optimisation platform built for corporate cab routing. It uses a multi-start Adaptive Large Neighbourhood Search (ALNS) algorithm on real road network data to assign employees to vehicles, minimise total travel cost and time, and respect each employee's scheduling, vehicle, and sharing constraints.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Algorithm](#algorithm)
- [Project Structure](#project-structure)
- [Setup & Installation](#setup--installation)
- [Running the App](#running-the-app)
- [Input Format](#input-format)
- [Output & Dashboard](#output--dashboard)
- [Configuration](#configuration)
- [Test Cases](#test-cases)
- [Acknowledgements](#acknowledgements)

---

## Overview

Velora solves the **Multi-Trip Vehicle Routing Problem with Time Windows (MTVRPTW)** tailored for corporate employee pick-up scheduling. Given a set of employees with pickup locations and time constraints, and a fleet of vehicles with varying capacities and costs, Velora finds the optimal assignment and routing plan.

**Key capabilities:**

- Assigns employees to vehicles while respecting capacity, vehicle-type preference, and sharing preference constraints
- Handles multiple trips per vehicle across a single shift
- Optimises a configurable weighted objective: `α × cost + β × time`
- Falls back to a relaxed constraint mode if not all employees can be assigned strictly
- Renders results on an interactive dashboard with maps, charts, and constraint health indicators

---

## Architecture

```
Browser (upload.html)
       │
       ▼
Flask App (app.py)
       │
       ├── DataLoader          — Reads Excel input (employees, vehicles, baseline, metadata)
       ├── ProblemState        — Holds all entities + objective function
       ├── TripConstraints     — Checks capacity, vehicle type, and time window feasibility
       ├── InitialSolutionBuilder — Probabilistic PWSA (Clarke-Wright Savings + GRASP)
       ├── ALNS                — Multi-start Adaptive Large Neighbourhood Search
       └── ResultsVerifier     — Validates solution + builds output dict
                │
                ▼
       final.html Dashboard (Leaflet maps + Chart.js)
```

The map graph is built from OpenStreetMap data for Bengaluru and stored as pre-processed CSV files. Routing uses the **A\* algorithm** with a haversine heuristic, and nearest-node lookup uses a **KD-Tree** for fast spatial queries.

---

## Algorithm

Velora uses a **three-phase pipeline** on each ALNS run:

### Phase 1 — Initial Solution (Probabilistic PWSA)

1. **Savings Calculation:** For every pair of employees, compute the Clarke-Wright savings of serving them together vs. individually. Savings are perturbed with ±10% noise for diversity.
2. **Seed Routes (GRASP Step 1):** The top-scoring pairs seed the initial trips.
3. **Regret-k Insertion (GRASP Step 2):** Remaining employees are inserted using regret-based selection from a Restricted Candidate List (RCL), choosing randomly from the top-k options by regret score.
4. **Multi-Trip Consolidation:** Consecutive trips on the same vehicle are merged if feasible.
5. **Final Validation:** All trips are re-checked; infeasible ones are removed and their employees re-inserted.

### Phase 2 — ALNS Optimisation

The builder above is called 25 times to generate diverse starts. Each is then refined by ALNS:

| Component | Detail |
|-----------|--------|
| **Destroy operators** | Random removal, worst removal, trip removal |
| **Repair operators** | Greedy insertion, regret-2 insertion |
| **Acceptance** | Simulated annealing with geometric cooling (`T × 0.9995`) |
| **Priority** | Maximise assigned employees first, then minimise cost |

### Phase 3 — Fallback

If Phase 2 does not assign all employees, constraints are relaxed (vehicle preference → ANY, sharing → TRIPLE) and a second ALNS pass is run. The solution with more assigned employees is kept.

---

## Project Structure

```
velora/
├── app.py                    # Flask application entry point
├── Dockerfile                # Container definition
├── requirements.txt          # Python dependencies
│
├── optimize/                 # Core optimisation package
│   ├── __init__.py
│   ├── main.py               # DataLoader, ProblemState, ResultsVerifier, optimize()
│   ├── models.py             # Dataclasses: Employee, Vehicle, Trip, Solution, …
│   ├── constraints.py        # TripConstraints — capacity, type, time checks
│   ├── initial_solution.py   # InitialSolutionBuilder (PWSA + GRASP)
│   ├── alns_engine.py        # ALNS, Destroy & Repair operators
│   └── mapgraph.py           # Graph loading, KD-Tree, A* routing
│
├── bengaluru/
│   ├── graph_nodes.csv       # OSM road nodes (id, lat, lng, reachable flag)
│   └── graph_edges.csv       # OSM road edges (id1, id2, length_km)
│
├── templates/
│   ├── upload.html           # File upload page
│   └── final.html            # Results dashboard
│
├── static/
│   ├── css/
│   │   ├── input.css         # Upload page styles
│   │   └── output.css        # Dashboard styles
│   ├── js/
│   │   └── colors.js         # Gradient colour utilities for tables & charts
│   └── img/
│       ├── logo.png
│       └── icon.png
│
├── TestCases/
│   └── TestCase_TC01.xlsx … TestCase_TC09.xlsx
│
└── algo/                     # Exploratory notebooks & prototype files
    ├── mapgraph.py
    ├── mapgraph2.py
    └── backend.ipynb
```

---

## Setup & Installation

### Prerequisites

- Python 3.10+
- ~2 GB RAM (graph loading)

### Local Setup

```bash
# Clone the repository
git clone https://github.com/kaustavbhowal21/Velora.git
cd Velora

# Install dependencies
pip install -r requirements.txt

# Run the app
python app.py
```

The app pre-computes the road graph on startup (loads nodes and edges CSVs, builds the KD-Tree). This takes a few seconds.

### Docker

```bash
docker build -t velora .
docker run -p 7860:7860 velora
```

Then visit `http://localhost:7860`.

> **Note:** The `bengaluru/graph_nodes.csv` and `bengaluru/graph_edges.csv` files are stored in Git LFS. Run `git lfs pull` after cloning to download them.

---

## Running the App

| Route | Method | Description |
|-------|--------|-------------|
| `/` | GET | Redirects to `/input` |
| `/input` | GET | Upload page |
| `/output` | POST | Accepts `.xlsx` file, runs optimisation, returns HTML dashboard |
| `/result` | POST | Accepts `.xlsx` file, runs optimisation, returns raw JSON |

Navigate to `/input`, upload a TestCase Excel file, and click **Optimise Routes**. The dashboard renders automatically after the solver finishes.

---

## Input Format

The input is a single `.xlsx` file with **four sheets**:

### `employees`

| Column | Type | Description |
|--------|------|-------------|
| `employee_id` | string | Unique employee identifier |
| `priority` | int (1–5) | Scheduling priority (1 = highest) |
| `pickup_lat`, `pickup_lng` | float | Pickup coordinates |
| `drop_lat`, `drop_lng` | float | Office/drop-off coordinates |
| `earliest_pickup` | time | Earliest the vehicle may pick up this employee |
| `latest_drop` | time | Hard deadline to reach the office |
| `vehicle_preference` | `premium` / `normal` / `any` | Acceptable vehicle categories |
| `sharing_preference` | `single` / `double` / `triple` | Max co-passengers allowed |

### `vehicles`

| Column | Type | Description |
|--------|------|-------------|
| `vehicle_id` | string | Unique vehicle identifier |
| `fuel_type` | string | e.g. `CNG`, `Electric`, `Petrol` |
| `vehicle_type` | string | e.g. `Sedan`, `SUV` |
| `category` | `premium` / `normal` | Vehicle tier |
| `capacity` | int | Maximum passengers |
| `cost_per_km` | float | Operating cost in ₹/km |
| `avg_speed_kmph` | float | Average speed used for time estimation |
| `current_lat`, `current_lng` | float | Starting depot location |
| `available_from` | time | Earliest the vehicle can depart |

### `baseline`

| Column | Type | Description |
|--------|------|-------------|
| `employee_id` | string | References employee |
| `baseline_cost` | float | Cost if employee travels independently (₹) |
| `baseline_time_min` | float | Travel time if employee travels independently (min) |

### `metadata`

| Key | Description |
|-----|-------------|
| `alpha` / `objective_cost_weight` | Weight on cost in the objective (default `0.7`) |
| `beta` / `objective_time_weight` | Weight on time in the objective (default `0.3`) |
| `priority_1_max_delay_min` … `priority_5_max_delay_min` | Allowed delay (minutes) beyond `latest_drop` per priority level |

---

## Output & Dashboard

After optimisation, the dashboard is rendered with the following sections:

### Summary
- KPI cards: elapsed time, employees assigned, vehicles used, total distance, optimised cost & time, baseline comparisons, % optimised
- **Optimisation Overview**: Gauge charts for cost and time optimisation percentages
- **Constraint Health**: Cards showing percentage of employees whose vehicle type, sharing, and time constraints were satisfied
- **Fleet Overview**: Table of all vehicles coloured by usage
- **Distribution Charts**: Donut charts for cost and time breakdown by vehicle

### Employees
- Filterable table (All / Satisfied / Compromised / Unassigned)
- Pie chart showing assignment breakdown

### Violations *(shown only when violations exist)*
- Separate tables for vehicle type, sharing, and time violations
- Summary KPI cards per violation type

### Unassigned *(shown only when employees are unassigned)*
- Map showing unassigned employee pickup locations and their baseline solo routes to office
- Table with baseline cost and time for each unassigned employee

### Per-Vehicle Panes
- Leaflet map showing the vehicle's depot, all pickup points, and the polyline route
- Trip table with expandable rows showing per-employee pickup times and constraint status

---

## Configuration

The objective function is:

```
Objective = α × (travel_cost / baseline_cost) + β × (travel_time / baseline_time)
```

Set `alpha` and `beta` in the `metadata` sheet of your input file. They should sum to 1.0.

Priority-based time tolerance allows the solver to exceed `latest_drop` by a configurable buffer:

```
metadata sheet:
  priority_1_max_delay_min = 5
  priority_2_max_delay_min = 10
  priority_3_max_delay_min = 15
  priority_4_max_delay_min = 20
  priority_5_max_delay_min = 30
```

ALNS hyper-parameters can be tuned in `alns_engine.py → ALNSConfig`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `num_runs` | 25 | Number of independent ALNS restarts |
| `max_iter` | 2000 | Max iterations per run |
| `max_no_improve` | 400 | Early stopping patience |
| `q_min` / `q_max` | 1 / 4 | Removal size range |
| `temp_start` | 0.05 | Initial SA temperature (fraction of cost) |
| `cooling` | 0.9995 | SA cooling rate |

---

## Test Cases

Nine test cases are provided in `TestCases/`:

| File | Description |
|------|-------------|
| `TestCase_TC01.xlsx` | Small instance — basic feasibility check |
| `TestCase_TC02.xlsx` | Mixed vehicle types |
| `TestCase_TC03.xlsx` | High sharing preference variation |
| `TestCase_TC04.xlsx` | Premium-only employees |
| `TestCase_TC05.xlsx` | Tight time windows |
| `TestCase_TC06.xlsx` | Large employee count |
| `TestCase_TC07.xlsx` | Multiple depot locations |
| `TestCase_TC08.xlsx` | High priority spread |
| `TestCase_TC09.xlsx` | Combined stress test |

---

## Acknowledgements

We sincerely thank our project guide/mentor for their continuous support and valuable insights throughout development.

We also acknowledge the contributors to the open-source tools and libraries this project depends on:

- [Flask](https://flask.palletsprojects.com/) — Web framework
- [pandas](https://pandas.pydata.org/) — Data loading and Excel parsing
- [NumPy](https://numpy.org/) & [SciPy](https://scipy.org/) — Numerical operations and KD-Tree
- [OpenStreetMap](https://www.openstreetmap.org/) — Road network data for Bengaluru
- [Leaflet.js](https://leafletjs.com/) — Interactive maps
- [Chart.js](https://www.chartjs.org/) — Dashboard charts
- [Gunicorn](https://gunicorn.org/) — Production WSGI server

*This project was developed as part of General Championship.*
