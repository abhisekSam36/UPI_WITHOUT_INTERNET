UPI Offline Mesh — Backend Service

A Spring Boot backend demonstrating deferred, offline UPI payments routed through a device-to-device mesh network. This system enables users in zero-connectivity environments to execute cryptographically secure payments that hop across nearby devices until a bridge node regains internet access and silently settles the transaction with this server. 

This repository contains the backend settlement engine and a software-based mesh simulator to demonstrate the complete transaction lifecycle locally.

## Core Engineering Proofs

This system validates three critical distributed systems requirements:
1. **Zero-Trust Routing:** Payments travel through untrusted intermediaries without exposing payload data or allowing tampering, achieved via Hybrid RSA + AES-GCM encryption.
2. **Strict Idempotency:** Simultaneous delivery of the same payment packet by multiple bridge nodes results in exactly one settlement, enforced via atomic compare-and-set locks on the ciphertext hash.
3. **Replay Protection:** Tampered or duplicated packets are cryptographically rejected before reaching the database ledger.

## Architecture & Data Flow

┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND (this project)                  │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] Hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── Atomic putIfAbsent. Duplicates │
│       │                                  rejected immediately.          │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies auth tag — tampering throws exception)      │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within 24h window                        │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking fallback                 │
└─────────────────────────────────────────────────────────────────────────┘

## System Design & Problem Solving

**1. Untrusted Intermediaries**
Random network nodes carry financial transactions. To prevent read or write access, the system uses Hybrid Encryption (RSA-OAEP + AES-GCM). The sender generates a fresh AES-256 key, encrypts the JSON payload using AES-GCM for authenticated encryption, and encrypts the AES key with the server's public RSA-2048 key. If an intermediary flips a single bit, the GCM tag fails verification and the server drops the packet. 

**2. The Duplicate-Storm (Idempotency)**
Multiple bridge nodes may gain internet access simultaneously and POST the same packet. To prevent double-spending, the server computes a SHA-256 hash of the ciphertext and attempts a `ConcurrentHashMap.putIfAbsent` operation (simulating a Redis `SETNX`). Only the first thread successfully claims the lock; concurrent duplicates are short-circuited before consuming CPU cycles on RSA decryption. A unique database index on `transaction.packet_hash` provides a secondary defense layer.

**3. Replay Attacks**
An attacker capturing a ciphertext cannot replay it successfully. The payload includes a `signedAt` epoch timestamp (rejecting packets >24 hours old) and a unique UUID nonce. Because a true replay uses the exact same ciphertext bytes, the idempotency cache traps it. 

## Local Setup & Testing

**Prerequisites:** JDK 17+ 

**Run the Application:**
Linux/macOS: `./mvnw spring-boot:run`
Windows: `mvnw.cmd spring-boot:run`

**Access the Dashboard:**
Navigate to `http://localhost:8080` to access the mesh simulator UI.

**Execute Concurrency Tests:**
Run `./mvnw test` to execute `IdempotencyConcurrencyTest`. This test fires simultaneous threads delivering the same packet to assert that exactly one transaction settles while others are cleanly dropped.

## System Constraints & Future Scope

As a prototype for mesh-routed deferred settlement, this architecture has known constraints designed out of the current scope:
*   **Deferred Settlement:** Because the sender's balance cannot be verified offline, transactions act as cryptographic IOUs. In production, this requires pre-funded hardware-backed wallets (similar to UPI Lite architecture).
*   **Infrastructure Swaps:** For production, the H2 database would be replaced with PostgreSQL, and the JVM-local `ConcurrentHashMap` with a distributed Redis cluster.
*   **Hardware Layer:** The software mesh simulation bypasses the physical complexities of establishing reliable background BLE GATT connections between mobile devices.

---
**Author:** Abhisek Samantray