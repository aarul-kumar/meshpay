# MeshPay: Distributed Offline Payment & Settlement Engine

MeshPay is a distributed offline payment settlement engine focused on cryptographic integrity, deferred synchronization, and exactly-once settlement under intermittent connectivity.

Encrypted transaction payloads propagate peer-to-peer across mesh-connected nodes until an internet-enabled bridge node synchronizes settlement with the backend infrastructure. 

The architecture focuses on backend reliability patterns such as fault-tolerant transaction routing, hybrid cryptography, and idempotent settlement under unreliable network conditions.

## Why Offline Settlement Matters

Imagine two users attempting to exchange money inside a subway tunnel, disaster zone, or rural area without cellular coverage.

MeshPay models how encrypted payment instructions can propagate device-to-device across a local mesh network until any participant regains internet connectivity and synchronizes the transaction with the backend ledger.

---

# Key Features

## Hybrid End-to-End Encryption
Transactions maintain strict confidentiality and data integrity across untrusted intermediaries using a dual-layered cryptographic implementation:
* **Key Exchange:** RSA-OAEP (2048-bit) securely wraps and transmits ephemeral session keys.
* **Authenticated Encryption:** AES-256-GCM seals the dynamic transaction data, generating an explicit authentication tag.

Intermediate routing nodes can relay packets across the ad-hoc network but are structurally prevented from reading transaction contents, modifying data fields, or forging settlement requests.

## Idempotent Concurrent Settlement
In decentralized mesh environments, identical data packets routinely flood the backend through overlapping bridge gateways. MeshPay guarantees **exactly-once processing** via an early-stage defensive pipeline:
* **Pre-Decryption Evaluation:** Ingested packets are uniquely fingerprinted using a SHA-256 hash of their raw *ciphertext*.
* **Atomic Compare-and-Set:** An in-memory cache checks uniqueness via atomic operations, modeling distributed Redis `SETNX` semantics.
* Racing duplicate packets are immediately short-circuited and dropped (`DUPLICATE_DROPPED`) before executing expensive decryption or database actions.

## Distributed Mesh Routing
The system models a decentralized packet management topology featuring:
* Peer-to-peer opportunistic packet propagation via a gossip communication protocol.
* Multi-hop routing controls with dynamic Time-To-Live (TTL) limits to prevent infinite data loops.
* Edge bridge nodes (`hasInternet=true`) that buffer and flush accumulated payloads into the cloud ledger once network connectivity is recovered.

## Transaction Settlement Pipeline
The backend gateway processes every incoming payload through a zero-trust sequence:
1. **Ciphertext Hashing:** Calculates a SHA-256 footprint of the raw encrypted data block.
2. **Idempotency Claim:** Atomically checks the in-memory registry to drop duplicates early.
3. **Hybrid Decryption:** Unwraps the ephemeral session key via RSA-OAEP and decrypts the core message block.
4. **Integrity Verification:** Evaluates the AES-GCM authentication tag to catch any bit-level payload tampering.
5. **Freshness Validation:** Compares signed internal timestamps against a strict expiration window.
6. **Atomic Account Settlement:** Executes account debits and credits within an isolated database transaction.
7. **Ledger Persistence:** Commits a historical trace log containing the packet hash and settlement metadata.

---

# System Architecture

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         OFFLINE CLIENT NODE                             │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ Encrypt via Server's RSA Public Key                      │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Peer-to-Peer Mesh Propagation
                                       ▼
        ┌─────────┐  Hop   ┌─────────┐  Hop   ┌─────────┐
        │virtual- │ ─────▶ │virtual- │ ─────▶ │virtual- │ ◀── Bridge Node 
        │device-1 │        │device-2 │        │device-3 │     Regains 4G/Internet
        └─────────┘        └─────────┘        └────┬────┘
                                                   │
                                                   ▼ HTTPS POST Ingestion
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT CORE LEDGER BACKEND                     │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] SHA-256 Ciphertext Footprint Generation                            │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── Atomic Concurrent register     │
│       │                                  Short-circuits duplicates      │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt()                                      │
│       │  └─ RSA-OAEP unpacks unique AES session key                     │
│       │  └─ AES-256-GCM decodes transaction and validates Integrity Tag│
│       ▼                                                                 │
│  [4] Timestamp Validation (Replay checking window)                     │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│          @Transactional: Atomic Balance Credit/Debit Modifications      │
│          Optimistic Locking Safeguard via JPA @Version Verification     │
└─────────────────────────────────────────────────────────────────────────┘

