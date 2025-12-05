# Project Setup & Execution Guide — Iteration 1

This is the walkthrough for how I actually run the project end to end, from Docker all the way through to Tableau.

If you follow this in order, you’ll end up with Kafka streaming events, a small data lake being populated, an ETL job building warehouse tables, and a Tableau dashboard sitting on top of it.

---

## Prerequisites

### Required software

- **Docker Desktop**  
  Needs to be able to run local containers.

- **Python 3.10+**

- **PowerShell** (or another shell you’re comfortable with)  
  All examples here use PowerShell on Windows.

- **Tableau Desktop** (trial is fine)

---

### Python environment

You should be okay with:

- creating a virtual environment  
- installing packages with `pip`  
- running Python scripts from the command line  

---

## Tech stack (Iteration 1)

- **Kafka**  
  Event streaming backbone.

- **Python**  
  Producers, streaming consumer, ETL.

- **Postgres**  
  Analytics warehouse.

- **Local JSONL “lake”**  
  Raw Kafka events written to disk.

- **Polars**  
  Batch ETL on the lake.

- **Tableau**  
  Visualising and reviewing promotion decisions.

- **Redis**  
  Container is there in Docker Compose, but not used yet in Iteration 1 (kept for future low-latency use).

---

## Project structure

On disk, the project looks like this:

```text
airline-streaming-project/
│
├─ docker-compose.yaml
│
├─ producer/
│   └─ airline_producer.py
│
├─ streaming/
│   └─ consumer.py
│
├─ lake/
│   ├─ lake_writer.py
│   ├─ raw/    # created automatically when events arrive
│   └─ gold/   # created by ETL
│
├─ etl/
│   └─ build_warehouse.py
│
├─ requirements.txt
└─ README.md (plus this doc)
````

`lake/raw` and `lake/gold` don’t need to exist before you run anything. They’ll be created as the system runs.

---

## Python dependencies

Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install dependencies from `requirements.txt`:

```powershell
pip install -r requirements.txt
```

If installing manually, the key ones are:

```powershell
pip install kafka-python polars psycopg2
```

Optional sanity check:

```powershell
pip show kafka-python
pip show polars
```

---

## Step 1 — Start infrastructure (Docker)

From the project root:

```powershell
docker compose up -d
```

Check that the containers are up:

```powershell
docker ps
```

You should see at least:

* `zookeeper`
* `kafka`
* `postgres`
* `redis`

Redis is just along for the ride in Iteration 1.

---

## Step 2 — Verify Postgres is alive

```powershell
docker exec -it postgres psql -U airline -d airline_dw
```

Inside `psql`:

```sql
SELECT version();
\q
```

If this works, Postgres is ready for the warehouse tables later.

---

## Step 3 — Create Kafka topics

Enter the Kafka container:

```powershell
docker exec -it kafka bash
```

Create the topics:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic flight_lifecycle --partitions 3 --replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create --topic bookings --partitions 3 --replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create --topic seat_inventory --partitions 3 --replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create --topic seat_map --partitions 3 --replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create --topic inventory_metrics --partitions 3 --replication-factor 1

kafka-topics --bootstrap-server localhost:9092 \
  --create --topic promo_decisions --partitions 3 --replication-factor 1
```

Confirm:

```bash
kafka-topics --bootstrap-server localhost:9092 --list
```

Exit:

```bash
exit
```

---

## Step 4 — Start the lake ingestion service

This listens to Kafka and writes every event to disk under `lake/raw` as newline-delimited JSON (`.jsonl`).

```powershell
.\.venv\Scripts\Activate.ps1
python .\lake\lake_writer.py
```

Expected output:

```text
🪣 Lake ingestion started
```

Leave this running.
Folders under `lake/raw` will appear automatically.

---

## Step 5 — Start the streaming inventory & promotion engine

In a new terminal:

```powershell
.\.venv\Scripts\Activate.ps1
python .\streaming\inventory_engine.py
```

What this process does:

* keeps track of flights and departures in memory
* updates `inventory_metrics` when bookings arrive
* decides when promotions fire and emits `promo_decisions`

Example output:

```text
✈️ Tracking flight BA210_2025-11-28
📊 Metrics updated for BA210_2025-11-28
💸 Promo triggered for BA210_2025-11-28: 10%
```

---

## Step 6 — Start the producer

In a third terminal:

```powershell
.\.venv\Scripts\Activate.ps1
python .\producer\airline_producer.py
```

This generates flights, bookings, and seat inventory snapshots and pushes them to Kafka.

---

## Step 7 — Optional: Peek at Kafka topics

Example for `bookings`:

```powershell
docker exec -it kafka bash -c \
'kafka-console-consumer --bootstrap-server localhost:9092 --topic bookings --from-beginning'
```

You can also inspect:

* `inventory_metrics`
* `promo_decisions`

---

## Step 8 — Check the raw data lake

```powershell
ls .\lake\raw
ls .\lake\raw\bookings
```

Preview data:

```powershell
Get-Content .\lake\raw\bookings\*.jsonl | Select-Object -First 3
```

---

## Step 9 — Run the batch ETL (Polars)

```powershell
.\.venv\Scripts\Activate.ps1
python .\etl\build_warehouse.py
```

Expected output:

```text
 Writing fact_booking
 Writing fact_inventory_metrics
 Writing fact_promotions
```

Only fact tables are loaded in Iteration 1.

---

## Step 10 — Check the gold layer

```powershell
ls .\lake\gold
```

Expected files:

* `fact_booking.parquet`
* `fact_inventory_metrics.parquet`
* `fact_promotions.parquet`

---

## Step 11 — Verify Postgres tables

```powershell
docker exec -it postgres psql -U airline -d airline_dw
```

```sql
\dt

SELECT COUNT(*) FROM fact_booking;
SELECT COUNT(*) FROM fact_inventory_metrics;
SELECT COUNT(*) FROM fact_promotions;

\q
```

---

## Step 12 — Tableau setup

### 12.1 Install PostgreSQL driver

Follow Tableau’s prompt to install the PostgreSQL driver on first connection.

---

### 12.2 Connect Tableau to Postgres

* Server: `localhost`
* Port: `5432`
* Database: `airline_dw`
* Username: `airline`
* Password: `airline`
* SSL: Off

---

### 12.3 Data model in Tableau

Create relationships on `flight_id`:

* `fact_booking.flight_id` ↔ `fact_inventory_metrics.flight_id`
* `fact_booking.flight_id` ↔ `fact_promotions.flight_id`

---

### 12.4 Dashboards

**Inventory trend with promotion annotations**

* X-axis: Metric Time
* Dual Y-axis: Seat remaining and Discount percentage
* Colour by Flight Id

**Booking pressure as departure approaches**

* X-axis: Minutes To Departure (reversed)
* Y-axis: Load Factor
* Colour by Flight Id

Dashboard exists to review decisions, not drive live actions.

---

## What Iteration 1 proves

* streaming ingestion with Kafka
* stateful streaming decisions
* file-based data lake
* warehouse tables via Polars
* analytics review in Tableau

---

## Known limitations in Iteration 1

* promotions do not change booking behaviour
* booking probability is static
* ETL is manual
* Redis is unused
* no prediction or ML
* no `dim_flight` yet
