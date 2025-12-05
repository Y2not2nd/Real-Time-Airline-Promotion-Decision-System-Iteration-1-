# Airline Streaming Analytics — Iteration 1

![Architecture diagram](diagram/iteration-1.drawio.png.png)

## Tech stack

This project is built using the following technologies:

* **Python 3**
  Used for producers, streaming consumers, and batch ETL logic.

* **Apache Kafka**
  Acts as the real-time event backbone for flight, booking, inventory, and promotion events.

* **PostgreSQL**
  Serves as the analytical warehouse for cleaned, analytics-ready tables.

* **Polars**
  Used for batch ETL and data transformation from raw events into fact and dimension tables.

* **Tableau**
  Used to visualise decisions, inventory trends, and promotion outcomes.

* **Docker & Docker Compose**
  Used to run Kafka, Postgres, and supporting services locally.

* **Redis (present but unused in Iteration 1)**
  Included as part of the infrastructure but not actively used yet. It is reserved for future iterations where fast-access state or caching may be required.

---

## Why I built this

I didn’t want to build another project that just reads a static CSV and draws a chart.

What interested me was how **data actually behaves in real systems**: events arriving over time, incomplete information, state changing, and decisions being made before you have the full picture.

This project is mainly about **streaming and real-time data**, with batch ETL and visualisation layered on top. The focus isn’t on clever pricing logic or optimisation yet — it’s on making sure the data flow itself makes sense and can be trusted before anything more sophisticated is added.

I also wanted to work with data that felt real. Real-time operational airline data isn’t something you can realistically download, so I generated it myself. The values are synthetic, but the structure, timing, and decision points reflect the kinds of signals real systems rely on.

Instead of writing a single script that produces an answer once, the goal was to build something that could evolve naturally. That mirrors how real data work usually happens: set a baseline, verify behaviour, then iterate.

---

## What this project is doing

At a high level, this project simulates an airline environment where flights, bookings, inventory changes, and promotions are modelled as **event streams**.

Bookings arrive over time, seat availability changes dynamically, and promotions are triggered when flights are not filling as expected.

The goal is not to optimise revenue. The goal is to design a system that can:

* observe what is happening in real time
* react to it using simple, explainable rules
* evaluate those decisions later using analytics

The data is synthetic, but the questions being modelled are real:

* how many seats are left
* how close a flight is to departure
* how full the aircraft is
* when an intervention should be triggered
* what that intervention looked like at the time

---

## Why streaming first

Streaming forces a different way of thinking than batch processing.

You are not working with a finished dataset. You react to events as they arrive, maintain state over time, and make decisions before you have the full picture.

Kafka is used so the system behaves more like something you would see in production rather than a purely offline analysis. Designing it this way forces questions about timing, state, and observability to be addressed early instead of being hidden behind batch assumptions.

---

## Event design and data realism

The producer scripts generate events that resemble real airline operational data.

Each event includes fields that would matter in a real setting, such as:

* flight identifiers
* seat capacity and current bookings
* fare buckets and prices
* booking timestamps
* minutes to departure
* load factor
* promotion metadata

Even though the values are simulated, the collection points are realistic. This makes downstream ETL, analytics, and dashboards meaningful rather than toy examples.

---

## What Iteration 1 focuses on

Iteration 1 is about getting the **entire pipeline working end to end**.

Kafka topics are created and populated, producers emit different airline events, streaming consumers maintain state and emit derived metrics and promotions, raw data is written to a data lake, batch ETL creates analytics-ready tables, and dashboards visualise what decisions were made.

If the system runs cleanly from ingestion through to visualisation — and the outputs reflect what actually happened — Iteration 1 is considered successful.

At this stage the priority is correctness, visibility, and understanding data flow. Efficiency, tuning, and automation are intentionally deferred.

---

## Promotion logic in Iteration 1

Promotion decisions are intentionally simple and fully rule based.

For each flight, the streaming consumer evaluates:

* minutes remaining until departure
* current load factor

If a flight is approaching departure and the load factor falls below a threshold, a promotion event is triggered. The discount level is fixed and based on broad timing bands rather than optimisation logic.

The purpose of this logic is to create realistic decision events and make those decisions visible downstream. Whether a promotion was effective is evaluated later, not decided in real time.

---

## How decisions are evaluated

A deliberate design choice is separating **decision-making** from **decision evaluation**.

Promotions are triggered in the streaming layer using only the information available at that moment. Their impact, or lack of it, is analysed later using stored events and dashboards.

This allows decisions to be reviewed, questioned, and improved without altering what the system does in real time. This layer is observational by design.

---

## Known limitations of Iteration 1

The following limitations are intentional and documented so future iterations can be evaluated against a clear baseline:

* promotions do not affect booking behaviour
* booking probability is static
* there is no demand elasticity or feedback loop
* ETL is not scheduled yet
* no predictive logic is present
* no machine learning is used

These are deferred deliberately to keep the initial system understandable.

---

## Iteration 2 — Behaviour and realism

Iteration 2 will focus on making the simulation more realistic while keeping it explainable.

Planned improvements include:

* allowing promotions to influence booking probability (assumption-driven elasticity)
* running longer simulations with more flights
* introducing basic promotion cooldown rules
* tracking simple demand momentum (booking velocity)
* scheduling ETL execution instead of running it manually

The goal of Iteration 2 is to close the loop between decisions, user behaviour, and evaluation.

---

## Iteration 3 — Predictive logic (non-ML)

Iteration 3 introduces a predictive layer without machine learning.

This is expected to involve:

* historical aggregates
* simple heuristics based on past performance
* basic forecasting of when intervention may be needed

The intent is to add foresight while keeping logic transparent and easy to reason about.

---

## Iteration 4 — Machine learning (only if justified)

Machine learning is intentionally deferred.

Iteration 4 will only be pursued if the data quality is high enough, simpler approaches reach clear limits, and predictive accuracy provides meaningful value beyond rule-based logic.

If introduced, ML would sit on top of the existing pipeline and be evaluated alongside simpler approaches rather than replacing them.

---

## Final note

Iteration 1 is already a complete end-to-end system that processes data as it arrives and makes its decisions observable.

Each iteration builds on this foundation rather than rewriting it.