```

---

# Key Problems & Solutions

### Problem 1: Untrusted Intermediaries (Data Confidentiality & Integrity)

Because asymmetric RSA processing features strict structural data limits (~245 bytes for 2048-bit keys), it cannot directly encrypt dynamic JSON strings. MeshPay implements a standard hybrid encryption envelope pattern:

1. The client generates a unique, single-use **AES-256 key** for the transaction payload.
2. The payload string is encrypted using **AES-256-GCM**, producing the ciphertext block and a 16-byte authentication tag.
3. The server's public key seals the ephemeral AES key using **RSA-OAEP**.
4. The components are packed into a uniform transport block: `[256-byte RSA wrapped key][12-byte Initialization Vector (IV)][Ciphertext + 16-byte GCM tag]`.

Because AES-GCM provides fully authenticated encryption, any bit-level tampering by an intermediary carrier triggers a cryptographic exception during the backend decryption phase, rejecting the message before it touches business logic layers.

### Problem 2: Concurrent Duplicate Ingestion (The Duplicate-Storm Problem)

When multiple users walk out of an offline zone back into cellular coverage simultaneously, their devices execute parallel HTTPS POST requests to `/api/bridge/ingest` containing overlapping transaction sets.

By checking an atomic registry against the **ciphertext hash** rather than internal plain text IDs:

* The system avoids wasting CPU cycles on asymmetric RSA decryption for entries destined to be dropped.
* It neutralizes malicious carrier attacks where internal transaction UUIDs are modified to bypass filters. Since altering any part of the ciphertext breaks the GCM envelope authentication check, any payload tampering is detected immediately on decryption.
* Legitimate deliveries of the same packet produce identical ciphertext because the packet contents, AES session key, and IV remain unchanged across retransmissions.
* **Database Fallback:** The backend implements a unique index constraint on `transactions.packet_hash`, ensuring that even if an in-memory cache layer fails under extreme race conditions, the transaction table forces an ACID rollback on duplicate insertions.

### Problem 3: Replay Attack Mitigation

Attackers who capture transient data payloads out of the air cannot record a ciphertext block and re-broadcast it later to continuously drain user balances. MeshPay implements a two-tier protection check:

* **Timestamp Validation:** The encrypted envelope holds an immutable `signedAt` epoch millisecond field. The gateway service rejects any packets older than 24 hours. The signature cannot be updated by an attacker without invalidating the AES-GCM authentication tag.
* **Unique Nonce Generation:** Every generated payload is stamped with a distinct UUID nonce. Even if an account holder submits two identical back-to-back payments of ₹500, the changing nonces force distinct ciphertexts and distinct SHA-256 footprints, allowing safe, independent processing.

---

# Project Structure & Component Mapping

```text
meshpay/
├── pom.xml                                  # Dependencies & Lifecycle (Spring Boot 3.3, Java 17)
├── mvnw, mvnw.cmd                           # Native execution wrappers (Zero-install configuration)
└── src/
    ├── main/
    │   ├── java/com/demo/upimesh/
    │   │   ├── UpiMeshApplication.java      # Application bootstrap entry point
    │   │   │
    │   │   ├── config/
    │   │   │   └── AppConfig.java           # @EnableScheduling cache eviction configuration
    │   │   │
    │   │   ├── controller/
    │   │   │   ├── ApiController.java       # Gateway ingestion REST endpoints & system telemetry
    │   │   │   └── DashboardController.java # Interface routing engine
    │   │   │
    │   │   ├── crypto/
    │   │   │   ├── ServerKeyHolder.java     # Automated RSA keypair lifecycle generation
    │   │   │   └── HybridCryptoService.java # Asymmetric/Symmetric handling & SHA-256 hashing
    │   │   │
    │   │   ├── model/
    │   │   │   ├── Account.java             # JPA Entity managing optimistic locking (@Version)
    │   │   │   ├── Transaction.java         # Ledger table tracking unique packet hashes
    │   │   │   ├── MeshPacket.java          # Envelope wire frame (Opaque ciphertext + routing meta)
    │   │   │   └── PaymentInstruction.java  # Decrypted schema mapping plaintext payloads
    │   │   │
    │   │   └── service/
    │   │       ├── BridgeIngestionService.java # Transaction orchestrator (Hash -> Claim -> Decrypt -> Settle)
    │   │       ├── IdempotencyService.java  # Atomic lock management matching Redis SETNX behavior
    │   │       ├── SettlementService.java   # ACID balance operations over database resources
    │   │       ├── MeshSimulatorService.java # Network topology simulation and routing logic
    │   │       ├── DemoService.java         # Seeds initial accounts and initializes packet generation
    │   │       └── VirtualDevice.java       # Represents a single simulated phone node in the mesh
    │   │
    │   └── resources/
    │       ├── application.properties       # Persistent database parameters and application ports
    │       └── templates/
    │           └── dashboard.html           # Management and observation control panel
    └── test/java/com/demo/upimesh/
        └── IdempotencyConcurrencyTest.java  # Parallel thread race condition test harness

