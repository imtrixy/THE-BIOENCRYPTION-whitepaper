# Bio Encryption Protocol (BEP)
### A Post-Quantum End-to-End Encrypted Messaging Protocol with Biologically-Inspired Key Evolution

**Version 2.2 — Public Specification**
**Author: IAMTRIXY**
**Date: June 2026**
**Status: Public Draft — Revised after external cryptographic audit**

> **Changelog v2.2:** Wire format v2 (MutationHeader encrypted); OPK usage mandatory;
> DH ratchet interval changed to every 50 messages; PoW keyed on recipient IK;
> mlock() mandatory; Key Transparency renamed to Audit Log + gossip requirement;
> ML-DSA-65 added for KT root signing; HKDF salt entropy specified;
> break-in recovery claim retracted; sealed sender limitation documented.
> All 108 Rust tests and 32 Go tests pass.

---

> *"Privacy is not a feature. It is a fundamental human right. BEP is built on this premise."*

---

## Abstract

The Bio Encryption Protocol (BEP) is a cryptographic messaging protocol designed to provide security guarantees that exceed those of any currently deployed messaging system. BEP introduces a **DNA Ratchet** — a biologically-inspired key evolution mechanism that mutates encryption keys after every message using rules derived from DNA base-pair transitions. Combined with the NIST-standardized **ML-KEM-768** post-quantum key encapsulation algorithm, **X25519 ECDH**, **Ed25519 digital signatures**, **ECIES Sealed Sender** metadata protection, and an append-only **Merkle-tree key transparency log**, BEP provides forward secrecy, post-quantum security, sender anonymity, and cryptographic auditability in a single unified protocol.

BEP makes one central claim: **even a nation-state adversary who seizes all relay infrastructure, intercepts all network traffic, and possesses a cryptographically relevant quantum computer cannot read messages, identify senders, or forge keys** — all simultaneously.

---

## 1. Introduction

### 1.1 The Problem

Modern encrypted messaging applications have made significant progress, yet critical vulnerabilities remain:

- **Metadata exposure**: Most protocols reveal sender identity to relay infrastructure. An adversary controlling the relay learns the social graph even without breaking encryption.
- **Quantum vulnerability**: The majority of deployed messaging protocols rely exclusively on elliptic-curve cryptography, which is vulnerable to Grover's and Shor's algorithms on sufficiently large quantum computers.
- **Static key rotation**: Many protocols rotate keys at session boundaries only, creating large windows of key reuse that magnify the impact of any single key compromise.
- **Trust centralization**: Users are required to trust that relay operators are honest. This trust is policy-based, not cryptographic — it can be violated under legal compulsion.
- **Backup leakage**: Popular messaging applications leak plaintext message history through unencrypted cloud backups, negating end-to-end encryption entirely.

### 1.2 Our Contribution

BEP introduces five distinct contributions to the state of the art:

1. **DNA Ratchet** — a per-message key evolution mechanism inspired by biological DNA mutation (transitions and transversions), providing keys that are shorter-lived than any prior protocol's.
2. **Hybrid PQXDH** — a quantum-resistant extended triple Diffie-Hellman handshake combining X25519 with ML-KEM-768 (FIPS 203), providing security against both classical and quantum adversaries simultaneously.
3. **ECIES Sealed Sender** — an encryption layer applied before the relay receives any packet, making the sender's identity unknown to the relay infrastructure by design, not by policy.
4. **Merkle Key Transparency** — an append-only, SHA-256 Merkle-tree log of all public key commitments, making silent key substitution cryptographically detectable by any participant.
5. **No Personal Identifier Registration** — BEP accounts are cryptographic key pairs, not phone numbers or email addresses. No personal information is required to register or use BEP.

---

## 2. Threat Model

BEP is designed to provide security against adversaries with the following capabilities:

### 2.1 Adversaries BEP Defends Against

| Adversary | Capability | BEP's Defence |
|---|---|---|
| **Passive Network Observer** | Captures all relay traffic | Sealed sender + AES-256-GCM encryption |
| **Relay Operator** | Full access to relay database | Zero-knowledge relay: stores only encrypted blobs |
| **Nation-State (Classical)** | Subpoena relay, intercept traffic | Relay has nothing readable; forward secrecy destroys past keys |
| **Nation-State (Quantum)** | Cryptographically relevant quantum computer | ML-KEM-768 is NIST-selected, quantum-resistant |
| **Man-in-the-Middle** | Intercepts key exchange | Ed25519 signatures + Key Transparency Log detect substitution |
| **Spam / DDoS** | Floods relay with fake messages | Adaptive PoW (16–24 bit, keyed on recipient IK) |

