# Agents and Responsibilities
This agents document defines the responsibilities and interactions of each microservice (agent) in the proof‑of‑concept system.  It serves as a reference for developers and testers when implementing and coordinating the event‑driven authorisation flow. 

The follow sections define key agents/services and their responsibilities. These are defined using the format:

```
### [SERVICE NAME]
[DESCRIPTION]

#### Inputs
* [Input expectation #1]
* [Input expectation #2 and so on]

#### Outputs
* [Output expectation #1]
* [Output expectation #2 and so on]

#### Key Considerations
* [Key consideration #1]
* [Key consideration #2 and so on]
```

If any agent definitions do not match the pattern above, notify the user so that they can verify and correct the definition.

## Agent/Services

### Scheme Simulator (Producer)
Generates random **CAIN.001.001.04 AuthorisationInitiationV04** messages to simulate incoming transaction requests.  Builds ISO 20022 XML using Java (e.g., Prowide or JAXB) and posts to the Orchestrator’s REST endpoint.  Waits for **CAIN.002.001.04 AuthorisationResponseV04** responses.

#### Inputs
* None (internally generates requests)

#### Outputs
* HTTP POST to Orchestrator
* Awaits HTTP response from the above POST.

#### Key Considerations
* Should randomize amounts, currencies, merchant category codes, card tokens and set message IDs (by using UUIDs)
* Optionally read sample messages from files.
* Must track correlation IDs to match responses.
* After 3 seconds, if a response has not been received, the request to the Orchestrator should time out. At this point, an error should be logged indicating the scheme had to 'stand in'. The service should continue with the next request and not terminate fully.

### Authorisation Orchestrator
The central mediator and entry point into the event‑driven flow.  Accepts incoming ISO messages, validates the Business Application Header (BAH), converts message payloads to Avro `TransactionEvent` records and publishes them to Kafka.  Tracks responses via correlation ID, converts Avro decision events back to CAIN.002 messages, and responds to the simulator.

#### Inputs
* HTTP requests from simulator
* Kafka topics (`transactions.decision.response`, `transactions.postauth.response`)

#### Outputs
* Kafka events on stage topics
* HTTP response to simulator

#### Key Considerations
* Should be reactive/non‑blocking.
* Must enforce a timeout (≤ 3 s) when awaiting the final decision.
* Responsible for registering Avro schemas and using the Schema Registry for serialization.

### Security Check Service
Performs card security validations such as ARQC verification, card status, CVV/CVC check and expiry.  Reads events from `transactions.security.request`, augments the record with a `securityStatus` and publishes to `transactions.preauth.request`.

#### Inputs
* Kafka topic `transactions.security.request`

#### Outputs
* Kafka topic `transactions.preauth.request`

#### Key Considerations
* Should use Avro `SpecificRecord` classes for input and output.
* Keep processing fast (cryptographic checks may be CPU intensive).
* Maintain an in‑memory or mock database of card and key data.

### Fraud Pre‑Authorisation Service
Performs fraud screening and risk scoring.  Applies rules such as transaction velocity, geolocation mismatches, merchant risk categories and amount thresholds.  Appends a `fraudStatus` to the record.  Publishes the result to `transactions.decision.request`  with a check type of 'fraud'.

#### Inputs
* Kafka topic `transactions.preauth.request`

#### Outputs
* Kafka topic `transactions.decision.request`

#### Key Considerations
* Should be stateless or use an in‑memory cache for rule thresholds.
* Should use some basic rule evaluations. This is not meant to be a full fraud check but a basic demonstration.
* Ensure the service remains scalable since fraud checks are often the bottleneck.

### Merchant Loop Pre‑Authorisation Service
Performs a check to see if the merchant details (e.g. merchant ID, MCC) match approve or decline rules. Appends a `merchantLoopStatus` to the record.  Publishes the result to `transactions.decision.request` with a check type of 'merchantLoop'.

#### Inputs
* Kafka topic `transactions.preauth.request`

#### Outputs
* Kafka topic `transactions.decision.request`

#### Key Considerations
* Should be stateless or use an in‑memory cache for merchant loop configuration.
* Start with some basic rules. For example, if a MCC is a gambling related MCC, we reject the transaction.

### Decision Service (Aggregator)
Aggregates security and pre-auth check results to arrive at the final approve/decline decision.  Assigns an authorisation code for approvals and reason codes for declines.  Publishes events to both `transactions.postauth.request` and `transactions.decision.response`.

#### Inputs
* Kafka topic `transactions.decision.request`

#### Outputs
* Kafka topics `transactions.decision.response` and `transactions.postauth.request`

#### Key Considerations
* Implemented with Spring Kafka over Kafka Streams. After the proof of concept, this would still be a Java microservice.
* Must correlate events by transaction ID and produce decisions deterministically.

### Post‑Authorisation/Notification Service
Handles post‑authorisation tasks.  At this stage a service could send SMS/email notifications, update an internal ledger/billing system, or trigger settlement processes. For this PoC, we will mock out send an email notification.  Publishes confirmation events to `transactions.postauth.response`.

#### Inputs
* Kafka topic `transactions.postauth.request`

#### Outputs
* Kafka topic `transactions.postauth.response`

#### Key Considerations
* Should integrate with external APIs or mock services for notifications.
* Runs asynchronously; its failure should not block final decision response.

### Kafka Cluster & Schema Registry
Provides event streaming and schema governance.  Hosts topics for each stage and persists events for replay or debugging.  Schema Registry stores Avro schemas and enforces compatibility.

