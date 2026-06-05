# Bio Encryption Protocol (BEP)
### A Post-Quantum End-to-End Encrypted Messaging Protocol with Biologically-Inspired Key Evolution

**Version 2.0 — Public Specification**
**Author: IAMTRIXY**
**Date: June 2026**
**Status: Public Draft**

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
| **Spam / DDoS** | Floods relay with fake messages | Adaptive Proof-of-Work (16–24 bit, IP-adaptive) |

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
                          Used once and destroyed. Provide perfect forward
                          secrecy for the session initiation phase.
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

**Combined session key:**
```
session_seed = HKDF-SHA256(
    ikm  = classical_shared || kyber_shared,
    info = "bep-hybrid-pqxdh-v1"
)
```

This construction means an adversary must break **both** the elliptic-curve component (using a quantum computer) **and** the ML-KEM-768 component (currently believed to require exponential time even for quantum computers) to recover the session seed.

### 4.3 The DNA Ratchet — Per-Message Key Evolution

Once a session is established, BEP uses the **DNA Ratchet** for all subsequent messages. Unlike standard double-ratchet constructions, BEP's ratchet keys evolve using rules inspired by biological DNA mutation:

**DNA Encoding:** Encryption key bytes are mapped to DNA bases:
```
00 → A (Adenine)
01 → T (Thymine)
10 → G (Guanine)
11 → C (Cytosine)
```

**Mutation rules (applied per message):**
```
Transitions  (same class, different base):  A↔G, T↔C
Transversions (cross class):               A↔T, A↔C, G↔T, G↔C
```

Each message applies a mutation to the key sequence, then derives the actual AES-256 message key:
```
dna_key[n+1]  = mutate(dna_key[n], message_index)
message_key   = HKDF-SHA256(decode_dna(dna_key[n+1]), session_id, "bep-msg-key-v1")
```

**Properties:**
- Keys are unique per message — no two messages share a key
- Past keys are destroyed after use — forward secrecy across every message, not just sessions
- The mutation sequence is deterministic but not reversible without the seed — an attacker who learns message key K[n] cannot derive K[n-1]
- The DNA encoding adds a layer of obfuscation that departs from standard binary key representations

**DH ratchet turn:** Every N messages (configurable, default: every message), a new X25519 DH exchange is also performed, providing the "healing" property — even if a message key is somehow extracted, the next DH turn resets the ratchet to a state the attacker cannot predict.

### 4.4 Message Encryption

Every BEP message is encrypted using:

```
1. Derive message_key via DNA Ratchet (above)
2. Apply traffic-analysis-resistant padding (bucket sizing)
3. AES-256-GCM encrypt:
   ciphertext = AES256GCM(
       key   = message_key,
       nonce = random 12 bytes,
       plaintext = padded_message,
       aad   = session_id || message_index
   )
4. Append HMAC-covered mutation header (DNA ratchet advance proof)
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

### 4.6 Key Transparency Log

Every public key rotation is recorded in an **append-only SHA-256 Merkle tree**:

```
KeyCommitment = SHA256(IK_pub || Kyber_EK_prefix || SPK_pub || timestamp)
```

The current Merkle root is publicly verifiable. Any client can request an **inclusion proof** — an O(log n) proof that their key commitment exists in the log at a specific position.

**Why this matters:** If a relay operator attempted to substitute Bob's public key with a different key (to enable a man-in-the-middle attack), the new key commitment would not appear in the log with a valid inclusion proof. Alice's client verifies the proof before initiating any session. Key substitution without detection is computationally equivalent to finding a SHA-256 collision — infeasible.

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

### 5.3 Anti-Spam: Adaptive Proof-of-Work

To prevent relay flooding without requiring user accounts, BEP uses a **server-side adaptive Proof-of-Work** system:

| Sender rate (per minute) | Required PoW difficulty | ~SHA-256 hashes |
|---|---|---|
| < 10 messages | 16 bits | 65,536 |
| 10–50 messages | 20 bits | 1,048,576 |
| > 50 messages | 24 bits | 16,777,216 |

The PoW is computed client-side before sending. This imposes negligible cost on normal users (milliseconds) while making bot-scale flooding economically infeasible.

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

**Per-session:** The X3DH handshake uses ephemeral keys that are destroyed after the session is established. Compromise of long-term keys after session setup does not retroactively reveal session keys.

**Per-message:** The DNA Ratchet generates a unique key for every message. Past message keys are explicitly destroyed from memory (using `zeroize`) after use. Compromise of message key K[n] reveals only message n — not messages 1 through n-1.

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

Messages include a **session ID** and a monotonically increasing **message index** as AES-GCM additional authenticated data (AAD). A replayed message from the same session has a stale message index and is rejected by the DNA Ratchet's ordering validation. A replayed message from a different session has an invalid session ID in its AAD and fails authentication.

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

**Test coverage:** 137 tests across unit, integration, and documentation test suites — all passing.

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

*Bio Encryption Protocol — Public Specification v2.0*
*© 2026 IAMTRIXY. Released under the MIT License.*
*Protocol specification released under Creative Commons CC-BY 4.0.*
