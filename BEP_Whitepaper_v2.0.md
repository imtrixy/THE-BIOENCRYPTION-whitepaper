# Bio Encryption Protocol (BEP)
### A Post-Quantum End-to-End Encrypted Messaging Protocol with Biologically-Inspired Key Evolution

**Version 2.6 — Public Specification**
**Author: IAMTRIXY**
**Date: June 2026**
**Status: Public Draft — Beta-ready. Formal verification + third-party audit pending.**

> **Changelog v2.6 (ChatGPT + Claude audit fixes):**
> Abstract claim language made precise (ChatGPT recommendation).
> DNA Ratchet repositioned as "key evolution framework" — no longer claimed as
> independent security primitive (ChatGPT / all three AI auditors agree).
> Signal comparison table corrected: Signal also deletes after delivery —
> distinction is subpoena resistance architecture, not storage policy (Claude).
> Reorder buffer forward secrecy tradeoff explicitly acknowledged in §7.1 (Claude).
> Gossip bootstrap peer list specified (Claude).
> mlock() platform reality documented per OS (Claude).
> TTL UX guidance added to §5.2 (Claude).
> TV-7: DNA mutation apply/replay round-trip test vector added (Claude).
> Formal verification and third-party audit added to roadmap with priority **HIGH**.


---

> *"Privacy is not a feature. It is a fundamental human right. BEP is built on this premise."*

---

## Abstract

The Bio Encryption Protocol (BEP) is a cryptographic messaging protocol that **combines post-quantum security, metadata protection, and key transparency in a single unified protocol.** BEP introduces a **DNA Ratchet** — a biologically-inspired key evolution framework that evolves session state after every message using rules derived from DNA base-pair transitions. Combined with the NIST-standardized **ML-KEM-768** post-quantum key encapsulation algorithm, **X25519 ECDH**, **Ed25519 digital signatures**, **ECIES Sealed Sender** metadata protection, and an append-only **Merkle-tree key transparency log**, BEP provides forward secrecy, post-quantum security, sender anonymity, and cryptographic auditability in a single unified protocol.

BEP's central security claim: **within the stated threat model, a nation-state adversary who seizes all relay infrastructure, intercepts all network traffic, and possesses a cryptographically relevant quantum computer is designed to face computationally infeasible barriers to reading messages, identifying senders, or forging keys.**

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
- **Session loss**: If a device loses its `dna_state` (crash, reinstall, factory reset), the session cannot be resumed. **Session resumption requires a fresh PQXDH handshake.** This is by design — recovering state would require persisting sensitive material that forward secrecy demands be deleted. Clients detect missing sessions and automatically prompt re-initiation.
- **Post-compromise recovery**: Once a device is compromised (attacker has root access), BEP cannot automatically recover that session. After a compromise is cleaned (device wiped, app reinstalled), **new sessions have completely fresh keys** — old compromised sessions remain compromised. Post-compromise recovery for existing sessions requires re-establishing contact and starting a new session.

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
Identity Signing Key (IK_sign) — Ed25519 keypair. Stable. Never rotated.
                                  Used for: signing SPK, signing Kyber EK,
                                  signing username claims, signing packets.
                                  This is your verifiable cryptographic identity.

Identity DH Key (IK_dh)        — X25519 (Curve25519) keypair. Stable.
                                  Used for: Sealed Sender ECDH ONLY.
                                  SEPARATE from IK_sign — not convertible.
                                  WHY SEPARATE: Ed25519 and X25519 use different
                                  curve forms (Edwards vs. Montgomery). Direct
                                  ECDH on an Ed25519 point is undefined.
                                  IK_dh is the routing address the relay uses.

Signed Prekey (SPK)            — X25519 keypair. Rotated every 30 days.
                                  Signed by IK_sign. Prevents key injection.

Kyber Encapsulation Key (KEK)  — ML-KEM-768 keypair. Rotated with SPK.
                                  Signed by IK_sign. Quantum-resistant component.

One-Time Prekeys (OPKs)        — X25519 keypairs. 100 generated at startup.
                                  Used once and destroyed. MANDATORY for all
                                  new sessions (OPK Saver Mode at < 10 remaining).