```

---

# Technology Stack

* **Backend Framework:** Java 17, Spring Boot 3.3, Spring Data JPA
* **Database Layer:** H2 Embedded Relational Database Engine
* **Cryptography Toolkit:** RSA-2048 (RSA-OAEP), AES-256-GCM, SHA-256 Hashing
* **Concurrency Controls:** Lockless atomic compare-and-set operations via `ConcurrentHashMap`, Database Optimistic Locking via JPA `@Version`
* **UI Layer:** Thymeleaf, Vanilla JavaScript, HTML5 / CSS3

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

# Scaling Architecture

The core application code separates pure business logic from external infrastructure dependencies. Transitioning this system from a self-contained local model to a horizontally scaled architecture involves swapping out components as follows:

| Component Profile | Local In-Memory Model | Scaled Infrastructure |
| --- | --- | --- |
| **Persistence Layer** | Embedded H2 Database Engine | Relational Cloud Datastore (e.g., PostgreSQL / AWS Aurora) |
| **Idempotency Cache** | Localized `ConcurrentHashMap` | Distributed Cache Cluster Engine (**Redis** via `SET NX EX`) |
| **Key Management** | In-Process Asymmetric Generation | Hardware Security Modules (**HSM** via AWS KMS / HashiCorp Vault) |
| **Network Channels** | In-Memory Object Mesh Mapping | Physical Transport Interfaces (**BLE GATT** / Wi-Fi Direct) |
| **Authentication Handshakes** | Open Entry Ingestion Gateways | Mutual TLS (**mTLS**) / Signed Bridge Node Certificates |

---

# System Trade-offs & Limitations

MeshPay intentionally focuses on core transaction routing safety and settlement correctness under intermittent connectivity conditions. These constraints represent real-world distributed systems trade-offs (favoring Availability over Consistency under the CAP theorem):

1. **The Ledger Blindspot:** Because payment signatures happen entirely off-grid, a recipient device cannot verify if the sender actually possesses the required funds in their central bank account. The payment confirmation on the sender's device represents an encrypted, signed IOU rather than a cleared transaction. If the sender's account balance is insufficient when the packet finally reaches the backend, the settlement will be marked as `REJECTED`. Real offline implementations (like *UPI Lite*) mitigate this by using a pre-funded, hardware-backed wallet to cryptographically guarantee fund availability offline.
2. **Offline Double-Spending Risk:** A malicious actor can exploit the offline window by sending the same ₹500 balance to Node A in one location, and then routing that identical balance payload to Node B in another disconnected area. Whichever packet achieves the fastest propagation path to the backend wins settlement validation, while trailing entries are rejected at the edge layer.
3. **Physical Radio Constraints:** Real background Bluetooth Low Energy (BLE) routines introduce device-specific engineering hurdles, such as aggressive background service power-throttling on Android and strict peripheral-mode access policies inside iOS. This repository abstracts those physical-layer behaviors to focus on the cryptographic and settlement pipeline.

---