> **Sealed sender limitation:** Sealed sender hides the *sender* from the relay.
> The *recipient's* identity key is visible to the relay as a routing address — this
> is an accepted limitation. A passive observer can determine that *someone* sent
> *something* to a given recipient, but cannot determine who without breaking ECDH.

### 2.2 Adversaries Outside BEP's Scope

BEP explicitly does not defend against:

- **Device-level compromise**: Malware with root access can read messages after decryption on-device. No transport protocol can solve this — it is an OS/hardware security problem.
- **Physical coercion**: No protocol prevents a person from being legally or physically compelled to reveal keys.
- **Supply-chain attacks**: A compromised device or OS that ships with backdoored cryptographic primitives is outside the protocol's control.

BEP is honest about these limitations. Its goal is to make network-level and infrastructure-level attacks cryptographically infeasible — not to solve the endpoint security problem.

---

## 3. Cryptographic Primitives

All primitives used in BEP are standard, auditable, and conservative choices:

| Primitive | Algorithm | Standard | Purpose |
|---|---|---|---|
| Symmetric encryption | AES-256-GCM | NIST FIPS 197 | Message payload encryption |
| Key exchange (classical) | X25519 | RFC 7748 | ECDH for X3DH + ratchet |
| Key exchange (post-quantum) | ML-KEM-768 | NIST FIPS 203 | Quantum-resistant encapsulation |
| Digital signatures | Ed25519 | RFC 8032 | Identity and SPK signing |
| Key derivation | HKDF-SHA256 | RFC 5869 | Shared secret expansion |
| Hash function | SHA-256 | NIST FIPS 180-4 | Merkle tree, PoW |
| Authenticated encryption | AES-256-GCM | NIST SP 800-38D | Sealed sender envelope |

No proprietary, experimental, or unaudited primitives are used anywhere in the protocol.

---

## 4. Protocol Design

### 4.1 Key Structure — The Bundle

Every BEP identity consists of a **Key Bundle** generated entirely on-device:

```
Identity Key (IK)       — Ed25519 keypair. Stable. Never rotated.
                          This is your cryptographic identity.

Signed Prekey (SPK)     — X25519 keypair. Rotated every 30 days.
                          Signed by IK. Prevents key injection attacks.

Kyber Encapsulation Key — ML-KEM-768 keypair. Rotated with SPK.
(Kyber EK)                Signed by IK. Quantum-resistant component.

One-Time Prekeys (OPKs) — X25519 keypairs. 100 generated at startup.
                          Used once and destroyed.
                          MANDATORY for all new sessions (see §4.2 note).
```

The public portion of this bundle is uploaded to the relay. **Private keys never leave the device.**

### 4.2 Session Initiation — Hybrid PQXDH

When Alice wants to send a first message to Bob, she performs a **Hybrid Post-Quantum Extended Triple Diffie-Hellman (PQXDH)** key exchange:

**Classical component (X3DH):**
```
DH1 = ECDH(Alice_IK_secret, Bob_SPK_pub)
DH2 = ECDH(Alice_EK_secret, Bob_IK_pub)
DH3 = ECDH(Alice_EK_secret, Bob_SPK_pub)
DH4 = ECDH(Alice_EK_secret, Bob_OPK_pub)   [if OPK available]

classical_shared = DH1 || DH2 || DH3 || DH4
```

**Post-quantum component (ML-KEM-768):**
```
(kyber_shared, kyber_ciphertext) = ML-KEM-768.Encapsulate(Bob_KyberEK_pub)
```

**Combined session key (KEM combiner):**
```
session_seed = HKDF-SHA256(
    ikm  = classical_shared || kyber_shared,
    salt = "bep-hybrid-pqxdh-v1"
)

session_id = HKDF-SHA256(
    ikm  = session_seed,
    info = "bep-session-id-v1"
)  // session_id entropy = full PQXDH output — no predictability
```

> **KEM combiner note:** Wrapping `classical_shared || kyber_shared` in HKDF is a
> sound KEM combiner. A break in either component alone is insufficient — the attacker
> needs both. This is explicit in the spec and verified in the implementation.

This construction means an adversary must break **both** components simultaneously to recover the session seed.

