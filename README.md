# Misaka Network

**Post-Quantum Native Privacy Blockchain with Lightweight Pruned Nodes**

> Sender anonymity · Receiver anonymity · Hidden amounts · Quantum-resistant · 3-min finality

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Summary](#architecture-summary)
3. [Governance System](#governance-system)
   - 3.1 [Participants & Roles](#31-participants--roles)
   - 3.2 [Proposal Lifecycle](#32-proposal-lifecycle)
   - 3.3 [Governance Transaction Structure](#33-governance-transaction-structure)
   - 3.4 [Voting Weight & Deadlock Fallback](#34-voting-weight--deadlock-fallback)
4. [Economic Model (Tokenomics)](#economic-model-tokenomics)
   - 4.1 [Fee Structure](#41-fee-structure)
   - 4.2 [Reward Distribution](#42-reward-distribution)
   - 4.3 [Slashing Policy](#43-slashing-policy)
5. [Archive Node Trust Model](#archive-node-trust-model)
6. [Post-Quantum Cryptography Parameters](#post-quantum-cryptography-parameters)
   - 6.1 [Falcon-512](#61-falcon-512)
   - 6.2 [Kyber-768](#62-kyber-768)
   - 6.3 [LaRRS Ring Signature Parameters](#63-larrs-ring-signature-parameters)
   - 6.4 [Deterministic Randomness (DRBG)](#64-deterministic-randomness-drbg)
7. [Node Types](#node-types)
8. [Transaction Format](#transaction-format)
9. [Running a Node](#running-a-node)

---

## Overview

Misaka Network is a Layer-1 blockchain designed around three principles:

- **Privacy by default** — every transaction hides sender, receiver, and amount using post-quantum lattice primitives
- **Lightweight participation** — validators and light nodes operate without storing the full transaction history; only operator-managed Archive Nodes hold the complete chain
- **Quantum resistance** — no elliptic-curve cryptography anywhere in the stack; Falcon-512, Kyber-768, and SHA3-256 throughout

| Parameter | Value |
|-----------|-------|
| Block time | 180 s (3 minutes) |
| Block size target | 4 MB |
| Transaction size | 50 – 200 KB |
| Target throughput | 0.2 – 0.3 TPS |
| Consensus | Tendermint-style BFT, 4–16 validators |
| Signature | Falcon-512 (NIST PQC Standard) |
| Ring signature | LaRRS (Module-LWE based) |
| Key exchange | Kyber-768 (CRYSTALS-Kyber) |
| Hash | SHA3-256 |
| Security level | 128-bit post-quantum |

---

## Architecture Summary

```
┌──────────────────────────────────────────────────────────┐
│  Application Layer                                        │
│  Wallet SDK · Jamtis address derivation · TX builder      │
├──────────────────────────────────────────────────────────┤
│  Privacy Layer                                            │
│  Seraphis TX model · LaRRS ring sigs · PQ commitments     │
│  Stealth addresses · Confidential amounts                 │
├──────────────────────────────────────────────────────────┤
│  Consensus Layer                                          │
│  Tendermint BFT · Falcon-512 votes · Epoch governance     │
├──────────────────────────────────────────────────────────┤
│  Network Layer                                            │
│  libp2p gossip · Noise XX (Kyber-768) · Kademlia DHT      │
├──────────────────────────────────────────────────────────┤
│  Storage Layer                                            │
│  UTXO set (all nodes) · Pruned blocks (validators)        │
│  Full history (Archive nodes only)                        │
└──────────────────────────────────────────────────────────┘
```

---

## Governance System

Misaka adopts a **Cardano-inspired "separation of powers"** model, mapping the three-branch structure onto Misaka's own node roles. All governance votes use Falcon-512 signatures, making the governance layer itself quantum-resistant.

### 3.1 Participants & Roles

| Role | Misaka Equivalent | Governance Authority |
|------|-------------------|----------------------|
| **Proposer** | Any network participant who locks a deposit | Submits new validator nominations, parameter changes, or constitutional amendments |
| **Validator (SPO equivalent)** | Active validator nodes | Technical feasibility judgment; ≥ ⌊2N/3⌋ + 1 approval required |
| **Archive Governance (CC equivalent)** | Foundation / initial operator nodes | Monitors compliance with the network constitution; veto on proposals that violate base policy |

> The three roles are intentionally independent. A validator cannot simultaneously act as Archive Governance for the same proposal, preventing collusion.

---

### 3.2 Proposal Lifecycle

The lifecycle mirrors Cardano's Governance Action flow, adapted for Misaka's epoch-based block time.

#### ① Proposal Submission

Anyone operating a network node (including Archive nodes) can submit a proposal by broadcasting a `GovernanceAction` transaction:

```
GovernanceAction TX
├── action_type       : u8     // e.g. 0x01 = Add Validator
├── deposit           : u64    // MSK tokens locked (spam prevention)
├── anchor_url        : Vec<u8> // off-chain rationale document URL
└── anchor_hash       : [u8;32] // SHA3-256 of the document content
```

- The **deposit** is returned if the proposal passes ratification, or forfeited if it is rejected within the voting period.
- The **anchor** binds the on-chain proposal to a human-readable rationale, ensuring full transparency of intent—visible to all participants.

#### ② Voting Period (Epoch-based)

Voting windows align with **epochs** (default: 100 blocks ≈ 5 hours). Immediate on-chain application does not occur during the voting period.

Active validators cast votes using their Falcon-512 consensus key:

```rust
enum Vote { Yes, No, Abstain }

struct GovernanceVote {
    proposal_id : [u8; 32],
    validator_id: [u8; 32],   // Falcon-512 pubkey hash
    vote        : Vote,
    weight      : u64,        // staked MSK at time of vote snapshot
    signature   : [u8; 1281], // Falcon-512 signature
}
```

#### ③ Ratification: ⌊2N/3⌋ + 1 Threshold

At epoch boundary, ratification passes when **both** of the following hold:

1. **Validator quorum**: weighted YES votes ≥ ⌊2N/3⌋ + 1 of total stake-weighted validator votes
2. **Archive Governance clearance**: ≥ majority of Archive Governance nodes have not raised a constitutional objection

The double-check prevents a supermajority of validators from forcing through changes that violate the network's foundational constitution (analogous to Cardano's Constitutional Committee veto).

#### ④ Enactment

Ratified proposals take effect at the **next epoch boundary**, ensuring all nodes can prepare the state transition without a mid-epoch inconsistency.

```
Epoch N       Voting period closes
Epoch N+1     Ratification check executes
Epoch N+2     Enactment: new validator_id added to BFT set
```

The new validator's Falcon-512 public key hash (`validator_id`) is appended to the on-chain validator set, and the node begins participating in consensus from the first block of Epoch N+2.

---

### 3.3 Governance Transaction Structure

```rust
struct GovernanceProposal {
    proposal_id          : [u8; 32],  // SHA3-256(tx_id || output_index)
    action_type          : u8,
    // 0x01 = Add Validator
    // 0x02 = Remove Validator
    // 0x03 = Parameter Change (block size, fee floor, epoch length …)
    // 0x04 = Constitutional Amendment (requires Archive Governance approval)
    target_validator_id  : [u8; 32],  // Falcon-512 pubkey hash of candidate
    deposit              : u64,       // locked MSK tokens
    anchor_url           : Vec<u8>,   // off-chain rationale URL
    anchor_hash          : [u8; 32],  // SHA3-256 of rationale document
    voting_epoch_start   : u64,       // epoch when voting opens
    voting_epoch_end     : u64,       // epoch when voting closes
}
```

`action_type 0x04` (Constitutional Amendment) requires **Archive Governance super-majority** (≥ 3/4) in addition to the standard validator quorum, protecting foundational rules from simple majority capture.

---

### 3.4 Voting Weight & Deadlock Fallback

**Stake-weighted voting**: each validator's vote is weighted by their staked MSK balance at the snapshot taken at `voting_epoch_start`. This aligns economic incentives with governance outcomes.

```
effective_weight(validator_i) = staked_MSK_i / total_staked_MSK
weighted_yes = Σ effective_weight(i) for all i where vote_i == Yes
```

Ratification requires `weighted_yes ≥ 2/3`.

**Deadlock resolution**: if a proposal fails to reach quorum by `voting_epoch_end`, it expires and the deposit is refunded. The proposer may re-submit with a revised anchor. There is no automatic escalation path — this is intentional, favouring network stability over forced progress.

---

## Economic Model (Tokenomics)

### 4.1 Fee Structure

PQ signatures are large (Falcon-512: ~1.3 KB; LaRRS ring proofs: ~12–24 KB per input). The fee model must reflect byte costs without making privacy prohibitively expensive.

**Fee formula**:

```
tx_fee = max(FEE_FLOOR, tx_size_bytes × FEE_PER_BYTE)

FEE_FLOOR    = 0.001 MSK   (minimum fee regardless of size)
FEE_PER_BYTE = 0.000002 MSK/byte
```

For a typical 100 KB transaction:

```
fee = max(0.001, 100_000 × 0.000002) = max(0.001, 0.200) = 0.200 MSK
```

Both `FEE_FLOOR` and `FEE_PER_BYTE` are governance parameters (action_type `0x03`) and can be adjusted by validator vote.

**Fee as percentage of transaction value** (for value-based transactions):

An additional floor of **2% of the transaction output value** applies, replacing the byte-based fee if higher:

```
fee = max(byte_based_fee, 0.02 × output_value)
```

This 2% total fee is then split among three reward pools as described below.

---

### 4.2 Reward Distribution

Every block's collected fees are redistributed via a `DistributedCoinbase` transaction:

| Recipient | Share | Rationale |
|-----------|-------|-----------|
| **Pruned (Validator) Nodes** | 1.5% of tx value (75% of fee) | Block production, BFT consensus, ring sig verification |
| **Archive Nodes** | 0.5% of tx value (25% of fee) | Full history storage, bootstrapping, explorer service |
| **Admin / Foundation** | Base fee overhead + rounding dust | Protocol maintenance, governance infrastructure |

Within the **Validator** pool, rewards are distributed proportional to each validator's **staked MSK**:

```
validator_i_reward = pool_amount × (staked_MSK_i / total_staked_MSK)
```

The block proposer receives no additional bonus beyond their stake-proportional share, eliminating incentives for selfish mining.

No new tokens are created via block rewards — the supply is strictly fee-recycled. Total supply is fixed at genesis.

---

### 4.3 Slashing Policy

Misaka adopts **Cardano's non-confiscatory penalty model**: there is no stake slashing (confiscation of locked tokens). Instead, misbehaving validators face **reward reduction and removal**:

| Offence | Penalty |
|---------|---------|
| **Equivocation** (double-sign same height) | Immediate removal from validator set via automatic governance action; forfeit pending rewards for current epoch |
| **Liveness failure** (absent > 100 consecutive rounds) | Reward multiplier reduced to 0 for the missed rounds; governance flag raised after 200 rounds |
| **Invalid block proposal** | Reward withheld for that round; repeated offences trigger governance removal vote |

The rationale for non-confiscation: in a small validator set (4–16 nodes), stake slashing creates extreme economic fragility from bugs or network partitions. Reward reduction achieves the same deterrent effect without risking network collapse.

Slashing records are stored on-chain as `SlashRecord` entries and are permanent, providing reputational accountability even without token confiscation.

---

## Archive Node Trust Model

### Decentralisation Requirements

Archive nodes are operator-managed (foundation, exchanges, public-good operators), but the protocol enforces the following to prevent centralisation:

- **Minimum viable archive set**: governance parameters enforce ≥ 3 independent archive operators for mainnet
- **Bootstrap diversity**: new nodes must accept UTXO snapshots from ≥ 2 independent archive sources and verify consistency before trusting either
- **Censorship resistance**: if a new node cannot reach an archive bootstrap within 60 s, it falls back to syncing block headers from validators (slower but uncensorable)

### Snapshot Integrity Verification

New nodes receive UTXO snapshots compressed with zstd. The snapshot's authenticity is verified without downloading full history:

1. Archive node provides: `{utxo_snapshot, snapshot_height, merkle_proof}`
2. New node fetches the block header at `snapshot_height` from ≥ 2 validators (independently)
3. Recompute `SHA3-256(utxo_snapshot)` and verify it matches `header.utxo_root`
4. Verify the block header's BFT certificate (≥ ⌊2N/3⌋ + 1 Falcon-512 validator signatures)

This provides **cryptographic binding** between the snapshot and the BFT-finalised chain state without ZKP, since the BFT certificate itself serves as the trust anchor. Future versions may add a ZK succinct proof of UTXO set derivation for stronger guarantees against archive equivocation.

---

## Post-Quantum Cryptography Parameters

### 6.1 Falcon-512

Used for: validator block signatures, BFT votes, governance votes, wallet spend authorisation.

| Parameter | Value |
|-----------|-------|
| Lattice dimension `n` | 512 |
| Modulus `q` | 12289 |
| NTRU lattice | Yes (power-of-2 cyclotomic ring) |
| Signature size | ~666 bytes (compressed) / 1281 bytes (padded) |
| Public key size | 897 bytes |
| Security level | 128-bit post-quantum (NIST Level I) |
| Signing Gaussian width `σ` | ≈ 165 |
| Hash-to-point | SHAKE256 |

Falcon-512 is a NIST PQC standardised signature scheme (FIPS 206 draft). It provides compact signatures relative to other lattice schemes, making it suitable for the high-frequency BFT message passing in Misaka's consensus layer.

### 6.2 Kyber-768

Used for: P2P Noise XX handshake, wallet stealth address key encapsulation (Jamtis KEM step), memo encryption.

| Parameter | Value |
|-----------|-------|
| Module rank `k` | 3 |
| Polynomial degree `n` | 256 |
| Modulus `q` | 3329 |
| Noise distribution `η₁` / `η₂` | 2 / 2 |
| Ciphertext size | 1088 bytes |
| Public key size | 1184 bytes |
| Shared secret size | 32 bytes |
| Security level | 180-bit classical / 164-bit post-quantum (NIST Level III) |

CRYSTALS-Kyber is a NIST PQC KEM standard (FIPS 203). Level III provides a comfortable security margin above the 128-bit minimum, absorbing future cryptanalytic advances against Module-LWE.

### 6.3 LaRRS Ring Signature Parameters

LaRRS (Lattice Ring Signature Scheme) is built on Module-LWE and provides post-quantum ring signatures for Misaka's transaction anonymity layer.

| Parameter | Symbol | Value | Notes |
|-----------|--------|-------|-------|
| Module rank | `k` | 3 | Same module structure as Kyber |
| Polynomial degree | `n` | 256 | NTT-friendly |
| Modulus | `q` | 8380417 | = 2²³ − 2¹³ + 1 (prime, NTT-friendly) |
| L2 norm bound | `β` | 2¹⁸ | Rejection sampling bound on response vectors |
| Rejection rate | — | ~26% per round | Gaussian tail probability |
| Challenge space | `c` | 256-bit challenge via SHA3-256 | Fiat-Shamir transform |
| Ring size | `N` | 11–16 UTXOs | Default 16 for maximum anonymity |
| Proof size (N=16) | — | ~11–14 KB per input | Per-ring-member response vector |
| Security level | — | 128-bit post-quantum | Under Module-LWE hardness |
| Security assumption | — | MLWE, SIS | Reduction to Module Short Integer Solution |

**Link tag construction**:

```
link_tag = SHA3-256(H_tag(sk_spend) || tx_nonce)
```

where `H_tag` is a domain-separated hash-to-lattice function. The link tag is deterministic for a given spend key and unique per transaction, enabling double-spend detection without revealing key identity.

**Proof size trade-off**:

Increasing ring size from 11 to 16 adds ~4 KB per proof but raises the anonymity set by 45%. For Misaka's 50–200 KB transaction budget, ring size 16 is the recommended default.

### 6.4 Deterministic Randomness (DRBG)

All signing operations in Misaka use a **Hash-DRBG (SP 800-90A Rev.1)** seeded from a post-quantum safe source:

```
seed_material = OS_entropy (256 bits, /dev/urandom or platform equivalent)
               || application_context_string
               || timestamp_nonce

DRBG = SHAKE256-DRBG(seed=seed_material, security_strength=256)
```

**Why SHAKE256-DRBG over CTR-DRBG (AES)**:

CTR-DRBG relies on AES, which is vulnerable to Grover's algorithm (effective 128-bit key security halved to 64-bit). SHAKE256-DRBG has no algebraic structure exploitable by quantum algorithms; its security reduces to the collision resistance of SHA3, which loses only a square-root factor under Grover — still ≥ 128-bit post-quantum at 256-bit output width.

**Deterministic key derivation** (wallet):

```
master_seed  ← BIP-39 mnemonic → entropy (256 bits)
spend_key    ← HKDF-SHA3-256(master_seed, info="misaka_spend_v1", len=64)
view_key     ← HKDF-SHA3-256(master_seed, info="misaka_view_v1",  len=64)
addr_key     ← HKDF-SHA3-256(view_key,    info="misaka_addr_v1",  len=32)
```

HKDF with SHA3-256 ensures that compromising one derived key does not reveal the master seed (one-way HKDF extraction), and that all wallet keys are reproducible from the mnemonic alone.

**Ephemeral signing nonces** (Falcon-512):

Falcon's trapdoor sampler internally uses a tree-based Gaussian sampler (NTRU-solve). The randomness for each signing operation is drawn fresh from the SHAKE256-DRBG instance, never reused. Nonce reuse in Falcon is catastrophic (reveals the secret key), so the DRBG state is advanced and persisted atomically before each signing call.

---

## Node Types

| | Light Node | Validator Node | Archive Node |
|--|-----------|---------------|-------------|
| Purpose | Wallet / user | Block production | Full history |
| RAM | 128–512 MB | 4 GB | 16+ GB |
| Storage | 50–200 MB | 50–200 GB SSD | 2+ TB NVMe |
| Block headers | ✓ All | ✓ All | ✓ All |
| UTXO set | Owned outputs only | Full | Full |
| Transaction proofs | ✗ | Recent (pruned) | ✓ All |
| Ring proofs | ✗ | Verifies, then prunes | ✓ All |
| BFT consensus | ✗ | ✓ | Observer only |
| Governance voting | ✗ | ✓ (stake-weighted) | ✓ (CC role) |

---

## Transaction Format

```rust
struct Transaction {
    version     : u8,
    inputs      : Vec<TxInput>,       // each carries a ring + link tag
    outputs     : Vec<TxOutput>,      // stealth addresses + commitments
    ring_proof  : PQRingProof,        // LaRRS proof
    conf_proof  : BalanceProof,       // lattice Pedersen balance + range proofs
    link_tags   : Vec<[u8; 32]>,      // double-spend prevention
    fee_commit  : [u8; 32],           // commitment to explicit fee
    tx_extra    : Vec<u8>,            // encrypted memo, ephemeral KEM ciphertext
}

struct TxInput {
    ring        : Vec<[u8; 32]>,      // N UTXO commitments (decoys + real)
    link_tag    : [u8; 32],
    enote_image : [u8; 32],
}

struct TxOutput {
    stealth_addr: JamtisStealthAddr,
    commitment  : [u8; 32],           // lattice Pedersen commitment
    enc_amount  : [u8; 48],           // ChaCha20-Poly1305 encrypted amount
    view_tag    : u8,                  // 1-byte scanning optimisation
}
```

---

## Running a Node

```bash
# Clone and install
git clone https://github.com/misaka-network/misaka-node
cd misaka-node
npm install

# Configure node type
cp config/example.json config/node.json
# Set: nodeType = "validator" | "light" | "archive"
#      chainId, listenPort, bootstrapPeers, dataDir

# Generate Falcon-512 validator key
npm run keygen -- --output ./keys/validator.key

# Start
npm start -- --config config/node.json
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MISAKA_CHAIN_ID` | `misaka-mainnet-1` | Chain identifier |
| `MISAKA_DATA_DIR` | `./data` | Blockchain data directory |
| `MISAKA_LISTEN_PORT` | `28080` | P2P gossip port |
| `MISAKA_RPC_PORT` | `28081` | REST/RPC API port |
| `MISAKA_LOG_LEVEL` | `info` | Log verbosity |
| `MISAKA_PRUNE_HORIZON` | `1000` | Blocks to retain before pruning |

---

## Security Assumptions

All cryptographic components reduce to the following well-studied hard problems:

| Primitive | Hard Problem |
|-----------|-------------|
| Falcon-512 | NTRU / Ring-SIS over power-of-2 cyclotomic rings |
| Kyber-768 | Module-LWE (MLWE) |
| LaRRS ring signature | MLWE + Module-SIS |
| SHAKE256-DRBG | Collision resistance of SHA3 |
| Lattice Pedersen commitments | SIS (Short Integer Solution) |

No component relies on the discrete logarithm problem or integer factorisation, providing full resistance to Shor's algorithm.

---

*Misaka Network · Technical Reference · v1.0*
