# ObserveX 🛡️⚡

**ObserveX** is an enterprise-grade, privacy-first, two-tier observability and AIOps platform. It pairs a **Tier 1 Local Collector** (deployed in the customer environment) that enforces zero raw data egress with a **Tier 2 Central SaaS & AIOps** platform that ingests allow-listed metadata, detects subtle anomalies using statistical baselines (EWMA), and generates explainable, human-in-the-loop remediation runbooks narrated by Google Gemini.

---

## 🌟 Key Features

- **Strict Data Minimization & Privacy by Design (Tier 1):**
  - Raw logs, usernames, IPs, and passwords never leave the customer network.
  - Multi-stage secret redaction using high-speed Regex patterns, Microsoft Presidio, and GLiNER token classification.
  - Log clustering via Drain3 into template IDs.
  - High-performance local HMAC-SHA256 pseudonymization for entity tracking (`h_` hosts, `u_` users).
  - Network-isolated collector architecture (`internal: true`) with a single audited, cryptographically-proven egress gateway (`collector.v1` envelope with SHA-256 ACK).
- **Central Multi-Tenant SaaS & Data Plane (Tier 2):**
  - High-throughput ingestion edge built on Spring Boot 3.5 with strict schema validation (`fail-on-unknown-properties`).
  - Apache Kafka event streaming with KRaft mode and partitioned telemetry queues.
  - Multi-tenant PostgreSQL ledger ensuring strict tenant data isolation.
- **Intelligent AIOps & LangGraph Remediation Agent:**
  - Dynamic EWMA baselining ($z \ge 3$) alongside static threshold alerts.
  - Root cause analysis and cross-service incident correlation (network degradation vs. application latency).
  - Human-in-the-loop approvals with dry-run safe action simulations and automatic rollbacks.
  - Decoupled Gemini LLM narrative explanations (fast deterministic templates first, followed by background asynchronous Gemini enhancement).
- **Interactive Simulation & Microservices Testbed:**
  - Microservices fleet: `product-service`, `order-service`, `payment-service` (Spring Boot 3.3).
  - Full stream simulator generating chaos scenarios (latency spikes, CPU drift, brute-force auth attacks, secret leakage, duplicate/out-of-order delivery).
  - Local diagnostic UI and SaaS Operations Dashboard.

---

## 🏗️ Architecture Overview

```
                      CUSTOMER ENVIRONMENT (TIER 1)
┌────────────────────────────────────────────────────────────────────────┐
│  Spring Boot Microservices        Telemetry Ingestion Stream           │
│  [Product :8081] [Order :8082]    [Stream Simulator :7100]             │
│  [Payment :8083]                  [Stream Adapter :7000]               │
│          │                                   │                         │
│          ▼ (Logs)                            ▼ (Metrics / Logs)        │
│   Fluent Bit (:24224 / :9880) ───────────────┤                         │
│   Prometheus (:9090) ────────────────────────┘                         │
│          │                                                             │
│ ═════════▼═════════════════ tier1_core (internal network) ═══════════ │
│   Python Processor (Collector)                                         │
│   ├─ SQLite Store (Raw logs, 2h TTL)                                   │
│   ├─ Redaction Engine (Regex + Presidio + GLiNER)                      │
│   ├─ Drain3 Log Template Miner                                         │
│   └─ Failed Login & Anomaly Detector                                   │
│          │ (Allow-listed metadata only)                                │
│   Egress Gateway (:8100)                                               │
│   ├─ Allow-list Schema Validation                                      │
│   ├─ Final Secret Quarantine Scan                                      │
│   ├─ SHA-256 Egress Proof Audit Log                                    │
│   └─ SQLite Reliable Outbox & Retry                                    │
└──────────┬─────────────────────────────────────────────────────────────┘
           │ HTTPS + Bearer Token (egress_net)
           ▼
                      CENTRAL SAAS PLATFORM (TIER 2)
┌────────────────────────────────────────────────────────────────────────┐
│   SaaS Ingestion Edge (:8443) ──▶ Apache Kafka (telemetry.ap-south-1)  │
│   (Spring Boot Ingest Profile)         │ group: db-writer  │ aiops     │
│                                        ▼                   ▼           │
│                                  PostgreSQL Ledger   AIOps Worker      │
│                                        ▲             ├─ EWMA Baseline  │
│                                        │             ├─ LangGraph      │
│   SaaS Dashboard & API (:8080) ────────┘             ├─ Gemini LLM     │
│   (Operators: op-tenant-a)                           └─ Action Policy  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Data Isolation: What Leaves Tier 1 vs. What Never Leaves

| Data Category | Handled In Tier 1 (Customer Core) | Allowed into Tier 2 (Central SaaS) |
|---|:---:|:---:|
| **Raw Logs & Stack Traces** | Kept in local 2-hour SQLite | ❌ **Never Leaves** |
| **IP Addresses & Hostnames** | Processed locally | ❌ Only pseudonyms (`h_...`) |
| **Usernames & Credentials** | Processed locally | ❌ Only pseudonyms (`u_...`) |
| **API Keys, Secrets & Tokens** | Redacted & Quarantined | ❌ Only hit counts (e.g., `{"AWS_KEY": 1}`) |
| **Metric Aggregates** | Computed in 2-second windows | ✅ P95, Avg, Max latency/CPU |
| **Security Alerts** | Auth attack detected locally | ✅ Source pseudonym, count, severity |
| **Health State & Reasons** | Evaluated locally | ✅ State (OK/Fair/Poor) & metric breaches |
| **Delivery Proof** | SHA-256 digest computed | ✅ Matching SHA-256 ACK verification |

---

## 🚀 Quick Start

### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/)
- At least 6 GB RAM allocated to Docker Desktop (GLiNER / Spring Boot builds)

### 1. Clone & Configure
```bash
git clone https://github.com/Jatin544-art/ObserveX.git
cd ObserveX