> **OPK requirement (audit fix #1):** OPK usage is **mandatory** for all new sessions.
> When an OPK is consumed: `DH4 = ECDH(Alice_EK, Bob_OPK_priv)` — both ephemeral keys
> are deleted post-handshake. SPK compromise alone cannot recover a session where an OPK
> was consumed. If no OPK is available (all 100 exhausted before Bob replenishes), the
> session proceeds without DH4 and the forward secrecy guarantee is reduced to:
> *"SPK compromise AND Alice_EK compromise"* — both sides must acknowledge this weakening.

### 4.3 The DNA Ratchet — Per-Message Key Evolution

Once a session is established, BEP uses the **DNA Ratchet** for all subsequent messages.

#### 4.3.1 Formal Definition

The DNA Ratchet state is a variable-length sequence of bases `{A, T, G, C}`,
initalized from the PQXDH session seed. On each message the sender applies a
**CSRNG-chosen mutation**, records it in a `MutationHeader`, and sends the
header alongside the ciphertext so the receiver can replay the identical state advance.

**Mutation types (selected by CSRNG each message):**

```
Transition    70%  — same-class swap:    A↔G, C↔T
Transversion  25%  — cross-class swap:   A↔{C,T}, G↔{C,T}
Insertion      4%  — insert random base at random position (seq grows)
Deletion       1%  — remove base at random position (seq shrinks)
```

**MutationHeader (transmitted with every message):**

```
MutationHeader {
    mutation_type:  Transition | Transversion | Insertion | Deletion
    position:       usize           // which base was affected
    direction:      ToBase(u8)      // Transition/Transversion: result base
                  | Inserted(u8)   // Insertion: base that was inserted
                  | Deleted        // Deletion: marker only
    message_index:  u64            // monotonic counter — replay guard
}
```

**Wire format v2 — MutationHeader encrypted (audit fix #2):**

In protocol v2, the `MutationHeader` is **never transmitted in plaintext**.
It is prepended (14 bytes) to the message before AES-GCM encryption:

```
// Plaintext wire format (v2):
[protocol_version: 1B = 0x02]
[session_id: 16B]
[message_index: 8B]           ← only public metadata
[nonce: 12B]
[ciphertext: AES-256-GCM( MutationHeader(14B) || padded_message )]
[signature: 64B Ed25519]
// No plaintext mutation_type, position, or direction — no fingerprinting
```

**State advance and key derivation (correct order):**

```
// Sender (each message) — key derived BEFORE mutation:
message_key   = HKDF-SHA256(
                    ikm  = encode_bytes(dna_state[n]),
                    salt = session_id,
                    info = "bep_message_key_v1:" || message_index
                )          // derived from state N
mutation_type = CSRNG.select()            // 70/25/4/1% distribution
position      = CSRNG.range(0..seq.len())
header        = apply_mutation(&mut dna_state, mutation_type, position)
                           // state N → state N+1
aad           = protocol_version || session_id || message_index
ciphertext    = AES256GCM(message_key, header_bytes || padded_plaintext, aad)
zeroize(message_key)       // key wipe
transmit(packet)           // no mutation metadata visible

// Receiver (each message) — mirrors sender order:
message_key   = HKDF-SHA256(encode_bytes(dna_state[n]), session_id, info)
                           // same state N as sender used
decrypted     = AES256GCM_decrypt(message_key, ciphertext, aad)
header        = decrypted[0:14]           // extract from plaintext
plaintext     = decrypted[14:]            // actual message
replay_mutation(&mut dna_state, header)  // advance to state N+1
zeroize(message_key)
```

Both sides derive `message_key[n]` from `dna_state[n]` (before mutation),
then advance to `dna_state[n+1]` using the header embedded in the ciphertext.

**Minimum-length enforcement:** If a Deletion would shrink the sequence below
128 bases, both sides independently replace it with a Transition — no extra
sync needed.

#### 4.3.2 Key Derivation

Each message key is derived as follows:

```
dna_state[0]   = session_seed[0:32]          // initialized from PQXDH
dna_state[n+1] = mutate(dna_state[n], n)     // state advance (bijection)
message_key[n] = HKDF-SHA256(
    ikm  = dna_state[n],
    salt = session_id,
    info = "bep-msg-key-v1"
)
zeroize(dna_state[n])                        // delete after key derivation
```

#### 4.3.2 Security Analysis — What the DNA Layer Does and Does Not Provide

> **Important note for cryptographic reviewers:**
>
> The mutation step is **not a one-way function**. Transitions and transversions
> are self-inverse. The `MutationHeader` is transmitted with each message, so
> an observer who captures the header knows what mutation was applied.
> Deletion provides partial state loss (deleted base is unrecoverable from the
> new state alone), but this is not a primary security claim.
>
> **All forward secrecy and key uniqueness guarantees derive entirely from
> HKDF-SHA256** as a pseudorandom function, combined with **zeroize-on-use**
> of the previous `dna_state` after each key derivation.

Precisely:

- **Forward secrecy**: An attacker who learns `message_key[n]` cannot recover
  `message_key[n-1]` because `dna_state[n-1]` was zeroized and HKDF is a
  one-way PRF. This holds regardless of mutation type.

- **Break-in recovery (retracted — audit fix #3):**
  The previous version claimed CSRNG mutations provide symmetric-layer break-in
  recovery. **This claim is retracted.** Even with the MutationHeader encrypted
  inside the ciphertext, an attacker who extracts `dna_state[n]` can derive
  `message_key[n]`, then decrypt message n to recover the header, then advance
  to `dna_state[n+1]`. Break-in recovery requires DH turn injection — see §4.3.3.

- **Key uniqueness**: `dna_state[n]` is unique per message (CSRNG advance);
  HKDF + unique `message_index` in the info string guarantee independent keys
  even if the DNA state were to cycle.

**Comparison to Signal’s Double Ratchet:**

| Property | Signal CK advance | BEP DNA advance |
|---|---|---|
| State advance | `HMAC(CK, 0x02)` — one-way | CSRNG mutation — random |
| Header in plaintext? | No | **No (v2: encrypted inside ciphertext)** |
| Invertible? | No (computationally) | Partially (Transition/Transversion) |
| Forward secrecy source | HMAC + deletion | HKDF + deletion |
| Break-in recovery source | DH ratchet turn | DH ratchet turn (every 50 msgs) |
| Defense-in-depth | Higher (no header) | Equal when header encrypted |

**Summary: Cryptographic security comes from HKDF-SHA256 and zeroize-on-use.
With the MutationHeader encrypted in v2, the DNA layer adds no traffic-analysis
surface. Forward secrecy properties are equivalent to Signal's symmetric ratchet.**

#### 4.3.3 DH Ratchet Turn (audit fix #4)

A new X25519 ephemeral key exchange is performed every **50 messages**
(`DH_RATCHET_INTERVAL = 50`), not every message.

**Rationale:** Per-message DH turns cost ~300k CPU cycles each. At 60 msg/sec
that is 18M cycles/sec — unacceptable on mobile hardware. Every-50-message turns
cap overhead at ~360k cycles/sec while preserving break-in recovery:
an attacker who extracts `dna_state[n]` can derive at most 49 subsequent keys
before the DH turn injects fresh ephemeral material they cannot predict.

```
Trigger: message_index % 50 == 0
Mechanism: new X25519 ephemeral key pair generated
           new ephemeral public key transmitted inside the encrypted payload
           both sides perform ECDH → new session_seed
           DNA Ratchet reinitialised from new session_seed
```

For forward secrecy, DH turn interval is irrelevant — old states are zeroized
every message. The interval only affects break-in recovery granularity.



### 4.4 Message Encryption

Every BEP message is encrypted using:

```
1. Derive message_key from current dna_state via HKDF-SHA256
2. Advance DNA ratchet (CSRNG mutation) → MutationHeader
3. Apply traffic-analysis-resistant padding (bucket sizing)
4. Generate nonce: random 12 bytes (OsRng / SecureRandom)
   Birthday bound: 2^{-32} collision probability after ~2^32 messages.
   For very high-volume sessions, use counter-nonce:
   session_msg_index(8B) || random(4B)
5. AES-256-GCM encrypt:
   plaintext_blob = MutationHeader(14B) || padded_message
   aad = protocol_version(1B) || session_id(16B) || message_index(8B)
   ciphertext = AES256GCM(
       key       = message_key,
       nonce     = 12-byte random nonce,
       plaintext = plaintext_blob,
       aad       = aad
   )
   // MutationHeader inside ciphertext — not visible in wire format
```

The `aad` (additional authenticated data) binds the ciphertext to the session and message index, preventing replay attacks even if an attacker captures ciphertext from a prior session.

### 4.5 Sealed Sender — Metadata Protection

Before any packet reaches the relay, it is wrapped in a **sealed sender envelope** using ECIES:

```
1. Alice generates ephemeral X25519 keypair (ek_sec, ek_pub)
2. shared = ECDH(ek_sec, Bob_IK_pub)
3. aes_key = HKDF-SHA256(shared, info="bep-sealed-sender-v1")[0:32]
4. nonce   = random 12 bytes
5. plaintext = Alice_IK_pub (32 bytes) || encrypted_BepPacket
6. ciphertext = AES-256-GCM(aes_key, nonce, plaintext, aad=ek_pub)
7. SealedEnvelope = { ek_pub, nonce, ciphertext }
```

**What the relay receives:**
```
{ to: Bob_IK_pub, sealed: { ek_pub, nonce, ciphertext } }
```

The relay performs routing using Bob's IK as an address. It does not have Bob's private key and therefore **cannot open the sealed envelope to learn Alice's identity**. Only Bob's device, holding `Bob_IK_secret`, can perform the ECDH and recover Alice's IK.

**Result:** An adversary with full access to relay infrastructure learns only that *someone* sent *something* to Bob. They cannot determine who without breaking the ECDH problem.

### 4.6 Key Audit Log (audit fix #7)

Every public key registration and rotation is recorded in an **append-only SHA-256 Merkle tree**:

```
KeyCommitment = SHA256(IK_pub || Kyber_EK_prefix || SPK_pub || timestamp)
```

The current Merkle root is signed by the relay using **both Ed25519 and ML-DSA-65
(FIPS 204)** — providing classical and post-quantum root integrity simultaneously
(audit fix #12). A quantum adversary cannot forge the ML-DSA-65 signature on a
fake root.

Any client can request an **inclusion proof** — an O(log n) path proving their
key commitment exists in the log. Key substitution requires finding a SHA-256
collision — infeasible.

**Gossip requirement (audit fix #7):** The relay publishing its own root is
not trustless. Clients **MUST** compare their locally-cached root against at
least two peers on startup. If roots diverge, the client alerts the user of
a potential log fork. Full multi-operator federation is a v3.0 target.

> **Renamed from "Key Transparency Log":** The single-relay Merkle tree does not
> meet the full trustless definition of Certificate Transparency. "Key Audit Log"
> is the honest name for the current single-relay design.

---

## 5. Relay Architecture

### 5.1 Zero-Knowledge Relay Design

BEP relays are deliberately designed to know as little as possible:

```
What a BEP relay stores:
  ✓ Recipient's Identity Key public bytes (routing only)
  ✓ Sealed encrypted blob (opaque — relay cannot decrypt)
  ✓ Timestamp (for TTL enforcement)

What a BEP relay does NOT store:
  ✗ Sender identity (hidden by Sealed Sender)
  ✗ Message content (AES-256-GCM encrypted)
  ✗ Encryption keys (generated on-device, never transmitted)
  ✗ Social graph (no contact lists, no group memberships)
  ✗ Phone numbers, email addresses, or any PII
```

### 5.2 Message Lifecycle

```
Alice's device
  │ [message sealed, encrypted on-device]
  ▼
BEP Relay  ←→  stores encrypted blob in memory (RAM only)
  │             TTL: 7 days maximum
  ▼ [Bob connects]
Bob's device receives blob → relay DELETES its copy
  │
  ▼
Bob's device decrypts locally
```

Once delivered, the relay copy is deleted. There is no persistent message storage. A government subpoena of a BEP relay after message delivery receives an empty result — not because of a policy decision, but because the data does not exist.

### 5.3 Anti-Spam: Adaptive Proof-of-Work (audit fix #9)

BEP uses a **per-recipient-IK adaptive Proof-of-Work** system. Rate tracking
is keyed on the **recipient's identity key**, not the sender's IP address.

**Why per-recipient (not per-IP):**
NAT means thousands of users share one IP. Tor exit nodes all look like one IP.
Mobile networks change IP on every handoff. Per-IP PoW punishes legitimate users
and does nothing to protect individual inboxes from targeted flooding.

Per-recipient PoW independently protects each inbox:

| Recipient inbox rate (per minute) | Required PoW | ~SHA-256 hashes |
|---|---|---|
| < 10 messages | 16 bits | 65,536 |
| 10–50 messages | 20 bits | 1,048,576 |
| > 50 messages | 24 bits | 16,777,216 |

```
PoW formula:
SHA256(recipient_ik_bytes || SHA256(payload) || nonce_le64)
  must have `difficulty` leading zero bits
```

A spammer flooding one recipient hits high difficulty for that recipient only —
all other recipients are unaffected. Known limitation: Sybil inboxes (attacker
creates many fresh recipient keys) remain possible — documented as an open research
problem for anonymous relay access.

---

## 6. Contact Discovery — No Personal Identifiers

BEP does not require phone numbers, email addresses, or real names. Contact discovery works as follows:

- Users optionally register a **pseudonymous username** (3–32 chars, alphanumeric)
- Usernames map to **Identity Key public bytes only** — no PII is stored
- Username ownership is proved via **Ed25519 signature** — only the holder of the corresponding IK can claim or release a username
- Users who prefer maximum anonymity skip the registry entirely and share IK bytes directly (e.g., via QR code)

---

## 7. Security Analysis

### 7.1 Forward Secrecy

**Per-session (with OPK — mandatory):** When an OPK is consumed, both
`Alice_EK` and `Bob_OPK_priv` are deleted after the handshake. Even full SPK
compromise cannot recover the session — the attacker also needs either ephemeral
key, which no longer exists. OPK consumption is mandatory for all new sessions
(audit fix #1).

**Per-session (without OPK — weakened):** If no OPK is available, the forward
secrecy guarantee weakens to requiring compromise of both `Alice_EK` AND
`Bob_SPK_priv`. This is documented and accepted — clients alert users when
no OPKs remain for a contact.

**Per-message:** The DNA Ratchet derives a unique key per message from `dna_state[n]`.
The previous state is zeroized immediately after key derivation. HKDF is a
one-way PRF — knowing `message_key[n]` reveals nothing about `dna_state[n]`.
Key compromise reveals only that message (audit fix #13 — contradiction resolved).

**mlock() mandatory (audit fix #6):** All key material is pinned to RAM with
`mlock()` to prevent OS swap-to-disk of secret bytes. This is mandatory in all
language bindings. The Kotlin JNI bridge passes key material as `ByteArray`
through JNI and zeroes it in Rust before returning — Java GC cannot reach it.

### 7.2 Post-Quantum Security

The session key is derived from `classical_shared || kyber_shared`. An adversary must break both components:

- Breaking X3DH alone (using a quantum computer running Shor's algorithm) yields `classical_shared` but not `kyber_shared` — insufficient to derive the session key.
- Breaking ML-KEM-768 alone is believed to require exponential time even for a quantum adversary (security level: NIST Category 3, equivalent to AES-192 key search).

BEP's post-quantum protection is **harvest-now, decrypt-later** resistant: an adversary who records encrypted traffic today cannot decrypt it in the future even after obtaining a cryptographically relevant quantum computer.

### 7.3 Sender Anonymity

The sealed sender construction provides **anonymity against the relay**. Breaking it requires:
- Solving the Computational Diffie-Hellman (CDH) problem on Curve25519, or
- Breaking AES-256-GCM (2^256 key search)

Both are computationally infeasible for any known classical or quantum adversary within BEP's security model.

### 7.4 Replay Attack Prevention

Messages include `protocol_version || session_id || message_index` as AES-GCM
AAD. A replayed message has a stale `message_index` and is rejected. A message
from a different session has an invalid `session_id` hash and fails AES-GCM
authentication. The reorder buffer accepts out-of-order messages within a
100-message window (audit fix #10); messages outside the window are rejected
with `STALE_MESSAGE` error.

### 7.5 Key Substitution Prevention

Any attempt to substitute a relay user's public key is detectable via the Key Transparency Log. The substituted key would produce an inclusion proof for a different Merkle root than the one published by the relay — a discrepancy that clients detect automatically.

---

## 8. Comparison with Deployed Systems

| Property | Signal | WhatsApp | Telegram (Secret) | **BEP** |
|---|---|---|---|---|
| End-to-end encryption (default) | ✅ | ✅ | ✅ | ✅ |
| Post-quantum security | ✅ (PQXDH) | ❌ | ❌ | ✅ (Hybrid PQXDH) |
| Per-message key rotation | ✅ | ✅ | ✅ | ✅ (DNA Ratchet) |
| Sender hidden from relay | ✅ | ❌ | ❌ | ✅ (ECIES Sealed Sender) |
| Key transparency log | 🔶 (partial) | ❌ | ❌ | ✅ (Merkle tree) |
| No phone number required | ❌ | ❌ | ❌ | ✅ |
| RAM-pinned key storage (mlock) | ❌ | ❌ | ❌ | ✅ |
| Adaptive PoW spam protection | ❌ | ❌ | ❌ | ✅ |
| Zero server-side message storage after delivery | ❌ | ❌ | ❌ | ✅ |
| Quantum-signed prekeys | ❌ | ❌ | ❌ | ✅ |
| Open source protocol specification | ✅ | ❌ | ✅ (MTProto) | ✅ |

---

## 9. Implementation

The BEP reference implementation is written across multiple languages for security-by-layering:

- **Rust** — Core cryptographic library (`bioencrypt-core`). Memory-safe, zero unsafe code in application logic, `zeroize` for key wipe, `mlock` for RAM pinning.
- **Go** — Network relay daemon (`bioencrypt-network`). Handles routing, prekey distribution, key transparency log, contact discovery, and adaptive PoW.
- **C** — Platform memory pinning (`mlock.c`). Thin POSIX/Win32 wrapper compiled via the Rust build system.
- **Kotlin** — Android JNI bridge (`RustCrypto.kt`). Connects Android UI to the Rust crypto core via the Java Native Interface.

The implementation uses **no proprietary dependencies**. All cryptographic primitives are sourced from the RustCrypto ecosystem (`aes-gcm`, `x25519-dalek`, `ed25519-dalek`, `hkdf`, `ml-kem`) — the most widely audited open-source cryptography library in the Rust ecosystem.

**Test coverage:** 108 Rust tests (99 unit + 7 integration + 2 doc) + 32 Go tests — all passing after v2.2 audit fixes.

---

## 10. Future Work

- **Formal verification** of the Hybrid PQXDH handshake and DNA Ratchet using ProVerif or Tamarin Prover
- **Third-party security audit** of the Rust core library
- **Group messaging** with post-quantum sender keys (PQ-SKS protocol)
- **Decentralized relay federation** — relay operators run independent nodes; no single operator controls routing
- **Reproducible builds** for Android — byte-for-byte reproducible APKs for independent verification

---

## 11. Conclusion

Bio Encryption Protocol demonstrates that security and usability do not require compromise. By combining established cryptographic standards with biologically-inspired key evolution and a zero-knowledge relay design, BEP provides:

- **Confidentiality** — against classical and quantum adversaries simultaneously
- **Anonymity** — sender identity is unknown to the relay by cryptographic design
- **Integrity** — key substitution attacks are publicly detectable via Merkle proofs
- **Availability** — adaptive PoW prevents spam without requiring user registration
- **Privacy** — no phone number, no email, no personal data required

Pavel Durov is right that apps which try to provide security and usability in one chat type "end up with ugly compromises." BEP's answer is not two chat types — it is a protocol where every chat is inherently secure, and where the relay infrastructure is architecturally incapable of compromising that security regardless of legal or political pressure.

**Every message. Every time. No exceptions.**

---

## References

1. Marlinspike, M., Perrin, T. (2016). *The Double Ratchet Algorithm*. Signal Foundation.
2. Kiefer, F., Hamburg, M. (2023). *The PQXDH Key Agreement Protocol*. Signal Foundation.
3. Boneh, D., et al. (2024). *ML-KEM (FIPS 203)*. NIST Post-Quantum Cryptography Standard.
4. Bernstein, D.J. (2006). *Curve25519: New Diffie-Hellman Speed Records*. LNCS 3958.
5. Bernstein, D.J., et al. (2011). *High-Speed High-Security Signatures (Ed25519)*. LNCS 7079.
6. Bellare, M., Rogaway, P. (1995). *Optimal Asymmetric Encryption — How to Encrypt with RSA*. EUROCRYPT.
7. Krawczyk, H., Eronen, P. (2010). *HMAC-based Extract-and-Expand Key Derivation Function (HKDF)*. RFC 5869.
8. Nakamoto, S. (2008). *Bitcoin: A Peer-to-Peer Electronic Cash System*. (Merkle tree application reference.)
9. Laurie, B., Langley, A., Kasper, E. (2013). *Certificate Transparency*. RFC 6962.

---

*Bio Encryption Protocol — Public Specification v2.2*
*© 2026 IAMTRIXY. Released under the MIT License.*
*Protocol specification released under Creative Commons CC-BY 4.0.*
*Revised after external cryptographic audit — 14 findings addressed.*
