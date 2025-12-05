# Real-Time Airline Promotion Decision System — Iteration 2 Design

## A. Repository Duplication Plan
- **New repository name:** `Real-Time-Airline-Promotion-Decision-System-Iteration-2` (created separately; Iteration 1 remains untouched).
- **Files copied unchanged from Iteration 1 (baseline):**
  - All Docker, Kafka, PostgreSQL, and Python scaffolding from Iteration 1: `docker-compose.yaml`, `requirements.txt`, base ETL scripts (`etl/`), data lake layout (`lake/`), producer scaffolding (`producer/`), and streaming scaffolding (`streaming/`) are duplicated as-is to provide runtime parity.
- **Files modified/added in Iteration 2:**
  - `README.md` and Project Setup & Execution Guide: updated to describe Iteration 2 behaviour and how to run it independently.
  - `streaming/consumer.py` (MODIFIED): add behavioural realism, feedback-aware state, and explainable rule-based scoring.
  - `streaming/state.py` (NEW): explicit state holder for promotion exposure, bookings, and campaign-level counters (Polars-backed persistence snapshotting to disk/DB as needed).
  - `streaming/rules.py` (NEW): self-contained, explainable rule engine for promotion influence on booking probability and suppression logic.
  - `streaming/models.py` (NEW/MODIFIED if already present): simple data classes for events with new fields (promotion_influence, baseline_prob, adjusted_prob, exposure_counts).
  - `producer/synthetic_events.py` (MODIFIED): include promotion-feedback attributes (e.g., recent exposure flag, prior bookings) for more realistic simulations.
  - `doc/ITERATION2_DESIGN.md` (NEW): this design document.

## B. Detailed Design Changes
- **Behavioural realism introduced:**
  - Promotions now have a **cool-down** and **fatigue** model: repeated exposures within a short window reduce effectiveness and can trigger suppression to avoid spamming.
  - Booking probability is a combination of a **baseline route/price sensitivity** plus **promotion influence** that decays with exposure and time since last promotion.
  - A **feedback loop** updates customer and route-level state on each booking and promotion event to influence subsequent decisions.
- **Promotion influence on booking probability (rule-based):**
  - Start with `baseline_prob` derived from fare class + route popularity (configurable table).
  - Apply **promotion uplift**: +`promo_uplift` when the customer has not seen the promo in the last `cooldown_minutes`.
  - Apply **fatigue penalty**: -`fatigue_penalty` if exposure count in the last `fatigue_window_minutes` exceeds threshold.
  - Apply **recency decay**: uplift decays by `decay_factor ** minutes_since_last_promo`.
  - Resulting `adjusted_prob = clamp(baseline_prob + uplift - fatigue, min_prob, max_prob)`.
  - All parameters live in `rules.py` and are fully explainable; no ML used.
- **State management in streaming consumer:**
  - `StateStore` (in `state.py`) maintains:
    - `customer_state`: last promo timestamp, exposure count window, last booking timestamp.
    - `route_state`: aggregate bookings, conversions, last promo effectiveness.
    - `campaign_state`: per-promo exposures, conversions for observability.
  - State is **read/write through**: in-memory dict/Polars frame with periodic snapshot to disk/DB (e.g., PostgreSQL) via upserts to ensure crash recovery.
  - Consumer processes Kafka events sequentially; state updates are wrapped in idempotent functions keyed by event IDs to avoid double-apply.
  - Metrics emitted after each update (exposure_count, conversion_rate) for observability.

## C. File-by-File Changes (New vs Modified)

