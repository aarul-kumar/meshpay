# MeshPay - Distributed Offline Payment & Settlement Engine

MeshPay is a distributed offline payment settlement engine that makes digital payments possible even when users temporarily lose internet connectivity.

Instead of sending money directly through the internet, encrypted transaction packets propagate device-to-device across a local mesh network until any bridge node reconnects online and synchronizes settlement with the backend ledger.

The project focuses on backend engineering concepts such as:

- Secure hybrid cryptography
- Idempotent transaction processing
- Exactly-once settlement
- Offline synchronization
- Replay attack protection
- Concurrent duplicate prevention
- Fault-tolerant backend design

<img width="1114" height="522" alt="meshpay" src="https://github.com/user-attachments/assets/9935e9bd-9c23-4218-a454-6eac08ab1db6" />


## Why Offline Settlement Matters

Imagine two users attempting to send money inside:

- A subway tunnel
- A disaster zone
- A rural area with weak signal
- A crowded stadium with no network connectivity

Normally, digital payments fail completely without internet access.

Through MeshPay, encrypted payment instructions can still propagate across nearby devices and later reach the backend once any participant regains internet connectivity.

---

# Key Features

## Hybrid End-to-End Encryption
Every transaction is encrypted before leaving the sender’s device.

MeshPay implements a hybrid encryption model using:

- **RSA-2048 (RSA-OAEP)** for secure AES session key exchange
- **AES-256-GCM** for authenticated transaction encryption
- **SHA-256 hashing** for duplicate detection and integrity tracking

Because RSA cannot efficiently encrypt large dynamic transaction data directly, the system uses a hybrid encryption envelope:

1. A temporary AES-256 session key encrypts the transaction data
2. The AES key is securely encrypted using the server’s RSA public key
3. AES-GCM generates an authentication tag that detects any tampering

Intermediate devices can forward encrypted packets, but they cannot:

- Read transaction contents
- Modify sender or receiver data
- Alter transaction amounts
- Forge settlement requests

If even one bit changes inside the encrypted transaction, AES-GCM integrity verification fails and the backend rejects it.

## Idempotent Concurrent Settlement
Offline mesh environments naturally create duplicate packet delivery.

The same transaction may arrive simultaneously from multiple bridge devices once connectivity is restored.

MeshPay guarantees exactly-once settlement using:

- SHA-256 ciphertext hashing
- Atomic compare-and-set operations
- In-memory idempotency tracking
- Database-level uniqueness constraints

Duplicate packets are rejected before expensive decryption or settlement logic executes.

The idempotency layer models how distributed systems commonly use Redis `SETNX` semantics for atomic uniqueness claims.

## Distributed Mesh Routing
The system simulates decentralized transaction propagation using:

- Peer-to-peer forwarding
- Gossip-style packet propagation
- Multi-hop routing
- Time-To-Live (TTL) limits
- Internet-enabled bridge nodes

A transaction can travel across multiple offline devices before eventually reaching the backend settlement service.

## Transaction Settlement Pipeline
Every incoming packet passes through a zero-trust validation pipeline:

1. Generate SHA-256 hash of encrypted packet
2. Atomically claim idempotency ownership
3. Decrypt transaction securely
4. Validate AES-GCM integrity tag
5. Verify timestamps and replay windows
6. Execute atomic account settlement
7. Persist transaction history safely

Settlement operations use:

- Spring transactional boundaries
- Optimistic locking via JPA `@Version`
- ACID-compliant database updates

---

# System Architecture

<img width="1453" height="1780" alt="Blank diagram" src="https://github.com/user-attachments/assets/d34d53e5-bfb1-4cab-b45b-3014e51e8f59" />

---

# Key Problems & Solutions

### Problem 1: Untrusted Intermediaries (Data Confidentiality & Integrity)

Transactions travel through multiple unknown intermediary devices before reaching the backend.

To preserve confidentiality and integrity:

- AES-256 encrypts transaction payloads
- RSA-OAEP securely wraps AES session keys
- AES-GCM authentication tags detect tampering automatically

Because AES-GCM provides authenticated encryption, any payload modification immediately invalidates the integrity check during backend decryption.

### Problem 2: Concurrent Duplicate Ingestion (The Duplicate-Storm Problem)

When multiple bridge nodes reconnect simultaneously, identical packets may flood the backend concurrently.

MeshPay prevents duplicate settlement using:

- SHA-256 ciphertext fingerprints
- Atomic compare-and-set registration
- Database unique constraints
- Optimistic locking safeguards

Only one settlement succeeds.

All duplicates are safely rejected.

### Problem 3: Replay Attack Mitigation

Attackers should not be able to capture encrypted packets and replay them later.

MeshPay mitigates replay attacks using:

- Signed timestamps
- Expiration windows
- Unique transaction nonces

Old or reused packets are rejected automatically.

Even two identical ₹500 payments generate different encrypted payloads because every transaction contains a unique nonce.

---

# Technology Stack

| Layer | Technologies Used |
|------|------------------|
| **Backend Framework** | Java 17, Spring Boot 3.3, Spring Data JPA |
| **Database Layer** | H2 Embedded Relational Database Engine |
| **Cryptography Toolkit** | RSA-2048 (RSA-OAEP), AES-256-GCM, SHA-256 Hashing |
| **Concurrency Controls** | Lockless atomic compare-and-set operations via `ConcurrentHashMap`, Database Optimistic Locking via JPA `@Version` |
| **UI Layer** | Thymeleaf, Vanilla JavaScript, HTML5 / CSS3 |