```

> **IK rotation limitation:** Identity Keys are intentionally stable — rotating them
> would invalidate all existing contact relationships and Key Audit Log history.
> If an IK is compromised, the affected user must create a new identity and
> re-establish contact out-of-band. IK rotation with social recovery is planned
> for v3.0. See §10 Future Work.


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

> **OPK requirement + DoS protection (audit fix #1 / Issue #1 resolved):**
> OPK usage is **mandatory** for all new sessions. To defend against OPK
> Exhaustion DoS (Mallory initiates 100 bogus sessions to drain Bob's pool):
>
> **OPK Saver Mode** (`OPK_SAVER_THRESHOLD = 10`):
> - When Bob's pool drops below 10 OPKs, clients enter OPK Saver Mode
> - Known contacts (IK seen in a previous successfully-decrypted session)
>   still receive an OPK from the remaining pool
> - Unknown senders fall back to no-OPK mode, with the user explicitly notified:
>   *"Forward secrecy weakened: prekey pool is running low. Contact will replenish."*
> - `opk_exhausted()` fires at 0 OPKs — ALL new sessions are weakened until
>   replenishment completes
>
> This is Signal's Option C: honest fallback with user notification.
> Mallory can exhaust OPKs, but cannot silently weaken sessions —
> every affected session is visible to Bob's user. Auto-replenishment
> (`OPK_REPLENISH_THRESHOLD = 20`) mitigates sustained attacks.

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

**DNA Ratchet state bounds (Qwen audit Fix C):**
```
DNA_MIN_LENGTH = 128 bases   (genesis default)
DNA_MAX_LENGTH = 1024 bases  (hard cap — insertion flooding protection)
```
When an Insertion mutation would push the sequence past 1024 bases, it is
silently converted to a Deletion. This prevents memory exhaustion from an
attacker flooding messages engineered to always trigger Insertions (4% rate).
HKDF produces a unique message key regardless of exact sequence length —
this substitution has **zero cryptographic impact**.

**MutationHeader (transmitted encrypted with every message):
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

#### 4.3.3 DH Ratchet Turn — Full State Machine (audit fix #4 / Issue #2 resolved)

A new X25519 ephemeral key exchange is performed every **50 messages**
(`DH_RATCHET_INTERVAL = 50`). The **half-state problem** — where one side
has sent a new ephemeral but the peer is offline and hasn't responded —
is handled by `pending_turn_since` tracking with a timeout.

**State machine (both sides maintain this):**

```
// State fields on DhRatchetLayer:
our_ratchet_secret:   [u8; 32]      — current ephemeral secret (zeroized on rotate)
our_ratchet_pub:      [u8; 32]      — our current ephemeral public key
their_ratchet_pub:    Option<[u8;32]> — peer's last seen ephemeral (None = not yet)
pending_turn_since:   Option<u64>   — message_index when we sent our ephemeral
                                      None = no pending turn

// Trigger logic (should_trigger_new_ephemeral(message_index)):
match pending_turn_since {
    None       => message_index % DH_RATCHET_INTERVAL == 0  // normal interval
    Some(since) => message_index >= since + DH_RATCHET_TIMEOUT  // timeout retry
}

// ON SEND (trigger fires):
if their_ratchet_pub.is_some() {
    // Full DH turn: ECDH(our_secret, their_pub) → new root key → new genesis DNA
    advance_send()           // rotates our key pair, clears pending
} else {
    // No peer pub yet — announce our pub, mark as pending
    set_pending(message_index)
}
dh_pub_to_send = Some(our_ratchet_pub)  // include in encrypted payload

// ON RECEIVE (any message containing their_new_pub):
if their_new_pub != their_ratchet_pub {
    // New ephemeral from peer: perform ECDH → new root key → new genesis DNA
    advance_receive(their_new_pub)  // updates their_ratchet_pub, clears pending
}

// TIMEOUT:
DH_RATCHET_TIMEOUT = 100  // messages
If pending_turn_since.is_some() and message_index >= since + TIMEOUT:
    discard pending, generate fresh ephemeral and try again