### NEW: `streaming/rules.py`
```python
"""Rule-based promotion influence for Iteration 2 (no ML)."""
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Dict

@dataclass
class RuleParams:
    baseline_prob: float
    promo_uplift: float = 0.15
    fatigue_penalty: float = 0.10
    decay_factor: float = 0.98
    cooldown_minutes: int = 30
    fatigue_window_minutes: int = 120
    min_prob: float = 0.01
    max_prob: float = 0.95

# Example baselines per route or fare class
ROUTE_BASELINES: Dict[str, float] = {
    "JFK-LAX": 0.08,
    "LAX-SFO": 0.05,
    "SEA-DEN": 0.04,
    "DEFAULT": 0.03,
}


def get_baseline(route: str) -> float:
    return ROUTE_BASELINES.get(route, ROUTE_BASELINES["DEFAULT"])


def adjusted_probability(route: str, now: datetime, last_promo: datetime | None, exposures_in_window: int, params: RuleParams | None = None) -> tuple[float, dict]:
    params = params or RuleParams(baseline_prob=get_baseline(route))
    baseline = params.baseline_prob
    uplift = params.promo_uplift
    fatigue = params.fatigue_penalty if exposures_in_window > 2 else 0.0

    if last_promo:
        minutes_since = (now - last_promo) / timedelta(minutes=1)
        if minutes_since < params.cooldown_minutes:
            uplift *= 0.5  # dampen during cooldown
        uplift *= params.decay_factor ** max(minutes_since, 0)
    else:
        minutes_since = None

    adjusted = min(max(baseline + uplift - fatigue, params.min_prob), params.max_prob)
    explanation = {
        "baseline": baseline,
        "uplift": uplift,
        "fatigue": fatigue,
        "minutes_since_last_promo": minutes_since,
        "adjusted_prob": adjusted,
    }
    return adjusted, explanation
```

### NEW: `streaming/state.py`
```python
"""State storage for Iteration 2 streaming consumer."""
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Dict, Optional
import polars as pl

@dataclass
class CustomerState:
    last_promo_at: Optional[datetime] = None
    exposures: list[datetime] = field(default_factory=list)
    last_booking_at: Optional[datetime] = None

@dataclass
class CampaignState:
    promo_id: str
    exposures: int = 0
    bookings: int = 0

class StateStore:
    def __init__(self):
        self.customers: Dict[str, CustomerState] = {}
        self.campaigns: Dict[str, CampaignState] = {}

    def record_promo(self, customer_id: str, promo_id: str, ts: datetime):
        cust = self.customers.setdefault(customer_id, CustomerState())
        cust.last_promo_at = ts
        cust.exposures.append(ts)
        camp = self.campaigns.setdefault(promo_id, CampaignState(promo_id))
        camp.exposures += 1

    def record_booking(self, customer_id: str, promo_id: str | None, ts: datetime):
        cust = self.customers.setdefault(customer_id, CustomerState())
        cust.last_booking_at = ts
        if promo_id:
            camp = self.campaigns.setdefault(promo_id, CampaignState(promo_id))
            camp.bookings += 1

    def exposure_count(self, customer_id: str, window: timedelta, now: datetime) -> int:
        cust = self.customers.get(customer_id)
        if not cust:
            return 0
        cutoff = now - window
        cust.exposures = [t for t in cust.exposures if t >= cutoff]
        return len(cust.exposures)

    def to_polars_frames(self) -> dict[str, pl.DataFrame]:
        return {
            "customers": pl.DataFrame([
                {
                    "customer_id": cid,
                    "last_promo_at": c.last_promo_at,
                    "last_booking_at": c.last_booking_at,
                    "exposure_count": len(c.exposures),
                }
                for cid, c in self.customers.items()
            ]),
            "campaigns": pl.DataFrame([
                {"promo_id": cid, "exposures": c.exposures, "bookings": c.bookings}
                for cid, c in self.campaigns.items()
            ]),
        }
```

### MODIFIED: `streaming/consumer.py` (core loop sketch)
```python
from datetime import datetime, timedelta
import json
from kafka import KafkaConsumer, KafkaProducer
from .state import StateStore
from .rules import adjusted_probability, RuleParams

PROMO_TOPIC = "promotions"
BOOKING_TOPIC = "bookings"
FEEDBACK_TOPIC = "promotion-feedback"

state = StateStore()

consumer = KafkaConsumer(PROMO_TOPIC, value_deserializer=lambda m: json.loads(m.decode("utf-8")))
producer = KafkaProducer(value_serializer=lambda m: json.dumps(m).encode("utf-8"))

for message in consumer:
    event = message.value
    now = datetime.utcnow()
    customer_id = event["customer_id"]
    promo_id = event["promo_id"]
    route = event.get("route", "DEFAULT")

    exposures_in_window = state.exposure_count(customer_id, timedelta(minutes=120), now)
    prob, explanation = adjusted_probability(
        route=route,
        now=now,
        last_promo=state.customers.get(customer_id, None) and state.customers[customer_id].last_promo_at,
        exposures_in_window=exposures_in_window,
        params=RuleParams(baseline_prob=None),  # baseline inferred in adjusted_probability
    )

    state.record_promo(customer_id, promo_id, now)

    booking_occurs = random.random() < prob
    feedback_event = {
        "customer_id": customer_id,
        "promo_id": promo_id,
        "route": route,
        "baseline_prob": explanation["baseline"],
        "adjusted_prob": explanation["adjusted_prob"],
        "fatigue": explanation["fatigue"],
        "uplift": explanation["uplift"],
        "minutes_since_last_promo": explanation["minutes_since_last_promo"],
        "booking": booking_occurs,
        "event_ts": now.isoformat(),
    }

    producer.send(FEEDBACK_TOPIC, feedback_event)
    if booking_occurs:
        state.record_booking(customer_id, promo_id, now)
        producer.send(BOOKING_TOPIC, {
            "customer_id": customer_id,
            "route": route,
            "promo_id": promo_id,
            "booking_ts": now.isoformat(),
            "explanation": explanation,
        })
```

