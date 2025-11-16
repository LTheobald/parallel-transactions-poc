# Implementation Plan

This document tracks implementation work as a set of phases and concrete tickets.  
Use the checkboxes to track status (`[ ]` = Todo, `[x]` = Done).

---

## Phase 1 – Foundations
Goal: establish shared build, schemas, and local infrastructure.

### Tickets
- [ ] **P1-01** – Ensure our project has the basics. Add a .gitignore that is suited for Java, IntelliJ & VSCode. Add a simple README.
- [ ] **P1-02** – Create parent build (Maven/Gradle) with Java 17, Spring Boot, Kafka, Avro, and Micrometer dependencies.
- [ ] **P1-03** – Add `common-avro` module with Avro schemas for `TransactionEvent`, decision, and post-auth events.
- [ ] **P1-04** – Configure Avro code generation and Schema Registry client in `common-avro`.
- [ ] **P1-05** – Create `infra` module with Docker Compose for Kafka (KRaft) and Schema Registry as described in `agents.md`.
- [ ] **P1-06** – Add a minimal CI job to build all modules and run unit tests.

---

## Phase 2 – Core Flow
Goal: implement the main authorisation path end-to-end.

### Tickets
- [ ] **P2-01** – Implement `orchestrator-service` skeleton (Spring Boot app and HTTP endpoint for ISO 20022 messages).
- [ ] **P2-02** – Implement ISO 20022 parsing and mapping to Avro `TransactionEvent` in `orchestrator-service`.
- [ ] **P2-03** – Implement Kafka producer in `orchestrator-service` to publish to `transactions.security.request` with correlation IDs.
- [ ] **P2-04** – Implement `security-service`: consume `transactions.security.request`, apply security checks, and produce to `transactions.preauth.request`.
- [ ] **P2-05** – Implement `decision-service`: consume `transactions.decision.request`, aggregate by transaction ID, and produce to `transactions.decision.response` and `transactions.postauth.request`.
- [ ] **P2-06** – Implement correlation and response handling in `orchestrator-service`, including the 3-second timeout path and error response.
- [ ] **P2-07** – Add integration tests for the core flow (`orchestrator` → `security` → `decision`).

---

## Phase 3 – Enrichment Services
Goal: add fraud and merchant enrichment plus post-authorisation handling.

### Tickets
- [ ] **P3-01** – Implement `fraud-service`: consume `transactions.preauth.request`, apply basic fraud rules, set `fraudStatus`, and publish to `transactions.decision.request`.
- [ ] **P3-02** – Implement `merchant-loop-service`: consume `transactions.preauth.request`, apply merchant rules, set `merchantLoopStatus`, and publish to `transactions.decision.request`.
- [ ] **P3-03** – Implement `postauth-service`: consume `transactions.postauth.request`, perform mock notification, and publish to `transactions.postauth.response`.
- [ ] **P3-04** – Add integration tests covering parallel fraud and merchant-loop enrichment feeding into `decision-service`.

---

## Phase 4 – Tooling & Non-Functionals
Goal: provide simulation, load testing, metrics, and dashboards.

### Tickets
- [ ] **P4-01** – Implement `scheme-simulator` that generates random ISO 20022 requests and calls `orchestrator-service`.
- [ ] **P4-02** – Implement `test-harness` (JMeter/Gatling project) with configurable concurrency and a request dataset.
- [ ] **P4-03** – Expose Micrometer metrics in all services (HTTP, Kafka, and key business metrics).
- [ ] **P4-04** – Extend `infra` with Prometheus and Grafana, plus dashboards for latency, throughput, and error rates.
- [ ] **P4-05** – Create load-test scenarios and baseline reports verifying the 3-second end-to-end SLA.

---

## Phase 5 – Hardening
Goal: improve resilience, compatibility, and documentation.

### Tickets
- [ ] **P5-01** – Add failure scenario tests (Kafka outages, partial service failures, timeouts) and verify graceful degradation.
- [ ] **P5-02** – Add schema evolution tests for Avro messages, enforcing backward-compatible changes.
- [ ] **P5-03** – Review implementation against Coding Standards; create issues for any gaps and address critical ones.
- [ ] **P5-04** – Finalise documentation: update `agents.md`, `plans.md`, and add per-service READMEs.