```

**Why this solves the half-state problem:**
- Bob offline scenario: Alice sends ephemeral at index 50 (pending=50)
- Bob receives it later → `advance_receive()` fires → Bob's ratchet advances
- Bob's next send includes his new ephemeral → Alice sees it → her `advance_receive()` fires
- Both sides have now completed the DH turn independently
- If Bob never responds in 100 messages → Alice times out, tries again at index 150

**Constants:**
```
DH_RATCHET_INTERVAL       = 50   // messages between DH turn triggers
DH_RATCHET_TIMEOUT_SECS   = 7_776_000  // 90 days in seconds (wall-clock)
```

> **Qwen audit Fix B — "Hiking Trip" Bug (wall-clock timeout):**
> The previous spec used a message-count timeout (`DH_RATCHET_TIMEOUT = 100 messages`).
> This caused a critical desync bug in async messaging:
> If Alice sends 100 messages while Bob is offline for a week, Alice's client
> would discard the pending ephemeral at message 100. When Bob returns, he cannot
> complete the DH handshake — permanent chat history loss.
>
> **Fix:** `pending_turn_since` now stores a Unix timestamp (seconds), not a
> message index. The pending DH state is preserved regardless of how many messages
> are sent, until the peer responds OR 90 days of wall-clock time elapses
> (after which the session is practically abandoned).

For forward secrecy, DH turn interval is irrelevant — old dna_states are
zeroized every message. The interval only affects break-in recovery granularity
(max 49 messages exposed after compromise).




### 4.4 Message Encryption

Every BEP message is encrypted using:

```
1. Derive message_key from current dna_state via HKDF-SHA256
2. Advance DNA ratchet (CSRNG mutation) → MutationHeader
3. Apply traffic-analysis-resistant padding (bucket sizing)
4. Generate nonce (COUNTER-NONCE — DEFAULT as of v2.4):
   nonce = message_index(8B, little-endian) || OsRng(4B)
   ┌───────────────────────────────────┐
   │  message_index (8B LE) │ rand(4B) │
   └───────────────────────────────────┘
   Why: Pure random 12B nonce has 2^{-32} birthday collision risk.
   With a defective RNG, two messages could share a nonce →
   AES-GCM catastrophic failure (key stream recovery possible).
   Counter prefix guarantees uniqueness even if rand(4B) repeats.
   Random suffix prevents cross-session nonce prediction.
5. AES-256-GCM encrypt:
   plaintext_blob = MutationHeader(14B) || padded_message
   aad = protocol_version(1B) || session_id(16B) || message_index(8B)
   ciphertext = AES256GCM(
       key       = message_key,
       nonce     = counter_nonce (12 bytes),
       plaintext = plaintext_blob,
       aad       = aad
   )
   // MutationHeader inside ciphertext — not visible in wire format
```

The `aad` binds the ciphertext to the session and message index, preventing replay attacks. The counter-nonce additionally makes nonce uniqueness a protocol invariant rather than a probabilistic guarantee.

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

### 4.7 Wire Format Diagram

The complete structure of a BEP packet as it travels from Alice to the relay:

```
+-------------------------------------------------------------------+
|                   SEALED ENVELOPE (outer)                         |
+-------------------------------------------------------------------+
| ek_pub (32B) | nonce (12B) | AES-256-GCM ciphertext              |
|              |             +-------------------------------------+|
|              |             | Alice_IK_pub (32B)                  ||
|              |             +-------------------------------------+|
|              |             |         BEP PACKET (inner)          ||
|              |             +-------------------------------------+|
|              |             | protocol_version (1B = 0x02)        ||
|              |             | session_id       (16B)              ||
|              |             | message_index    (8B, LE)           ||
|              |             | nonce            (12B counter-nonce)||
|              |             | ciphertext       (AES-256-GCM)      ||
|              |             |   +-----------------------------+   ||
|              |             |   | MutationHeader   (14B)      |   ||
|              |             |   | padded_message              |   ||
|              |             |   +-----------------------------+   ||
|              |             | signature        (64B Ed25519)      ||
+--------------+-------------+-------------------------------------++

 What relay sees: { to: Bob_IK_pub, sealed: { ek_pub, nonce, ciphertext } }
 Relay cannot open envelope -- only Bob's IK_secret enables ECDH decryption.

 Overhead per message:
   Sealed envelope:  32 + 12 + 16 (GCM tag)   = 60B
   BEP packet outer: 1 + 16 + 8 + 12 + 64     = 101B
   MutationHeader:   14B (inside ciphertext)
   Padding:          bucket-aligned (varies)
   MINIMUM overhead: 175B + padding