#### Inputs
* Event producers and consumers

#### Outputs
* Event consumers and producers

#### Key Considerations
* Run in Docker containers
* Use Confluent docker images
* Run in Kraft mode so there is no need for a Zookeeper instance
* Configure proper partitions and replication factor.
* Secure Kafka and Schema Registry with TLS/SASL.
* Use compatibility mode in Schema Registry (e.g., BACKWARD).

### Prometheus/Grafana (Monitoring Agent)
Collects metrics (throughput, latency, errors) from all services via Micrometer.  Grafana dashboards visualise performance.

#### Inputs
* Metrics endpoints from services

#### Outputs
* Dashboards and alerts

#### Key Considerations
* Run as Docker containers
* Use exporters (JMX exporter, Spring Boot Actuator).
* Configure alerting for high latency or error rates.

### Test Harness (Load Generator)
Automated scripts using JMeter or Gatling to send concurrent requests to the orchestrator and measure response times and throughput.

#### Inputs
* Prepared ISO 20022 request dataset

#### Outputs
* Reports and metrics

#### Key Considerations
* Simulates peak loads and ensures the system stays within the 3 second SLA.

### GitHub CI (Automation Agent)
Defines automated pipelines for building, testing and validating the project on GitHub.

#### Inputs
* Pushes and pull requests to the GitHub repository (e.g. `main` and feature branches)
* Workflow configuration in `.github/workflows/*.yml`

#### Outputs
* CI build status checks on pull requests
* Build artifacts (e.g. JARs, test reports) for debugging

#### Key Considerations
* Use GitHub Actions workflows to:
  * Build all modules with the shared parent (Maven/Gradle) on Java 17.
  * Run unit tests for all services and modules on every push and pull request.
  * Optionally run integration tests (e.g. spinning up Kafka/Schema Registry via Docker) on pull requests or nightly.
  * Validate Avro schema generation and optionally run schema compatibility checks.
  * Integrate into SonarQube Cloud for code quality scans. Access tokens need to be set in secrets/environment variables.
  * Implement a check against the NIST CVE API (https://nvd.nist.gov/developers/vulnerabilities) via Maven. Access tokens need to be set in secrets/environment variables.
  * Enforce basic quality gates (e.g. test pass, no build warnings treated as errors where feasible).
* Protect the `main` branch so pull requests must pass CI before merge.
* Use separate workflows or jobs for fast feedback (unit tests) vs slower checks (integration/load tests).

## Interaction Summary

1. The **Scheme Simulator** sends an ISO 20022 **Authorisation Initiation** to the **Authorisation Orchestrator**.
2. The **Orchestrator** converts the request to an Avro `TransactionEvent`, registers the schema if necessary and publishes the event to Kafka (`transactions.security.request`).
3. The **Security Check Service** consumes the event, performs security validations and publishes the augmented record to `transactions.preauth.request`.
4. The **Fraud Service** processes the pre‑authorisation event and publishes to `transactions.decision.request`.
5. In parallel, the **Merchant Loop Service** processes the pre‑authorisation event and publishes to `transactions.decision.request`.
6. The **Decision Service** aggregates results, produces an approve/decline decision and publishes to two topics: `transactions.postauth.request` (for post‑processing) and `transactions.decision.response` (for the orchestrator).
7. The **Orchestrator** listens to `transactions.decision.response` for the correlation ID, creates a **CAIN.002.001.04** response message and returns it to the simulator.
8. Meanwhile, the **Post‑Auth Service** performs notifications or ledger updates and publishes a confirmation to `transactions.postauth.response`.
9. **Monitoring agents** collect metrics at every stage to observe throughput and latency.

## Coding Standards

### General
- Use Java 17+ and Spring Boot for all microservices.
- One microservice per bounded context described in this document.
- Prefer small, single-responsibility classes and methods.
- Use spaces for indentation
- Follow the standard Google Java code format rules

### Project Structure & Naming
- The base package to use should be `uk.co.ltheobald.parallelpoc`
- Package by feature (e.g. `uk.co.ltheobald.parallelpoc.security`, `uk.co.ltheobald.parallelpoc.fraud`), not strict layers.
- Use descriptive class names (`SecurityCheckService`, `FraudRuleEngine`) rather than abbreviations.
- Kafka topic names must follow `transactions.<stage>.<direction>` as documented in `agents.md`.

### API & Contracts
- Use Avro `SpecificRecord` for all Kafka messages.
- All public APIs (REST and Kafka) must have versioned schemas (`v1`, `v2`, etc.).
- Never break schema compatibility; use backward-compatible changes only.

### Error Handling & Logging
- Never swallow exceptions: log with context and propagate or map to a clear error response.
- Use JSON structured logging with `correlationId` / `transactionId` on every log line.
- Keep JSON logging to a single line when used in a developer/local context (e.g. output in an IDE)
- No `System.out.println`; always use the project logging abstraction (e.g. SLF4J).

### Configuration
- Externalise configuration (Kafka endpoints, timeouts, credentials) via environment variables or config files.
- Do not hard-code secrets or auth tokens in code or `agents.md`.

### Testing
- Each service must have unit tests for core decision logic (e.g. fraud rules, merchant rules, security checks).
- Add contract tests for Kafka message formats and REST endpoints.
- Critical paths (authorisation flow, timeouts) should have integration tests using embedded Kafka or testcontainers (where feasible).

### Observability
- Expose metrics via Micrometer and Spring Boot Actuator.
- Include latency, throughput and error-rate metrics per topic and per service.
