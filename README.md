# FIDO2 Passkey Recovery Protocol — Tamarin Formal Verification

A formal security analysis of a **peer-assisted FIDO2 passkey recovery protocol** using the [Tamarin Prover](https://tamarin-prover.github.io/). The repository contains two protocol variants (baseline and hardened), their Tamarin model files, generated dependency graphs, a custom proof oracle, and a DOT graph post-processor for visualisation.

---

## Overview

FIDO2/WebAuthn passkeys are bound to a device's hardware, which creates a usability problem: if a user loses their device, they lose access to all passkey-protected accounts. This project models and formally verifies a **collaborative recovery scheme** in which a trusted peer (e.g. a colleague) assists the recovery process by holding one cryptographic share of the user's private key.

The private key `ltk` is split into two shares using XOR:

```
sk1  ← random fresh value
sk2  = ltk ⊕ sk1
```

`sk1` is encrypted under the peer's public key and stored (via the server) with the peer. `sk2` and the encrypted FIDO key are stored by the server. Recovery requires both shares — neither the server nor the peer alone can reconstruct `ltk`.

---

## Repository Structure

```
.
├── 1_ICS_FIDO2PasskeyRecovery_Protocol.spthy           # Baseline — Insecure Channel Scheme (ICS)
├── 1_ICS_FIDO2PasskeyRecovery_Protocol_dependencyGraph.spthy   # Tamarin proof output (ICS)
├── 1_ECS_FIDO2PasskeyRecovery_Protocol.spthy           # Baseline — Extended/Encrypted Channel Scheme (ECS)
├── 1_ECS_FIDO2PasskeyRecovery_Protocol_dependencyGraph.spthy   # Tamarin proof output (ECS)
├── 2_ECS_FIDO2PasskeyRecovery_HardenedProtocol.spthy           # Hardened ECS Protocol
├── 2_ECS_FIDO2PasskeyRecovery_HardenedProtocol_dependencyGraph.spthy  # Tamarin proof output (Hardened)
├── FIDO2_Passkey_Recovery.oracle                        # Custom Tamarin proof-search oracle (Python 3)
├── tamarin-cleandot.py                                  # Entry-point for DOT graph post-processor
├── tamarincleandotlib.py                                # Core library for DOT graph post-processing
└── commandsForTerminal.txt                              # Ready-to-run Tamarin commands
```

---

## Protocol Variants

### 1. ICS — Insecure Channel Scheme (`1_ICS_...`)
Models the scenario where the **server-to-client (S2C) channel is not authenticated**. Messages from the server arrive via the public Dolev-Yao network (`In`/`Out` facts), meaning an active adversary can intercept and replay them. This represents a weaker deployment model, used as a baseline to identify channel-dependency of security properties.

### 2. ECS — Extended Channel Scheme (`1_ECS_...`)
All three channel directions are secure:
- **C2S**: Client → Server (secure)
- **S2C**: Server → Client (secure, unlike ICS)
- **P2P**: Peer-to-Peer (secure, QR code delivery)

This is the primary baseline model where the server-to-client link is protected.

### 3. Hardened ECS (`2_ECS_...`)
Builds on ECS with several improvements that address additional attack surfaces:

| Feature | Baseline ECS | Hardened ECS |
|---|---|---|
| Key pairs per user | 1 (shared `ltk`) | 2 (separate `ltk_user` and `ltk_peer`) |
| Nonce binding in QR code | ✗ | ✓ (`qr_code(<sk1, nonce>)`) |
| FIDO blob integrity check | ✗ | ✓ (ltk embedded in encrypted FIDO blob) |
| Reveal rules | 1 (`Reveal_Ltk`) | 2 (`Reveal_Ltk_User`, `Reveal_Ltk_Peer`) |
| Key isolation lemmas | ✗ | ✓ (own-peer-key and guardian-user-key) |
| Consistency lemma | ✗ | ✓ (`consistency_fido_recovery`) |

The **dual key pair design** ensures that a compromise of the user's recovery-role key (peer key) does not expose their authentication key (user key), and vice versa.

---

## Security Properties (Lemmas)

All three models verify the following core lemmas:

| Lemma | Description |
|---|---|
| `protocol_is_executable_strong` | Sanity / executability — at least one complete recovery trace exists without any key compromise |
| `secrecy_peer_share_sk1` | The peer's share `sk1` is never known to the adversary when no long-term keys are revealed |
| `secrecy_main_key` | The reconstructed private key `ltk` remains secret after recovery |
| `secrecy_fido_key` | The recovered FIDO key remains secret after recovery |

The **Hardened ECS** additionally verifies:

| Lemma | Description |
|---|---|
| `isolation_own_peer_key_compromise` | Recovering user's own peer key being leaked does **not** compromise the FIDO key |
| `isolation_guardian_user_key_compromise` | Guardian (peer) user key being leaked does **not** compromise `sk1` |
| `consistency_fido_recovery` | The FIDO key recovered is exactly the one that was originally generated |

---

## Running the Proofs

### Prerequisites

- [Tamarin Prover](https://tamarin-prover.github.io/) (tested with `--derivcheck-timeout=0`)
- Python 3 (for the oracle and DOT post-processor)
- `graphviz` (`dot` binary) for dependency graph rendering
- Python packages: `pydot`, `pyparsing`

### Commands

Run each model with the oracle and graph output enabled. Commands are provided in `commandsForTerminal.txt`:

```bash
# ICS Baseline
/usr/bin/time -v tamarin-prover --with-dot=./tamarin-cleandot.py \
  1_ICS_FIDO2PasskeyRecovery_Protocol.spthy \
  --prove --derivcheck-timeout=0 --saturation=50 --open-chains=1000 \
  --output=1_ICS_FIDO2PasskeyRecovery_Protocol_dependencyGraph.spthy \
  --quit-on-warning 2>&1 | tee 1_ICS_FIDO2PasskeyRecovery_Protocol_log.log

# ECS Baseline
/usr/bin/time -v tamarin-prover --with-dot=./tamarin-cleandot.py \
  1_ECS_FIDO2PasskeyRecovery_Protocol.spthy \
  --prove --derivcheck-timeout=0 --saturation=50 --open-chains=1000 \
  --output=1_ECS_FIDO2PasskeyRecovery_Protocol_dependencyGraph.spthy \
  --quit-on-warning 2>&1 | tee 1_ECS_FIDO2PasskeyRecovery_Protocol_log.log

# Hardened ECS
/usr/bin/time -v tamarin-prover --with-dot=./tamarin-cleandot.py \
  2_ECS_FIDO2PasskeyRecovery_HardenedProtocol.spthy \
  --prove --derivcheck-timeout=0 --saturation=50 --open-chains=1000 \
  --output=2_ECS_FIDO2PasskeyRecovery_HardenedProtocol_dependencyGraph.spthy \
  --quit-on-warning 2>&1 | tee 2_ECS_FIDO2PasskeyRecovery_HardenedProtocol_log.log
```

> **Note:** All models use the oracle `FIDO2_Passkey_Recovery.oracle` via the `heuristic: o` directive declared at the top of each `.spthy` file. Make sure the oracle file is in the same working directory.

---

## Oracle (`FIDO2_Passkey_Recovery.oracle`)

The oracle is a Python 3 script that guides Tamarin's backward proof search by assigning priority scores to proof goals. Lower score = higher priority.

| Score Range | Goal Type |
|---|---|
| 0–7 | Honest state facts (`State_*`), long-term keys, user registration |
| 10 | Persistent storage facts (`!Stored*`) |
| 15–25 | Channel facts (`Sec`, `In`, `Out`) |
| 80 | Term splitting equations (`splitEqs`) |
| 90 | Adversary knowledge (`!KU`, `KU(...)`) |
| 100–110 | Penalty for adversary performing XOR, asymmetric encryption, or QR code operations |

The penalisation of XOR and `aenc` for the adversary is critical: without it, Tamarin may spend unbounded time exploring adversary-derivable XOR combinations before trying the honest execution path.

---

## DOT Graph Post-Processor

`tamarin-cleandot.py` and `tamarincleandotlib.py` are a drop-in replacement for the `dot` binary, passed to Tamarin via `--with-dot`. The tool:

- Clusters proof-graph nodes by shared rule-name prefix (to visually group protocol threads)
- Applies heuristic-driven graph simplification (levels 0–3)
- Supports abbreviation of repeated sub-terms for readability
- Writes a debug log to `/tmp/tamarin-cleandot.log`

Originally authored by Cas Cremers (December 2012 – January 2013).

---

## Cryptographic Assumptions (Tamarin Builtins)

Both models declare:

```
builtins: asymmetric-encryption, xor
```

- **`asymmetric-encryption`**: Provides `aenc/2`, `adec/2`, and `pk/1` with the standard IND-CPA axioms.
- **`xor`**: Provides the XOR operator `(+)` with the standard equational theory (associativity, commutativity, self-cancellation).

---

## Channel Model

Three types of channels are modelled, all implemented as **secure** in ECS/Hardened models:

| Channel | Direction | Secure in ICS | Secure in ECS/Hardened |
|---|---|---|---|
| `C2S` | Client → Server | ✓ | ✓ |
| `S2C` | Server → Client | ✗ (Dolev-Yao) | ✓ |
| `P2P` | Peer → Client (QR) | ✓ | ✓ |
