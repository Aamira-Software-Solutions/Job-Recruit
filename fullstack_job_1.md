# Aamira Courier – Package Tracker Coding Exercise

**Estimated Effort:** \~6–8 hours of hands-on work (spread across a few days is fine). You do *not* need to build a production-ready system; we’re evaluating how you analyze requirements, make technical choices, and deliver a coherent working slice.

---

## 1. Business Context (What Problem Are We Solving?)

Acme Courier provides **same-day delivery** services in several metro areas. Dispatchers need to know where packages are, whether they’re moving on schedule, and when to intervene if something is stuck. Couriers use a mobile device to report status updates as they pick up and drop off packages throughout the day.

The company currently uses spreadsheets and text messages—error‑prone, delayed, and impossible to audit. We want a minimal but functional **internal Package Tracker** to replace this ad‑hoc process.

---

## 2. User Roles

**Courier** – Sends status/location updates from the field (mobile app, handheld scanner, or API client).

**Dispatcher** – Office staff viewing a live dashboard of all active packages; filters, searches, and drills into package history; receives alerts when something appears stuck.

**System / Alert Service** – Watches package state changes; raises alerts if a package hasn’t progressed in >30 minutes.

---

## 3. High-Level Goals

1. **Ingest updates** coming from couriers (you may implement this as REST; REST is easiest for demo).
2. **Store package state reliably** so we can restart the service without losing history.
3. **Expose a real-time dispatcher dashboard** showing current status, location, and ETA for every active package.
4. **Generate alerts** when a package hasn’t advanced for more than 30 minutes.

---

## 4. Data Model (Suggested – You May Extend)

At minimum, each **Package** should have:

| Field             | Type     | Required? | Notes                                                                 |
| ----------------- | -------- | --------- | --------------------------------------------------------------------- |
| `package_id`      | string   | Yes       | Unique across the system (e.g., barcode).                             |
| `status`          | enum     | Yes       | See status lifecycle below.                                           |
| `lat`             | float    | Optional  | Latest known courier latitude.                                        |
| `lon`             | float    | Optional  | Latest known courier longitude.                                       |
| `event_timestamp` | datetime | Yes       | When the courier recorded the event (may differ from server receipt). |
| `received_at`     | datetime | Yes       | Server receipt time; useful for debugging clock skew.                 |
| `note`            | string   | Optional  | Freeform comment from courier ("Left at front desk").                 |
| `eta`             | datetime | Optional  | Estimated delivery; may be computed or provided.                      |

### 4.1 Status Lifecycle (Recommended Enum Values)

You may change names, but please document if you do.

```
CREATED          – Label generated; not yet picked up.
PICKED_UP        – Courier has the package.
IN_TRANSIT       – Moving between facilities/route stops.
OUT_FOR_DELIVERY – On vehicle and en route to final drop.
DELIVERED        – Delivered to the recipient.
EXCEPTION        – Delay/address issue/weather hold.
CANCELLED        – Shipment cancelled.
```

Only **DELIVERED** and **CANCELLED** are terminal states; others are considered *active*.

---

## 5. Functional Requirements (Detailed)

Below are the must-haves. Meet these first; anything extra is a bonus.

### F1. Ingest Courier Updates

- Provide an endpoint or message consumer that accepts package status events.
- Minimum JSON payload:
  ```json
  {
    "package_id": "PKG12345",
    "status": "IN_TRANSIT",
    "lat": 39.7684,
    "lon": -86.1581,
    "timestamp": "2025-07-16T18:22:05Z",
    "note": "Departed Indy hub"
  }
  ```
- Must be **idempotent**: re-sending the same event should not corrupt data (hint: event hash, event\_timestamp + status de-dupe, or optimistic upsert).
- Handle **out-of-order** events (e.g., a late networked device sends an older status). Decide how you resolve: ignore stale? append to history but don’t roll back current? Document your choice.

### F2. Persist State & History

- Store *every* event in a durable store (SQL, NoSQL —your call) so you can reconstruct history.
- Track the **current state** per package (last valid event by timestamp).
- On service restart, dashboard should rebuild current view from persistence.

### F3. Dispatcher Dashboard (Real-Time View)

- Web UI that lists all **active** packages (not DELIVERED or CANCELLED) seen in the last 24 hours.
- Each row/card shows: `package_id`, current `status`, time since last update, and (if available) map location.
- Include a way to drill into a **Package Detail** view: chronological event timeline.
- **Real-time requirement:** When a new update arrives, dispatchers should see the change *without reloading the page.* Acceptable strategies: WebSockets, Server-Sent Events, long-polling, or periodic polling ≤5s. Document which you chose and why.
- Display `eta` if available. If you don’t compute ETA, show "—".