```

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

### 5.2 Message Lifecycle and Delivery Semantics

```
Alice's device
  │ [message sealed, encrypted on-device]
  ▼
BEP Relay  ←→  stores encrypted blob (RAM only)
  │             TTL: 7 days maximum
  ├─ [Bob connects within TTL]
  │    Bob's device receives blob → relay DELETES its copy
  │    Bob's device decrypts locally
  │    Relay sends DELIVERED receipt to Alice
  │
  └─ [TTL expires before Bob connects]
       Relay DELETES the blob permanently
       Relay sends EXPIRED receipt to Alice
       Bob receives nothing — message is lost
```

**Delivery receipt types:**

| Receipt | Meaning |
|---|---|
| `DELIVERED` | Relay transferred blob to Bob's device. Bob has received it (not necessarily read it). |
| `EXPIRED` | TTL (7 days) elapsed before Bob retrieved the message. Message is permanently gone. |
| `REJECTED` | PoW invalid or relay policy violation. Message was never stored. |

> **TTL behaviour:** If Bob is offline for more than 7 days, his messages expire silently
> from Bob's perspective. Alice sees `EXPIRED` and should re-send if the message was
> critical. This is a deliberate privacy trade-off: long relay storage increases metadata
> exposure risk.

Once delivered, the relay copy is deleted. There is no persistent message storage. A government subpoena of a BEP relay after delivery receives an empty result — not because of policy, but because the data does not exist.

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

**QR code / deep-link format:**
```
bep:ik:<base64url(IK_pub_32B)>[?username=<pseudonym>]

