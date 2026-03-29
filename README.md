# Project Specification: The Fault-Tolerant Edge Orchestrator

## Objective
Design and implement a decoupled Publisher-Subscriber (Pub/Sub) pipeline bridging a cloud service and an edge controller. The system must demonstrate resilience and data preservation during a simulated network partition.

## Technology Stack
* **Infrastructure:** Terraform, LocalStack
* **Containerization & Networking:** Docker, Docker Compose
* **Application Logic:** Python (boto3, redis-py)
* **Caching/Message Broker:** Redis, AWS SQS (via LocalStack)

---

## Phase 1: Infrastructure as Code (Terraform)
Provision the cloud-side message queues to handle asynchronous tasks and capture failure states.

* **Initialization:** Configure Terraform to target the LocalStack AWS endpoint (`http://localhost:4566`).
* **Resource Allocation:** Provision two standard Amazon SQS queues: `lab-commands-queue` and `lab-telemetry-queue`.
* **Failure State Mitigation (Dead Letter Queue):** Provision a Dead Letter Queue (DLQ). Configure the `lab-commands-queue` with a `redrive_policy` to route messages to the DLQ after 3 failed processing attempts. This prevents "poison pill" messages from infinitely looping and exhausting system resources.

## Phase 2: Network Topology & Security (Docker Compose)
Define strict operational boundaries between the cloud simulation and the local lab environment.

* **Service Definition:** Create a `docker-compose.yml` defining four services: `localstack`, `redis`, `cloud_app`, and `edge_app`.
* **Network Isolation:** Define two separate Docker networks: `cloud_net` and `edge_net`.
    * Attach `localstack` and `cloud_app` to `cloud_net`.
    * Attach `redis` and `edge_app` to `edge_net`.
    * Attach `edge_app` to `cloud_net` to simulate the outbound internet connection.
* **Security (OWASP):** Inject AWS credentials and connection strings via environment variables. Do not hardcode credentials in the application logic.
* **Testing Protocol:** Drop `edge_app` from `cloud_net` during runtime to simulate a Site-to-Site VPN failure or physical internet outage.

## Phase 3: The Cloud Publisher (Python)
Implement the cloud service to issue commands asynchronously, adhering to the Dependency Inversion Principle (SOLID).

* **Interface Definition:** Define an abstract base class `MessageBroker` with a `publish(message)` method.
* **Implementation:** Create an `SQSBroker` class utilizing `boto3` that implements the `MessageBroker` interface.
* **Execution:** Instantiate the `SQSBroker` and publish JSON payloads (e.g., `{"device": "PF400", "action": "move_to_home", "id": "123"}`) to the `lab-commands-queue`.
* **Testability:** Injecting the `MessageBroker` interface decouples the business logic from the AWS SDK, enabling the insertion of a `MockBroker` for zero-latency unit testing in CI/CD pipelines without network dependencies.

## Phase 4: The Edge Subscriber & Fallback (Python & Redis)
Develop the edge service to process commands and handle hardware latency and network drops gracefully.

* **Polling Mechanism:** Write a Python script for `edge_app` that continuously polls the `lab-commands-queue`. Implement exponential backoff to prevent API rate limiting.
* **Latency Simulation:** Simulate hardware execution time using `time.sleep(2)`.
* **Design Pattern (Circuit Breaker):** Implement a `try/except` block around the SQS polling logic. If LocalStack is unreachable, trip the circuit breaker.
* **Data Preservation & Time Complexity:** Upon network failure, redirect all generated telemetry data to the local `redis` instance using `redis-py`. Redis provides O(1) time complexity for writes, ensuring high-throughput sensor data is not lost while the cloud is unreachable.
* **Memory Management:** Configure Redis with a `maxmemory` policy (e.g., `allkeys-lru` or `volatile-ttl`) to drop the oldest telemetry data. This prevents an Out of Memory (OOM) fatal error if the network partition is prolonged.
* **Reconciliation:** Once the connection is restored, flush the Redis cache to the `lab-telemetry-queue`.