### F4. Stuck-Package Alerting (>30 Minutes)

- A package is **stuck** if *no status change* has occurred for >30 minutes *and* it is still active.
- When detected, trigger an alert (choose one):
  - Send e-mail (mock SMTP ok).
  - Log to a dedicated "alerts" table and display in dashboard.
  - Browser notification in dispatcher UI.
- Avoid alert spam: after first alert, either (a) re-alert every 30 minutes, or (b) alert once until state changes—document which you implemented.

### F5. Basic Security / Access Control (Lightweight)

- System is internal, but don’t leave APIs totally open.
- Acceptable: simple API token in header, HTTP basic auth, or signed HMAC per request.
- Document how to configure credentials.

### F6. Minimal Seed / Demo Data

- Include a script or README instructions to seed 5–10 sample packages and simulate updates so reviewers can run and see the UI quickly.

---

## 6. Non-Functional Expectations

| Area             | Minimum                                                      | Nice-to-Have                                               |
| ---------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| **Build/Run**    | Clear README; one or two commands to run locally.            | Docker Compose; Makefile; IaC snippet (Terraform, Pulumi). |
| **Reliability**  | Doesn’t crash on malformed input; logs errors.               | Retry strategy for transient DB/queue errors.              |
| **Code Quality** | Idiomatic, modular, readable, linted.                        | Layered architecture, typed models, domain services.       |

---

## 7. Assumptions & Clarifying Questions (We Assess This!)

Real projects always have gaps. Part of the evaluation is how you *surface* uncertainty. Please include an **ASSUMPTIONS.md** (or README section) that lists:

- Any requirement you interpreted.
- Any shortcuts or trade-offs (e.g., no auth in demo mode; in-memory cache with periodic flush; fixed map center).
- Known limitations and "what I’d do next."

We score you positively for thoughtful, explicit assumptions.

---

## 8. Evaluation Rubric (How We Score)

We review code, docs, and demo. Each dimension scored 1–5.

| Dimension                      | 1 (Poor)                             | 3 (Meets)                                                       | 5 (Excellent)                                                      |
| ------------------------------ | ------------------------------------ | --------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Requirements Understanding** | Missed key asks; no assumptions doc. | All required features present or clearly triaged.               | Anticipated edge cases; documented alternatives.                   |
| **Architecture & Design**      | Ad-hoc scripts.                      | Reasonable modular design; persistent store; live updates work. | Clear layering, extensibility, error handling, idempotence, tests. |
| **Code Quality**               | Hard to read; no structure.          | Clean enough to follow; basic tests.                            | Production-quality patterns; good naming; high test coverage.      |
| **Front-End UX (if built)**    | Barely usable.                       | Functional list & details; updates visible.                     | Polished, filters, map, alert indicators.                          |
| **Communication**              | Sparse README.                       | Setup + usage documented.                                       | Clear trade-offs, diagrams, short demo video.                      |

Bonus points: containerization, CI pipeline, deployed demo URL, metrics/logging, graceful error UI.

---



---

## 12. Minimal UI Wireframe (ASCII)

```
+------------------------------------------------------------+
|  Acme Package Tracker                                      |
+------------------------------------------------------------+
| Search: [___________]   Filter: [Active ▼]                 |
+----+------------+--------------+-----------+---------------+
| #  | Package ID | Status       | Last Seen | Location      |
+----+------------+--------------+-----------+---------------+
| 1  | PKG12345   | IN_TRANSIT   | 12m ago   | 39.76,-86.16  |
| 2  | PKG55510   | OUT_FOR_DEL. | 02m ago   | 39.78,-86.10  |
| 3  | PKG90001   | STUCK!*      | 47m ago   | 39.70,-86.21  |
+----+------------+--------------+-----------+---------------+
*Row highlights red; alert banner.
```

Clicking a row opens the package history + map pin.

---

## 13. What *Not* Required (Don’t Over-Engineer)

You do **not** need: full auth/roles system, production CI/CD, mobile apps, geo-routing optimization, multi-tenant billing, or bulletproof scaling. Focus on the core workflow.

---

## 14. How We Review Your Work

We will:

1. Check your github repo and README.
2. In an Online meeting, you will give a demo.
   1. Post sample events; confirm dashboard updates & stuck alerts.
3. Scan code structure, tests, and error handling.
4. Review your assumptions & design notes.


We care more about **clarity, reasoning, and correctness** than feature breadth.

---

## 15. Questions?

In a real engagement, you’d ask questions early. For the exercise, you may:

- Include questions inline in your README and answer them with your assumptions.
- (If this is a live interview) Feel free to email your contact with clarifications before starting.

---


**Thank you & good luck!** We’re excited to see how you think and build.