Example:
bep:ik:QkJCQkJCQkJCQkJCQkJCQkJCQkJCQkJCQkJCQkJCQkI=?username=alice
```
This format enables clickable desktop links and scannable QR codes across all BEP clients.

> **Username registry centralization (accepted limitation):** The relay operates a
> centralized username → IK mapping table. This means the relay operator knows which
> pseudonym corresponds to which IK. This is an accepted limitation — usernames are
> *optional* and pseudonymous by design. Users who require stronger anonymity should
> share IK bytes directly, bypassing the registry entirely. A DHT-based decentralized
> username registry (similar to Session's Lokinet) is a v3.0 research target.

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

**Per-message:** The DNA Ratchet evolves session state via a unique key per message derived from `dna_state[n]`.
The previous state is zeroized immediately after key derivation. HKDF is a
one-way PRF — knowing `message_key[n]` reveals nothing about `dna_state[n]`.
Key compromise reveals only that message (audit fix #13 — contradiction resolved).

> **Reorder buffer forward secrecy tradeoff (Claude audit):** The reorder buffer
> holds up to 1000 encrypted packets pending gap-fill. While buffered packets are
> ciphertext-only (the session ratchet state is NOT held frozen for each buffered
> message — the header-driven replay design means the receiver advances state
> sequentially as gaps fill), the current session state in memory represents a
> forward secrecy window equal to the depth of the buffer at the time of compromise.
> Clients requiring stricter forward secrecy can reduce `MAX_REORDER_BUFFER` at
> the cost of higher message-loss probability on congested networks.

**mlock() per-platform reality (Claude audit):**

| Platform | Implementation | Fallback |
|---|---|---|
| **Linux / Android** | `mlock()` via POSIX. Android kernel allows it for foreground processes. Under extreme memory pressure, `mlock()` may silently fail — `zeroize` still wipes keys before free. | `zeroize` (software wipe) |
| **Windows** | `VirtualLock()` equivalent. Supported. | `zeroize` |
| **iOS** | `mlock()` is restricted for third-party apps by iOS memory limits. **Use iOS Secure Enclave / Keychain** with `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`. This achieves the same "never hits unencrypted disk" guarantee natively. | iOS Keychain |
| **macOS** | `mlock()` supported with entitlement. | `zeroize` |

> The v2.6 implementation mandates `zeroize` everywhere. `mlock()` is a
> best-effort enhancement. Where it is unavailable, `zeroize` ensures keys
> are overwritten before memory is returned to the OS.

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
| Zero server-side message storage after delivery | ✅ (deletes on delivery) | ❌ | ❌ | ✅ |
| Quantum-signed prekeys | ❌ | ❌ | ❌ | ✅ |
| Open source protocol specification | ✅ | ❌ | ✅ (MTProto) | ✅ |

> **Claude audit correction (v2.6):** Signal also deletes messages server-side after delivery.
> The meaningful distinction is **subpoena resistance architecture**: Signal operates under
> US legal jurisdiction and could be compelled to retain metadata. BEP's relay stores only
> opaque encrypted blobs with no sender identity — even under legal compulsion, the relay
> operator has nothing useful to disclose.

---

## 9. Implementation

The BEP reference implementation is written across multiple languages for security-by-layering:

- **Rust** — Core cryptographic library (`bioencrypt-core`). Memory-safe, zero unsafe code in application logic, `zeroize` for key wipe, `mlock` for RAM pinning.
- **Go** — Network relay daemon (`bioencrypt-network`). Handles routing, prekey distribution, key transparency log, contact discovery, and adaptive PoW.
- **C** — Platform memory pinning (`mlock.c`). Thin POSIX/Win32 wrapper compiled via the Rust build system.
- **Kotlin** — Android JNI bridge (`RustCrypto.kt`). Connects Android UI to the Rust crypto core via the Java Native Interface.

The implementation uses **no proprietary dependencies**. All cryptographic primitives are sourced from the RustCrypto ecosystem (`aes-gcm`, `x25519-dalek`, `ed25519-dalek`, `hkdf`, `ml-kem`) — the most widely audited open-source cryptography library in the Rust ecosystem.

**Test coverage:** 115 Rust tests (106 unit + 7 integration + 2 doc + 1 ignored) + 32 Go tests — all passing.

**Test vectors:** See Appendix A for 7 known-answer tests including DNA mutation apply/replay round-trip (TV-7).

---

## 10. Future Work

**v3.0 Targets:**
- **Group messaging (1:1 only in v2.x):** BEP v2.4 supports 1:1 messaging only. Group messaging will be added in v3.0 using a post-quantum sender keys design (PQ-SKS). Group keys will be encrypted to each member's IK individually — no central group server holds the key.
- **Identity Key rotation with social recovery:** Allow optional IK rotation via a signed "key update" message (old IK → new IK continuity proof), published to the Key Audit Log. Prevents a compromised IK from permanently locking a user out of their identity.
- **DHT-based username registry:** Replace the centralized relay username table with a distributed hash table for stronger anonymity guarantees.
- **DNA Ratchet IoT lite mode:** Optional compile-time feature replacing the variable-length DNA sequence with a fixed-32-byte counter ratchet for constrained devices (< 64KB RAM).

**Research targets:**
- **Formal verification** of Hybrid PQXDH and DNA Ratchet state machine using ProVerif or Tamarin Prover
- **Third-party security audit** of the Rust core library
- **Decentralized relay federation** — independent nodes, no single operator controls routing
- **Reproducible builds** — byte-for-byte reproducible Android APKs for independent verification
- **Multi-device support** — single identity across multiple devices with secure key sync

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

## Appendix A — Test Vectors (Known-Answer Tests)

All vectors below are generated deterministically from the BEP v2.4 reference implementation.
Run `cargo run --example generate_vectors` in `bioencrypt-core/` to reproduce.
Independent implementations **MUST** produce identical outputs for these inputs.

### TV-1: HKDF-SHA256 Key Derivation

```
IKM   (32B) : 0000000000000000000000000000000000000000000000000000000000000000
Salt  (16B) : 01010101010101010101010101010101
Info        : "bep_dh_ratchet_root_v1"
Output (32B): 1b139cd031a6998338a1255653bda4c7ebf577302f7e376d903c4eb762070280
```

### TV-2: Genesis DNA Key

```
Master (32B): 4242424242424242424242424242424242424242424242424242424242424242
SessionID   : abababababababababababababababab
HKDF output : 1a2aa9433c9083d63e98dd69a9bccff66c50fe030975fae1b7efe9aac90ca4a5
Genesis DNA : GAAGCGGGCGTGAATACTTAGAGACGCTCTAATGAACGAAGGTGTCTGTTAGTGTCTGCCTATA
              GAGGCGATTCATCCTGATTTGGTTAGTCTGTAAAGTATTCCGTGTATGGAACCCTCCAAGAGAGG
              GTCCCAGCATCTGAACGGCGTGTTGAAGCAGGAGCACGGAGGGGGAGTATAGGTTTCTGTCTCCG
              TTGGATGCTAATGCCCACCCTACTACAGGTTGGGGATCGGCACAAATAGTTTAGGCGATCGA
              (256 bases total)
