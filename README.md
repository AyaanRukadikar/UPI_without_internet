# UPI Offline Mesh — Demo

A Spring Boot backend that demonstrates **offline UPI payments routed through a Bluetooth-style mesh network**. Picture yourself in a basement with no signal whatsoever. You want to send your friend ₹500. Your phone encrypts the payment and broadcasts it to nearby devices, where it hops from phone to phone until *someone* steps outside, catches 4G, and quietly uploads it to this backend. The backend then decrypts, deduplicates, and settles the transaction.

This repo contains the **server-side** of that system, along with a software simulator of the mesh — so you can walk through the entire flow on a single laptop without needing any real Bluetooth hardware.

---

## Table of Contents

1. [What this demo proves](#what-this-demo-proves)
2. [How to run it](#how-to-run-it)
3. [The demo flow (step by step)](#the-demo-flow-step-by-step)
4. [Architecture](#architecture)
5. [The three hard problems and how they're solved](#the-three-hard-problems-and-how-theyre-solved)
6. [File-by-file walkthrough](#file-by-file-walkthrough)
7. [API reference](#api-reference)
8. [Tests](#tests)
9. [What's NOT real (and what would change for production)](#whats-not-real-and-what-would-change-for-production)
10. [Honest limitations of the concept](#honest-limitations-of-the-concept)

---

## What this demo proves

The system demonstrates three things working together end to end:

1. **A payment can travel from the sender to the backend through untrusted intermediaries** without any of them being able to read or tamper with it. (Hybrid RSA + AES-GCM encryption.)
2. **Even when the same payment arrives at the backend simultaneously through multiple bridge nodes, it settles exactly once.** (Idempotency via atomic compare-and-set on the ciphertext hash.)
3. **Any tampered or replayed packet gets rejected** before it ever touches the ledger.

You can observe all three behaviours directly in the dashboard.

---

## How to run it

### Prerequisites

- **JDK 17 or newer** installed and available on PATH (or with `JAVA_HOME` configured). Verify with `java -version`.
- That's all. No database, no Redis, no Maven installation needed — the wrapper takes care of that. Just Java.

### Run on Windows

Open a terminal inside the project folder and execute:

```cmd
mvnw.cmd spring-boot:run
```

The first run will download Maven (~10 MB) and all project dependencies (~80 MB) — allow a couple of minutes. Every run after that starts in just a few seconds.

### Run on Mac/Linux

```bash
./mvnw spring-boot:run
```

### Open the dashboard

Once you see `Started UpiMeshApplication in X.XXX seconds`, navigate to:

**http://localhost:8080**

You'll be greeted by a dark-themed dashboard with everything needed to drive the demo.

### Stop the server

Press `Ctrl+C` in the terminal.

### Run the tests

```cmd
mvnw.cmd test
```

The most notable test is `IdempotencyConcurrencyTest` — it spawns three threads delivering the same packet at the same time and asserts that exactly one of them settles.

---

## The demo flow (step by step)

The dashboard provides four buttons that walk you through the complete pipeline. Follow this sequence:

### Step 1 — Compose a payment

Select a sender, receiver, amount, and PIN. Click **"📤 Inject into Mesh"**.

**What actually happens on the backend:**
- The server acts as the sender's phone.
- It constructs a `PaymentInstruction` with a unique nonce and current timestamp.
- It encrypts that instruction using the server's RSA public key (via hybrid encryption — details below).
- It wraps the ciphertext in a `MeshPacket` with a TTL of 5.
- It hands the packet off to `phone-alice`, a virtual offline device.

You'll now see `phone-alice` holding 1 packet.

### Step 2 — Run gossip rounds

Click **"🔄 Run Gossip Round"**. Then click it once more.

Each round, every device carrying a packet broadcasts it to all other devices within "Bluetooth range" (in our simulator, this means everyone). TTL decrements with each hop.

After 1 round, every device holds the packet. After 2 rounds, they still all hold it — the TTL is simply lower.

In a real-world scenario, this spreading would happen naturally as people move past each other in the basement.

### Step 3 — Bridge node walks outside

Click **"📡 Bridges Upload to Backend"**.

`phone-bridge` is the only device with `hasInternet=true`. The dashboard simulates that phone stepping outside and picking up a 4G signal. It then POSTs every packet it holds to `/api/bridge/ingest`.

The backend pipeline executes as follows:
1. Hash the ciphertext (`SHA-256`).
2. Attempt to claim the hash in the idempotency cache.
3. If claimed: decrypt using the server's RSA private key.
4. Validate freshness (ensure `signedAt` is within the last 24 hours).
5. Perform the debit/credit inside a single DB transaction.

Watch the **Account Balances** table — funds have moved. Check the **Transaction Ledger** — a new row has appeared.

### Step 4 — Demonstrate idempotency (the killer feature)

Reset the mesh. Inject a single packet. Run gossip twice. At this point **all 5 devices hold the exact same packet, including multiple bridges in a more complex configuration**.

To really see idempotency in action, you can modify `MeshSimulatorService.java` to seed several bridge devices — or simply:

1. Click "Inject" once.
2. Click "Gossip" twice.
3. Click "Flush Bridges" — since only `phone-bridge` is seeded as a bridge by default, only one upload occurs.

To exercise the *concurrent duplicate* scenario properly, run the test directly:
```cmd
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

This test creates one packet, launches 3 threads against `BridgeIngestionService.ingest()` simultaneously, and confirms that exactly one settles, two are dropped as duplicates, and the sender is debited exactly once.

---

## Architecture

```
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
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies the auth tag — tampering = exception)       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24h                          │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The three hard problems and how they're solved

### Problem 1: Untrusted intermediates

A random stranger's phone is carrying your transaction. How do you prevent them from reading the amount or altering it?

**Solution: Hybrid encryption (RSA-OAEP + AES-GCM).**

The sender encrypts the payload using the server's public key. Since only the server holds the private key, all intermediates see nothing but opaque ciphertext.

However, RSA can only encrypt small amounts of data (~245 bytes for a 2048-bit key), and our payload is JSON that may exceed that limit. So we apply the standard hybrid pattern:

1. Generate a fresh AES-256 key for *this packet*.
2. Encrypt the JSON with **AES-256-GCM** (fast and authenticated).
3. Encrypt only the AES key with **RSA-OAEP**.
4. Concatenate: `[256 bytes RSA-encrypted AES key][12 bytes IV][AES ciphertext + 16-byte GCM tag]`.

**Why GCM specifically?** It provides authenticated encryption. If any intermediate flips even a single bit in the ciphertext, decryption throws an exception — the GCM tag won't match. The server cannot be tricked into processing tampered data.

This is the same scheme that TLS uses. See `HybridCryptoService.java`.

### Problem 2: The duplicate-storm

Three bridge nodes are holding the same packet. They all step outside at the same moment and POST to `/api/bridge/ingest` within milliseconds of each other. If all three are processed naively, the sender gets debited ₹1500 instead of ₹500.

**Solution: Atomic compare-and-set on the ciphertext hash.**

The very first thing the server does upon receiving a packet is compute `SHA-256(ciphertext)` and attempt to "claim" that hash:

```java
// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;  // true = first claimer, false = duplicate
```

`ConcurrentHashMap.putIfAbsent` is atomic. Even if 100 threads call it at the exact same nanosecond, only one returns `null` (the first claimer) while the rest receive the existing entry. Only the first claimer moves on to decrypt and settle. All others are short-circuited as `DUPLICATE_DROPPED`.

**Why hash the ciphertext rather than the packetId or the cleartext?**
- `packetId` can be rewritten by a malicious intermediate. Two copies of the same payment could carry different packetIds — a poor deduplication key.
- The cleartext requires decryption first. We want to deduplicate *before* spending CPU cycles on RSA.
- The ciphertext is authenticated by GCM, so any tampering is detectable at decrypt time. Two legitimate deliveries of the same payment produce byte-identical ciphertexts (AES is deterministic for a given key+IV+plaintext, and the same packet means the same key+IV+plaintext).

In production, this `ConcurrentHashMap` would become Redis: `SET key NX EX 86400`. Same semantics, distributed across replicas.

There's also a defense-in-depth fallback: `transactions.packet_hash` carries a unique index. If the cache layer ever fails and two settlement attempts try to write the same hash, the database rejects the second one outright.

### Problem 3: Replay attacks

An attacker who captured a ciphertext weeks ago could replay it at any convenient time.

**Solution: Two layers.**

1. **Inside the encrypted payload**, the sender embeds `signedAt` (epoch millis). The server rejects any packet older than 24 hours. The attacker cannot alter `signedAt` without breaking the GCM tag.
2. **Inside the encrypted payload**, the sender also includes a **nonce** (UUID). Even if Alice legitimately sends Bob ₹100 on two separate occasions, the nonces differ → the ciphertexts differ → the hashes differ → both settle independently. A *replay* of one specific signed packet, however, is byte-identical, so the idempotency cache catches it.

See `BridgeIngestionService.java` for the freshness check implementation.

---

## File-by-file walkthrough

```
upi-offline-mesh/
├── pom.xml                                  Maven build, Spring Boot 3.3, Java 17
├── mvnw, mvnw.cmd                           Maven wrapper (no install needed)
├── README.md                                this file
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 in-memory DB, port 8080, TTLs
    │   └── templates/dashboard.html         The interactive demo UI
    └── java/com/demo/upimesh/
        ├── UpiMeshApplication.java          Spring Boot main class
        │
        ├── model/                           ── Domain layer
        │   ├── Account.java                 JPA entity. @Version = optimistic lock
        │   ├── AccountRepository.java       Spring Data JPA
        │   ├── Transaction.java             Settled-tx ledger. unique idx on packetHash
        │   ├── TransactionRepository.java   Spring Data JPA
        │   ├── MeshPacket.java              Wire format. Outer fields readable, ciphertext opaque
        │   └── PaymentInstruction.java      Decrypted payload (sender/receiver/amount/nonce/time)
        │
        ├── crypto/                          ── Cryptography layer
        │   ├── ServerKeyHolder.java         Generates RSA-2048 keypair on startup
        │   └── HybridCryptoService.java     RSA-OAEP + AES-256-GCM encrypt/decrypt + ciphertext hash
        │
        ├── service/                         ── Business logic
        │   ├── DemoService.java             Seeds accounts, simulates a sender phone
        │   ├── VirtualDevice.java           One simulated phone in the mesh
        │   ├── MeshSimulatorService.java    Gossip protocol across virtual devices
        │   ├── IdempotencyService.java      ConcurrentHashMap = JVM-local Redis SETNX
        │   ├── SettlementService.java       @Transactional debit + credit + ledger insert
        │   └── BridgeIngestionService.java  THE pipeline: hash → claim → decrypt → freshness → settle
        │
        ├── controller/                      ── HTTP layer
        │   ├── ApiController.java           All REST endpoints
        │   └── DashboardController.java     Serves the dashboard HTML at /
        │
        └── config/
            └── AppConfig.java               @EnableScheduling for cache eviction

src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java          The 3-bridges-at-once test + tamper test
```

---

## API reference

| Method | Path | What it does |
|---|---|---|
| GET | `/` | Dashboard HTML |
| GET | `/api/server-key` | Server's RSA public key (base64) |
| GET | `/api/accounts` | All accounts and their balances |
| GET | `/api/transactions` | Last 20 transactions |
| GET | `/api/mesh/state` | Current state of every virtual device |
| POST | `/api/demo/send` | Simulate sender phone — encrypt + inject packet |
| POST | `/api/mesh/gossip` | Run one round of gossip across the mesh |
| POST | `/api/mesh/flush` | Bridge devices with internet upload to backend (in parallel) |
| POST | `/api/mesh/reset` | Clear the mesh and idempotency cache |
| POST | `/api/bridge/ingest` | **The production endpoint.** Real bridges POST here |
| GET | `/h2-console` | Browse the in-memory database |

H2 console login: JDBC URL `jdbc:h2:mem:upimesh`, username `sa`, no password.

### Request format for `/api/bridge/ingest`

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

Response:
```json
{
  "outcome": "SETTLED",                     // or "DUPLICATE_DROPPED" or "INVALID"
  "packetHash": "a3f8c9...",
  "reason": null,                            // populated on INVALID
  "transactionId": 42                        // populated on SETTLED
}
```

---

## Tests

Run all tests:
```
mvnw.cmd test
```

Three tests are included:

- **`encryptDecryptRoundTrip`** — sanity check confirming that hybrid encryption is symmetric.
- **`tamperedCiphertextIsRejected`** — flips one byte in the ciphertext and verifies that `BridgeIngestionService` returns `INVALID` rather than crashing or settling.
- **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`** — the headline test. Three concurrent threads, one packet, simultaneous delivery. Asserts exactly one `SETTLED`, two `DUPLICATE_DROPPED`, and that the sender's balance changed by the exact amount exactly once.

---

## What's NOT real (and what would change for production)

This is a teaching demo. To bring it up to production grade, you'd replace the following:

| What's in the demo | What it would be in production |
|---|---|
| H2 in-memory DB | PostgreSQL / MySQL with replicas |
| `ConcurrentHashMap` for idempotency | Redis with `SET NX EX` |
| RSA keypair regenerated on every startup | Private key stored in an HSM (AWS KMS, HashiCorp Vault). Public key cached on devices. |
| Server-side `DemoService.createPacket()` | The same logic running on Android, ported to Kotlin |
| Software-simulated mesh (`MeshSimulatorService`) | Real BLE GATT or Wi-Fi Direct between physical phones |
| One settlement service owning the ledger | Integration with NPCI / a real bank core |
| No auth on `/api/bridge/ingest` | Mutual TLS or signed bridge-node certificates |
| In-memory accounts seeded at startup | Real KYC'd users, real VPAs, real PIN verification against the bank |
| H2 console exposed | Disabled entirely |
| No rate limiting | Per-bridge-node rate limits, per-sender velocity checks |
| Logs to console | Structured logs forwarded to a SIEM, with alerts on `INVALID` spikes |

The cryptography and idempotency code is essentially production-grade. It's the infrastructure surrounding it that would need to change.

---

## Honest limitations of the concept

I want this README to be genuinely useful when someone reviews the project, so let's be upfront about what this design **does not** solve. These aren't implementation bugs — they're fundamental to the "no internet anywhere in the chain" model:

1. **The receiver cannot verify that the sender actually has the funds.** When the sender shows their phone saying "₹500 sent," it's effectively an IOU, not a settled payment. If the sender's account is empty by the time the packet reaches the backend, the settlement will be `REJECTED` and the receiver has no recourse. *This is exactly why real offline UPI (UPI Lite) uses a pre-funded hardware-backed wallet* — to provide cryptographic proof of available funds offline.
2. **A malicious sender can double-spend while offline.** With only ₹500 in their account, they could send a packet to Bob in basement A, walk over to basement B, and send another ₹500 to Carol. Whichever packet hits the backend first goes through; the other gets `REJECTED`. This stems from the same root cause as #1.
3. **Bluetooth in the real world is difficult.** Background BLE on Android has been heavily throttled since Android 8. iOS peripheral mode is tightly locked down. Getting two strangers' phones to reliably form a GATT connection while apps aren't in the foreground is genuinely hard — and energy-intensive. This demo sidesteps that problem entirely by simulating the mesh in software.
4. **Privacy and liability concerns.** A stranger's phone carries your encrypted transaction packet. They can't read it, but its mere presence is metadata. A real deployment would need to address regulatory disclosure requirements and what happens if a device is seized.

For a college or portfolio project: frame it honestly as **"mesh-routed deferred settlement"** rather than "real-time offline UPI," and you'll have a much stronger pitch. The cryptography and idempotency work here is genuine engineering, and it's absolutely worth showcasing.

---

## Troubleshooting

**`java: command not found`** — Install JDK 17+. On Windows, run `winget install EclipseAdoptium.Temurin.17.JDK` or download directly from adoptium.net.

**Port 8080 already in use** — Update `server.port` in `application.properties` to a free port.

**First `mvnw.cmd` run hangs for a long time** — It's downloading Maven (~10 MB) followed by all project dependencies (~80 MB). On a normal connection, budget 2–3 minutes. After the initial download, startup takes roughly 5 seconds.

**`mvnw.cmd : The term 'mvnw.cmd' is not recognized`** — On PowerShell you need to prefix with `.\\`: `.\\mvnw.cmd spring-boot:run`.

**Tests fail intermittently** — The concurrency test is timing-sensitive. If it flakes occasionally, run it 3 times; if it consistently fails on your hardware, file a report with the actual failure output.

---

## License

Demo code, no license. Feel free to use it however you like for learning purposes.
