# Ethereum on IC Consensus

<p align="center">
  <img width="800" src="/ethereum-ic-banner.png" alt="Ethereum on IC Consensus">
</p>

> **Certified BFT consensus driving Ethereum block production with threshold-signed state attestations and optimistic pipelining.**

This fork of the [Internet Computer Protocol](https://internetcomputer.org) replaces the Ethereum Beacon Chain with IC's 2-round pBFT consensus, producing threshold-BLS-certified Ethereum state roots suitable for trustless cross-chain communication.

---

## Architecture

```mermaid
graph TD
    subgraph "IC Consensus — 2-round pBFT"
        BM["Block Maker<br/><code>get_payload()</code>"]
        N["Notary — Round 1<br/>Reliable Broadcast + QC"]
        F["Finalizer — Round 2<br/>Agreement QC-on-QC"]
    end

    subgraph "Ethereum Execution"
        GETH["geth (Engine API)<br/>JSON-RPC :8551"]
    end

    subgraph "Certification Layer"
        SM["EthExecutionClient<br/><code>StateManager + StateReader</code>"]
        CP["ExecCertificationPool<br/>Threshold BLS Signing"]
        HTTP["/api/v2/execstatus<br/>HTTP + CBOR"]
    end

    subgraph "External"
        BRIDGE["Cross-Chain Bridges<br/>& Light Clients"]
    end

    BM -- "engine_getPayloadV1" --> GETH
    GETH -- "EthPayload in IC Block" --> BM
    N -- "notarization_hint()" --> SM
    F -- "deliver_batch()" --> SM
    SM -- "engine_newPayloadV1 +<br/>forkchoiceUpdatedV2" --> GETH
    SM -- "list_state_hashes_to_certify()" --> CP
    CP -- "deliver_state_certification()" --> SM
    SM --> HTTP
    HTTP -- "Certified State Root<br/>+ BLS Signature" --> BRIDGE

    style BM fill:#4a90d9,stroke:#2c5282,color:#fff
    style N fill:#4a90d9,stroke:#2c5282,color:#fff
    style F fill:#4a90d9,stroke:#2c5282,color:#fff
    style GETH fill:#3c3c3d,stroke:#1a1a1a,color:#fff
    style SM fill:#38a169,stroke:#276749,color:#fff
    style CP fill:#38a169,stroke:#276749,color:#fff
    style HTTP fill:#38a169,stroke:#276749,color:#fff
    style BRIDGE fill:#d69e2e,stroke:#975a16,color:#fff
```

---

## Optimistic Pipelining

> Blog: [Optimistic EVM Execution with Rotating Leader Consensus](https://medium.com/@farazshaikh/consensus-notes-fixed-tradeoff-between-rotating-leader-and-optimistic-execution-for-ethereum-b7d1fb928e62)

The key latency optimization: block N+1 starts building during block N's second consensus round.

```mermaid
sequenceDiagram
    participant BM as Block Maker
    participant C as IC Consensus
    participant E as EthExecutionClient
    participant G as geth

    Note over C: ── Block N ──
    BM->>G: engine_getPayloadV1
    G-->>BM: ExecutionPayload N
    BM->>C: Propose IC Block (with EthPayload N)

    C->>C: Round 1 — Notarization (QC)
    C->>E: notarization_hint(Block N)
    E->>G: engine_newPayloadV1(N)
    E->>G: forkchoiceUpdatedV2(head=N)

    Note over BM,G: ⚡ Pipeline: Block N+1 starts here
    BM->>G: engine_getPayloadV1
    G-->>BM: ExecutionPayload N+1

    C->>C: Round 2 — Finalization (QC on QC)
    C->>E: deliver_batch(Block N)
    E->>G: forkchoiceUpdatedV2(finalized=N)

    Note over E: If optimistic head ≠ finalized → reorg head to finalized

    BM->>C: Propose IC Block (with EthPayload N+1)
```

Without pipelining, each block waits 2 full consensus rounds. With the notarization hint, validators optimistically assume honest finalization and begin building the next block after round 1 — **cutting effective block time in half**.

Safety: if the optimistic assumption fails, HEAD forks are reverted but FINALIZED state is never compromised. This mirrors Ethereum's own commitment levels (HEAD / SAFE / FINALIZED).

---

## Certified Consensus — Cross-Chain Authentication

> Blog: [Certified Consensus: Cross-Chain Communication in L2/L3 Blockchains](https://medium.com/@farazshaikh/certified-consensus-state-attestation-function-for-cross-chain-communication-68688a08208c)

```mermaid
graph LR
    subgraph "Subnet Validators (N nodes)"
        V1["Validator 1<br/>BLS key share"]
        V2["Validator 2<br/>BLS key share"]
        V3["Validator …<br/>BLS key share"]
    end

    SR["Finalized ETH<br/>State Root"]
    CERT["Threshold BLS<br/>Certificate"]
    PK["Subnet<br/>Public Key"]

    SR --> V1 & V2 & V3
    V1 & V2 & V3 -- "T-of-N signing" --> CERT
    PK -. "verify" .-> CERT

    subgraph "Cross-Chain Consumer"
        VER["Verify certificate<br/>against subnet pubkey"]
        PROOF["EVM Merkle<br/>State Proof"]
    end

    CERT --> VER
    VER --> PROOF

    style SR fill:#3c3c3d,stroke:#1a1a1a,color:#fff
    style CERT fill:#38a169,stroke:#276749,color:#fff
    style PK fill:#4a90d9,stroke:#2c5282,color:#fff
    style VER fill:#d69e2e,stroke:#975a16,color:#fff
    style PROOF fill:#d69e2e,stroke:#975a16,color:#fff
```

Two building blocks enable cross-chain communication:

1. **Authentication** — Threshold BLS signatures give the subnet a persistent public key. Any finalized state root signed by T-of-N validators is verifiable by anyone holding the subnet's public key.
2. **Vector Commitment** — The EVM Merkle state tree is the de-facto standard. Once the state root is authenticated, standard Merkle proofs attest to any account balance, storage slot, or contract state.

Together: **Certified State Root + Merkle Proof = Trustless Cross-Chain Read**.

---

## Time-Locked Encryption — MEV Protection

> Blog: [Time Encrypted Mempool](https://medium.com/@farazshaikh/timelock-encrypted-and-encrypted-mempools-962ebd901faa)

MEV (Miner Extractable Value) lets block builders front-run, sandwich, or reorder transactions for profit. The solution is **pre-trade privacy**: encrypt transactions so their intent is hidden until execution order is already committed.

```mermaid
sequenceDiagram
    participant U as User / Wallet
    participant M as Encrypted Mempool
    participant BB as Block Builder
    participant C as Consensus
    participant D as Decryption

    Note over U,D: Phase 1 — Submission (intent is hidden)
    U->>U: Timelock-encrypt transaction
    U->>M: Submit encrypted tx

    Note over U,D: Phase 2 — Ordering (blind commitment)
    M->>BB: Batch of encrypted txs
    BB->>BB: Determine execution order<br/>(no knowledge of tx content)
    BB->>C: Propose ordered block<br/>of encrypted txs
    C->>C: Consensus commits to order

    Note over U,D: Phase 3 — Reveal & Execute
    D->>D: Time passes → txs auto-decrypt
    D->>BB: Plaintext txs in committed order
    BB->>BB: Execute in committed order

    Note over U,D: Any deviation from committed order<br/>is attributable to a specific malicious party
```

**How it works:**

1. **Encrypt** — Clients timelock-encrypt transactions before submission. The ciphertext is guaranteed to become decryptable after a fixed time window, with no trusted party needed.
2. **Commit** — The block builder orders encrypted transactions and publicly commits to a sequence. Since transaction content is hidden, front-running, sandwich attacks, and other MEV strategies have no information to exploit.
3. **Reveal & Execute** — After the time window, transactions decrypt and execute in the committed order. Any deviation from the published order is cryptographically attributable.

This eliminates the information asymmetry that makes MEV possible. The approach is not crypto-specific — traditional finance suffers the same problem (e.g., Citadel Securities was fined $700K for front-running client orders).

### BLS as Identity-Based Encryption — Dual Use of the Threshold Key

The same threshold BLS key infrastructure used for state root certification doubles as a **timelock encryption** scheme via the [Boneh-Franklin IBE construction](https://crypto.stanford.edu/~dabo/papers/bfibe.pdf).

```mermaid
graph TD
    subgraph "Shared Infrastructure"
        MPK["Subnet Master<br/>Public Key<br/>(BLS public key)"]
        SK["Threshold Secret Key<br/>(shared across N validators)"]
    end

    subgraph "Certification (Signing)"
        SR["State Root"]
        SIG["Threshold BLS<br/>Signature = Certificate"]
        SR --> SIG
        SK -. "T-of-N sign" .-> SIG
        MPK -. "verify" .-> SIG
    end

    subgraph "Timelock Encryption (IBE)"
        ID["Identity = future<br/>block height h"]
        CT["Ciphertext<br/>(encrypted tx)"]
        DK["Decryption Key =<br/>BLS signature on h"]
        PT["Plaintext tx"]

        MPK -- "IBE.Encrypt(id=h, msg)" --> CT
        ID --> CT
        SK -. "T-of-N sign h<br/>(happens naturally<br/>during consensus)" .-> DK
        CT -- "IBE.Decrypt(dk)" --> PT
        DK --> PT
    end

    style MPK fill:#4a90d9,stroke:#2c5282,color:#fff
    style SK fill:#4a90d9,stroke:#2c5282,color:#fff
    style SIG fill:#38a169,stroke:#276749,color:#fff
    style CT fill:#d69e2e,stroke:#975a16,color:#fff
    style DK fill:#38a169,stroke:#276749,color:#fff
    style PT fill:#3c3c3d,stroke:#1a1a1a,color:#fff
```

Under Boneh-Franklin, a BLS key pair is also an IBE key pair:

- **Master public key** = the subnet's BLS public key (already published for certificate verification).
- **Identity** = an arbitrary string — here, a future block height `h`.
- **Encrypt** = anyone can encrypt a transaction to identity `h` using only the master public key. No interaction with validators needed.
- **Decrypt** = the decryption key for identity `h` is the BLS signature on `h`. During normal consensus at height `h`, validators collectively produce exactly this signature as part of block certification. The decryption key materializes as a **byproduct of consensus** — no extra protocol round required.

This means the threshold BLS signing that already certifies state roots *also* provides the timelock decryption keys for the encrypted mempool, giving both features from a single cryptographic setup.

**Status:** Design-level proposal. The consensus and certification layers in this repo provide the substrate on which an encrypted mempool can be built.

---

## Commit History

| # | Commit | Tag | Summary |
|---|--------|-----|---------|
| 1 | `33647a7` | `FEAT-KVM` | Boot IC subnets on local KVM VMs — `icos_deploy_local.sh`, `bobcat`/`frz13` environments |
| 2 | `c4b1d64` | `FIX-DFX-REPL` | Artifact pool fix to hot-swap dfx replica with local Bazel/Cargo build |
| 3 | `2873987` | `FEAT-ETH-PART` | Guest OS partition layout: add encrypted Ethereum data partition for geth |
| 4 | `9215331` | `FEAT-GETH` | Integrate go-ethereum into the OS image — systemd service, genesis bootstrap |
| 5 | `d3a0b63` | `FEAT-EXEC-API` | Add Ethereum Engine API client library as Bazel dependency |
| 6 | `7002486` | `FEAT-ETH-INT` | Core consensus/execution integration — `EthPayloadBuilder`, `EthMessageRouting`, payload embedding |
| 7 | `4842eef` | `FEAT-ETH-CERT` | Threshold BLS certification of Ethereum state roots — `ExecCertificationPool`, `StateManager` impl |
| 8 | `c0a4eb2` | `FEAT-HTTP-CERT` | HTTP endpoint `/api/v2/execstatus` serving certified state roots as CBOR |
| 9 | `1b1cfa8` | `FEAT-OPTIMISTIC` | Optimistic block building via notarization hints — pipeline cuts latency in half |

**83 files changed, +2,634 / -127 lines** from base `a404154` (upstream IC).

---

## Key Components

### `rs/consensus/src/consensus/eth.rs`

The central integration point. `EthExecutionClient` wraps geth's Engine API and implements:

- **`EthPayloadBuilder`** — called by the block maker to fetch an Ethereum execution payload from geth and embed it in IC consensus blocks.
- **`EthMessageRouting`** — called by the finalizer (`deliver_batch`) and notary (`notarization_hint`) to push agreed-upon blocks back to geth.
- **`StateManager`** — queues finalized state roots for threshold BLS certification and delivers completed certificates.
- **`StateReader`** — serves the latest certified state root to the HTTP layer.

### `rs/artifact_pool/src/exec_certification_pool.rs`

A parallel certification pool (`ExecCertificationPoolImpl`) backed by LMDB. Mirrors the existing IC certification pool but tracks Ethereum state root attestations under a separate `ExecCertificationArtifact` kind, keeping P2P message multiplexing clean.

### `rs/http_endpoints/public/src/execstatus.rs`

Tower HTTP service at `/api/v2/execstatus`. Returns CBOR-encoded responses containing the latest certified height, threshold BLS certificate, and replica metadata — the external interface for cross-chain bridges and light clients.

### `rs/types/types/src/eth.rs`

Core data types:
- `EthPayload` — serialized execution payload + timestamp, embedded in IC block proposals.
- `EthExecutionDelivery` — consensus height + payload, passed to geth post-finalization.

---

## Blog Posts

These articles describe the theory behind this implementation:

- [**Certified Consensus: Cross-Chain Communication in L2/L3 Blockchains**](https://medium.com/@farazshaikh/certified-consensus-state-attestation-function-for-cross-chain-communication-68688a08208c) — Authentication via threshold signatures + EVM state trees as vector commitments for trustless cross-chain reads.

- [**Optimistic EVM Execution with Rotating Leader Consensus**](https://medium.com/@farazshaikh/consensus-notes-fixed-tradeoff-between-rotating-leader-and-optimistic-execution-for-ethereum-b7d1fb928e62) — Pipelining block production across consensus rounds using notarization hints, halving effective block time.

- [**Time Encrypted Mempool**](https://medium.com/@farazshaikh/timelock-encrypted-and-encrypted-mempools-962ebd901faa) — Pre-trade privacy via timelock encryption to eliminate MEV — a design-level proposal for future work.

---

## Quick Start

```bash
# Build the replica with Ethereum support
bazel build --compilation_mode=dbg //rs/replica:replica

# Deploy a local 2-subnet topology on KVM
./testnet/tools/icos_deploy_local.sh
```

---

## License

See [LICENSE](LICENSE).