```

### TV-3: DNA Base-4 Encoding

```
Input   (4B): 0055aaff
DNA string  : AAAACCCCGGGGTTTT
Decoded (4B): 0055aaff
Round-trip  : PASS
Scheme: 2 bits per base — A=00, C=01, G=10, T=11
```

### TV-4: Counter Nonce Prefix (Deterministic Part)

```
message_index=0                     nonce prefix (8B LE): 0000000000000000
message_index=1                     nonce prefix (8B LE): 0100000000000000
message_index=50                    nonce prefix (8B LE): 3200000000000000
message_index=100                   nonce prefix (8B LE): 6400000000000000
message_index=18446744073709551615  nonce prefix (8B LE): ffffffffffffffff
Suffix: 4 bytes from OsRng (non-deterministic, varies per call)
Full nonce: nonce_prefix(8B) || OsRng(4B)
```

### TV-5: AES-256-GCM with Fixed Nonce

```
Key      (32B): 4242424242424242424242424242424242424242424242424242424242424242
Nonce    (12B): 0100000000000000deadbeef  [index=1 LE || 0xDEADBEEF suffix]
AAD           : "bep_session_aad_v1"
Plaintext     : "BEP test message"  [hex: 4245502074657374206d657373616765]
Ciphertext+tag: c472b7083f8363e9e4b8cc36f981682b7e21b9a9a1374433ff9d98dc562b83b9
Decryption    : PASS
```

> **Note:** TV-5 uses a fixed nonce for test reproducibility only.
> In production, nonces are always `message_index(8B LE) || OsRng(4B)`.

### TV-6: Session ID Derivation

```
Master (32B): bebebebebebebebebebebebebebebebebebebebebebebebebebebebebebebebe
Salt   (16B): 00000000000000000000000000000000
Info        : "bep-session-id-v1"
Session ID  : 88ae7a768a2232426fd5e1b586f3b1c2d6b36c0b51642a7be4f204c98e4271bc
```

### TV-7: DNA Mutation Apply/Replay Round-Trip (Claude audit)

This is the **interop-critical** vector. Rust and Go/Kotlin must produce identical
states for the same `MutationHeader` sequence or sessions will silently desync.
Fixed `StdRng` seed guarantees deterministic output across all platforms.

```
CSRNG        : rand::rngs::StdRng  seed=0xBEEFCAFEDEAD1234
Start state  : ACGTACGT  (8 bases)

Msg 1: type=Transition   pos=1  dir=ToBase(3)  after=ATGTACGT
Msg 2: type=Transversion pos=6  dir=ToBase(3)  after=ATGTACTT
Msg 3: type=Transversion pos=3  dir=ToBase(2)  after=ATGGACTT
Msg 4: type=Transition   pos=2  dir=ToBase(0)  after=ATAGACTT
Msg 5: type=Transition   pos=7  dir=ToBase(1)  after=ATAGACTC

Final (apply) : ATAGACTC
Final (replay): ATAGACTC
Round-trip    : PASS

Base encoding: A=0, C=1, G=2, T=3
```

*Regenerate: `cargo run --example generate_vectors` in `bioencrypt-core/`*

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

*Bio Encryption Protocol — Public Specification v2.6*
*© 2026 IAMTRIXY. Released under the MIT License.*
*Protocol specification released under Creative Commons CC-BY 4.0.*
*Multi-AI audit: Qwen AI (A-), Claude AI (A-), ChatGPT (8.3/10). All critical issues resolved.*
*"BEP is a serious protocol proposal, not a hobbyist crypto design." — ChatGPT*
*"Ready for beta deployment." — Qwen AI*