---

# Installation

### Prerequisites

Ensure your machine has the Java 17 Development Kit (or newer) installed and on your system path:

```bash
java -version

```

### Execution Steps

1. Clone the repository and navigate into the project root:
```bash
git clone https://github.com/aarul-kumar/meshpay.git
cd meshpay

```


2. Launch the application server instance:
* **Windows:**
```cmd
.\mvnw.cmd spring-boot:run

```


* **macOS / Linux:**
```bash
chmod +x mvnw
./mvnw spring-boot:run

```




3. Open the dashboard UI in your browser at: `http://localhost:8080`

### Running the Concurrency Tests

To execute the automated integration test suite and verify system behavior under concurrent load:

```bash
./mvnw test

```

This launches `IdempotencyConcurrencyTest`, spawning parallel threads hitting `BridgeIngestionService.ingest()` at the exact same millisecond with identical packets. It verifies that exactly one settles successfully, two are dropped as duplicates, and the sender's account balance is debited exactly once.

---

# Transaction Lifecycle (Step-by-Step Flow)

The dashboard provides manual control over the four core phases of the system pipeline:

* **Step 1 — Compose & Inject:** Select an offline sender, receiver, valuation, and security PIN, then click **Inject into Mesh**. The backend acts as the sender's phone, constructing a `PaymentInstruction` with a unique nonce and timestamp, sealing it via the hybrid crypto envelope, and placing it inside `phone-alice` (an offline virtual device).
* **Step 2 — Gossip Propagation:** Click **Run Gossip Round**. Each round, every virtual device broadcasts its accumulated packets to all adjacent peer nodes within radio proximity, decrementing the packet TTL counter on every hop to control network overhead.
* **Step 3 — Bridge Synchronization:** Click **Bridges Upload to Backend**. This simulates a designated gateway node (`phone-bridge`) moving back into a zone with internet connectivity. The bridge issues an HTTPS POST request containing its collected mesh payloads directly to the `/api/bridge/ingest` endpoint.
* **Step 4 — Verify Idempotency:** Reset the mesh, inject a packet, and run the gossip rounds multiple times so that all virtual devices hold identical packet duplicates. Click the sync buttons to observe the ledger: exactly one payload processes to a `SETTLED` state, while concurrent duplicates are discarded instantly.

---

# API Endpoints Reference

### 1. Packet Ingestion Contract

* **Endpoint:** `POST /api/bridge/ingest`
* **Headers:**
```http
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

```


* **Payload Structure:**
```json
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "eyJWZWN0b3IiOiJNZXNoUGF5IiwiQ3J5cHRvIjp0cnVlLCJQYXlsb2FkIjoiaHVnZS1ibG9iIn0="
}

```


* **Response Payload (`200 OK`):**
```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9b4d2e1f0a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9",
  "reason": null,
  "transactionId": 42
}

```


*(Alternative outcome responses return `DUPLICATE_DROPPED` or `INVALID`)*

### 2. Control Endpoints

| Method | Endpoint | Operational Profile |
| --- | --- | --- |
| **GET** | `/` | Serves the interactive dashboard UI. |
| **GET** | `/api/accounts` | Fetches current ledger parameters and balances for all accounts. |
| **GET** | `/api/transactions` | Fetches historical log entries from the settled transaction ledger. |
| **GET** | `/api/mesh/state` | Inspects current data registries across all active virtual mesh nodes. |
| **POST** | `/api/demo/send` | Simulates a client phone wrapping and encrypting a transaction packet. |
| **POST** | `/api/mesh/gossip` | Executes a single gossip replication wave across the network graph. |
| **POST** | `/api/mesh/flush` | Signals internet-enabled bridges to upload their packets to the backend. |
| **POST** | `/api/mesh/reset` | Resets the in-memory cache layer, node registries, and ledger state. |
| **GET** | `/h2-console` | Directly accesses the embedded relational database engine console. |

---

# Limitations

MeshPay intentionally focuses on core transaction routing safety and settlement correctness under intermittent connectivity conditions. These constraints represent real-world distributed systems trade-offs (favoring Availability over Consistency under the CAP theorem):

1. **Offline Balance Verification Limitation:** Because transactions are created entirely offline, recipient devices cannot verify whether the sender actually has sufficient funds at that moment. The payment behaves more like an encrypted pending request until it finally reaches the backend. If the sender’s balance is insufficient during settlement, the transaction is rejected. Real systems like UPI Lite reduce this problem using pre-funded offline wallets.
2. **Offline Double-Spending Risk:** A malicious user could attempt to spend the same balance multiple times while devices are disconnected. For example, the same ₹500 transaction could be routed through different offline paths simultaneously. The first valid transaction reaching the backend succeeds, while later duplicates are rejected.
3. **Real Device Networking Constraints:** Actual offline communication systems using Bluetooth Low Energy (BLE) face additional challenges such as battery optimization limits, unstable connectivity, and mobile OS background restrictions. This project simplifies those physical networking behaviors to focus mainly on cryptography, routing, and settlement logic.