# Create local environment configuration
cp .env.example .env
```
*(Optional: Provide a `GEMINI_API_KEY` in `.env` for AI narrative generation; otherwise, the built-in deterministic templates are used automatically.)*

### 2. Start ObserveX
Start both Tier 1 and Tier 2 with the stream simulator:
```bash
docker compose --profile sim up -d --build
```

### 3. Verify System Health
Check that all containers and initialization jobs completed successfully:
```bash
docker compose ps
```
- `tier1-init`, `tier2-init`, and `kafka-init` will report `exited (0)`.
- All other services (`product-service`, `order-service`, `payment-service`, `python-processor`, `egress-gateway`, `saas-ingest`, `saas-api`, `aiops`, `postgres`, `kafka`, `fluent-bit`, `prometheus`) should be healthy and running.

---

## 🌐 Web Dashboards & Endpoints

| Service | Address | Description |
|---|---|---|
| **Tier 1 Local UI** | [http://127.0.0.1:8090](http://127.0.0.1:8090) | Inspect raw vs. redacted logs, egress proofs, pseudonym tables, and trigger microservice actions. |
| **Tier 2 SaaS Dashboard** | [http://127.0.0.1:8080](http://127.0.0.1:8080) | Multi-tenant monitoring, health trends, incidents, and LangGraph remediation approvals. |
| **Stream Simulator** | [http://127.0.0.1:7100](http://127.0.0.1:7100) | Injects chaos scenarios into the telemetry pipeline. |
| **Prometheus** | [http://127.0.0.1:9090](http://127.0.0.1:9090) | Prometheus metrics scraper. |
| **Product Service API** | [http://127.0.0.1:8081](http://127.0.0.1:8081) | Demo catalog and telemetry endpoints. |
| **Order Service API** | [http://127.0.0.1:8082](http://127.0.0.1:8082) | Order management and payment coordination. |
| **Payment Service API** | [http://127.0.0.1:8083](http://127.0.0.1:8083) | Payment processing and auth simulation. |

---

## 🧪 Interactive Chaos & Demo Scenarios

You can trigger scenarios directly from the **Tier 1 Local UI (http://127.0.0.1:8090)** buttons or via `curl`:

| Scenario | Trigger Command / UI Action | Expected Platform Behavior |
|---|---|---|
| **Secret in Logs** | Click **Secret in Log** or `curl -X POST localhost:8081/api/demo/leak` | Local UI shows raw vs redacted; SaaS privacy panel shows redaction count. Raw tokens never leave Tier 1. |
| **Auth Attack / Brute Force** | Click **Failed Logins (Auth Attack)** | Fires 25 failed attempts; warning security alert fires at 5, critical at 20; SaaS identifies `brute_force` and recommends firewall block on `src_id`. |
| **Network Latency Spike** | Click **latency spike** on the simulator section | Router latency spikes; knock-on latency is triggered on `app-01`; SaaS correlates network degradation with downstream impact. |
| **Application Slowdown** | `curl -X POST "localhost:8081/api/demo/slow?ms=1500"` | Diagnosed correctly as `app_latency` (latency spike without packet loss); triggers service restart runbook. |
| **CPU Drift Anomaly** | Click **cpu drift** | After ~2 minutes of baseline learning, EWMA detects drift before static thresholds breach. |
| **SaaS Disconnection** | `docker compose stop saas-ingest` | Gateway outbox safely buffers records locally. Restarting the ingest drains the buffer with zero data loss. |

---

## 🔍 Cryptographic Verification Tool

To prove complete data integrity and guarantee that every payload sent by the customer gateway was received without tampering or loss:

```bash
docker compose run --rm tier2-verify
```
**Expected Output:**
```text
PASS - zero loss, zero duplicates, hashes match
Sent: X | Received: X | Matched SHA-256: X | Lost: 0
```

---

## 📂 Repository Structure

```
├── docker-compose.yml              # Multi-tier orchestration definition
├── .env.example                    # Environment configuration template
├── TIER1.md                        # Tier 1 detailed technical documentation
├── tier2/
│   ├── TIER2.md                    # Tier 2 SaaS and AIOps documentation
│   ├── db/schema.sql               # PostgreSQL multi-tenant schema
│   ├── saas-app/                   # Spring Boot 3.5 SaaS ingestion & API services
│   └── aiops/                      # Python AIOps engine, LangGraph agent, and Gemini narrators
├── python-processor/               # Tier 1 local collector, egress gateway, and UI
│   ├── tier1/
│   │   ├── collector.py            # Local ingestion, redaction, and windowing
│   │   ├── gateway.py              # Egress gateway, schema enforcement, outbox
│   │   ├── adapter/                # Telemetry stream adapter
│   │   ├── sim/                    # Infrastructure chaos simulator
│   │   └── ui/                     # Local browser dashboard (static HTML/JS)
│   └── tests/                      # Pytest unit and integration test suite
├── product-service/                # Spring Boot demo microservice
├── order-service/                  # Spring Boot demo microservice
└── payment-service/                # Spring Boot demo microservice
```

---

## 🧪 Running Tests

Run the Tier 1 Python collector test suite (using the Docker test environment):
```bash
docker run --rm -v "${PWD}/python-processor:/app" -w /app tier1-python:latest pytest -q
```
All 29 test suites validate secret redactions, window calculations, pseudonymization, and alert thresholds.

---

## 📄 License
This project is developed as part of the Hackathon initiative. Distributed under the MIT License.