### MODIFIED: `producer/synthetic_events.py`
- Add booking-behaviour inputs: price buckets, route popularity, customer segment.
- Emit promotion events with realistic cadence (respect fatigue window) and include new fields: `route`, `fare_class`, `promo_id`, `segment`, `event_ts`.

### MODIFIED/NEW: `streaming/models.py`
- Define data classes `PromotionEvent`, `BookingEvent`, and `FeedbackEvent` including `adjusted_prob`, `fatigue`, `uplift`, and timestamps for transparency.

### MODIFIED: `README.md` & Project Setup Guide
- Add “Iteration 2 — Behavioural Realism” section describing new topics, event fields, and how to run the new repo independently via Docker Compose.

## D. Data Flow Impact
- **Kafka Topics:**
  - `promotions` (unchanged producer topic).
  - `promotion-feedback` (NEW): emitted by consumer to log each decision with explanation fields.
  - `bookings` (modified payload): now contains `promo_id`, `adjusted_prob`, and `explanation` block.
- **Event fields added:**
  - `baseline_prob`, `adjusted_prob`, `fatigue`, `uplift`, `minutes_since_last_promo`, `booking` flag on feedback events.
  - Booking events add `promo_id` (nullable) and `explanation` dictionary.
- **Flow:** promotions → consumer applies rules/state → feedback + optional booking events → sink to PostgreSQL/Parquet for analytics.

## E. Documentation Updates
- **README.md:**
  - New section “Iteration 2 — Behavioural Realism” explaining rule parameters, feedback topic, and how to interpret logs.
  - Update architecture diagram to include feedback loop and state store.
  - Update run steps: `docker-compose up`, then run `producer` and `streaming` services for Iteration 2.
- **Project Setup & Execution Guide (doc/):**
  - Explicit steps to clone new repo `Real-Time-Airline-Promotion-Decision-System-Iteration-2`.
  - List new environment variables if any (e.g., state snapshot path, feedback topic names).
  - Add troubleshooting for state snapshot/Polars dependencies.

## F. Safety & Verification
- **Validation steps:**
  - Run docker-compose; ensure topics `promotions`, `promotion-feedback`, `bookings` are created.
  - Produce sample promotions; verify consumer logs show baseline/uplift/fatigue calculations and adjusted probabilities.
  - Confirm `promotion-feedback` events persist in PostgreSQL or Parquet with non-null `adjusted_prob`.
  - Observe that repeated promotions within the fatigue window reduce `adjusted_prob` and increase suppression.
- **Observable signals:**
  - Logs: each decision emits structured JSON with explanation fields.
  - Metrics: exposure_count, conversion_rate per promo (export via stdout or Prometheus textfile).
  - Data: state snapshots show rising exposure counts and booking conversions per campaign.
- **Independence from Iteration 1:**
  - New repo name, new state paths (e.g., `/data/iter2/state.parquet`), and isolated Docker Compose project name ensure no collision with Iteration 1 containers or volumes.

## Assumptions
- Iteration 2 reuses the same core stack (Kafka, PostgreSQL, Python, Polars) with no cloud dependencies.
- Redis is not required; in-memory + periodic persistence is sufficient for this scope.
- The provided code snippets illustrate structure; exact wiring (imports, module paths) should follow the duplicated Iteration 1 repo layout.
