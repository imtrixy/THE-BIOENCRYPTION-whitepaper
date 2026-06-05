<div align="center">

# 🧬 Bio Encryption Protocol (BEP)
### Public Whitepaper Repository

[![License: CC BY 4.0](https://img.shields.io/badge/Whitepaper-CC%20BY%204.0-lightblue.svg)](https://creativecommons.org/licenses/by/4.0/)
[![License: MIT](https://img.shields.io/badge/Code%20Samples-MIT-green.svg)](LICENSE-MIT)
[![Protocol Version](https://img.shields.io/badge/Protocol-v2.0-blueviolet.svg)](BEP_Whitepaper_v2.0.md)
[![Status](https://img.shields.io/badge/Status-Public%20Draft-orange.svg)]()
[![Author](https://img.shields.io/badge/Author-IAMTRIXY-black.svg)](https://github.com/imtrixy)

**A Post-Quantum End-to-End Encrypted Messaging Protocol**  
**with Biologically-Inspired Key Evolution**

[📄 Read the Whitepaper](#-read-the-whitepaper) · [🔐 Security Highlights](#-security-highlights) · [⚖️ License](#%EF%B8%8F-license) · [📬 Contact](#-contact)

</div>

---

## 📄 Read the Whitepaper

👉 **[BEP_Whitepaper_v2.0.md](BEP_Whitepaper_v2.0.md)**

This document is the complete public technical specification of the Bio Encryption Protocol. It covers:

- Threat model and adversary definitions
- Cryptographic primitives (AES-256-GCM, X25519, Ed25519, ML-KEM-768, HKDF-SHA256)
- Hybrid PQXDH session initiation
- The DNA Ratchet per-message key evolution mechanism
- ECIES Sealed Sender metadata protection
- SHA-256 Merkle-tree Key Transparency Log
- Zero-knowledge relay design
- Security analysis and comparison with Signal, WhatsApp, and Telegram

---

## 🔐 Security Highlights

BEP makes one central claim:

> **Even a nation-state adversary who seizes all relay infrastructure, intercepts all network traffic, and possesses a cryptographically relevant quantum computer cannot read messages, identify senders, or forge keys — all simultaneously.**

| Property | Signal | WhatsApp | Telegram (Secret) | **BEP** |
|---|:---:|:---:|:---:|:---:|
| End-to-end encrypted (default) | ✅ | ✅ | ✅ | ✅ |
| Post-quantum (ML-KEM-768) | ✅ | ❌ | ❌ | ✅ |
| Sender hidden from relay | ✅ | ❌ | ❌ | ✅ |
| Key transparency (Merkle) | 🔶 | ❌ | ❌ | ✅ |
| No phone number required | ❌ | ❌ | ❌ | ✅ |
| Per-message key rotation | ✅ | ✅ | ✅ | ✅ |
| RAM-pinned key storage | ❌ | ❌ | ❌ | ✅ |
| Zero server storage after delivery | ❌ | ❌ | ❌ | ✅ |
| Biologically-inspired DNA Ratchet | ❌ | ❌ | ❌ | ✅ |

---

## 🧬 What is the DNA Ratchet?

BEP's signature innovation is the **DNA Ratchet** — a key evolution mechanism inspired by biological DNA mutation.

Every encryption key is encoded as a DNA base sequence (A, T, G, C). After each message, the key evolves using biologically valid mutation rules:

- **Transitions**: A↔G, T↔C (same purine/pyrimidine class)
- **Transversions**: A↔T, A↔C, G↔T, G↔C (cross-class)

This means **every message uses a unique key** — not just per-session, but per-message. A key extracted from message 500 reveals nothing about messages 1–499. All prior keys are cryptographically gone.

---

## 🛡️ Why BEP is Subpoena-Resistant

The relay architecture is **zero-knowledge by design**:

```
What BEP relays store:
  ✓ Recipient's public key bytes  (routing only)
  ✓ Sealed encrypted blob         (relay cannot decrypt this)
  ✓ Timestamp                     (for 7-day TTL enforcement)

What BEP relays do NOT store:
  ✗ Sender identity   — hidden inside the sealed envelope
  ✗ Message content   — AES-256-GCM encrypted on-device
  ✗ Encryption keys   — never leave the user's device
  ✗ Contact lists     — no social graph data
  ✗ Personal info     — no phone numbers, emails, or names
```

When Bob receives a message, the relay copy is **immediately deleted**. After delivery, there is nothing to subpoena — not as a policy decision, but because the data does not exist.

---

## 📐 Protocol At a Glance

```
Session Setup (Hybrid PQXDH):
  classical_shared = X3DH(Alice_IK, Alice_EK, Bob_IK, Bob_SPK, Bob_OPK)
  quantum_shared   = ML-KEM-768.Decapsulate(Bob_KyberEK, kyber_ciphertext)
  session_key      = HKDF-SHA256(classical_shared || quantum_shared)

Per-Message Encryption (DNA Ratchet):
  dna_key[n+1]  = mutate(dna_key[n], message_index)
  message_key   = HKDF-SHA256(decode_dna(dna_key[n+1]))
  ciphertext    = AES-256-GCM(message_key, nonce, padded_plaintext)

Metadata Protection (Sealed Sender):
  ek            = X25519.GenerateEphemeral()
  shared        = ECDH(ek.secret, Bob_IK_pub)
  envelope_key  = HKDF-SHA256(shared, "bep-sealed-sender-v1")
  sealed        = AES-256-GCM(envelope_key, nonce, Alice_IK || ciphertext)
  relay_sees    = { to: Bob_IK, sealed: sealed }  ← no sender info
```

---

## 📚 Reference Implementation

The BEP reference implementation:

| Component | Language | Purpose |
|---|---|---|
| `bioencrypt-core` | **Rust** | Core cryptography — memory-safe, auditable |
| `bioencrypt-network` | **Go** | Relay daemon — routing, PoW, transparency |
| `mlock.c` | **C** | RAM pinning — prevents key swap to disk |
| `RustCrypto.kt` | **Kotlin** | Android JNI bridge |

**Test coverage: 137 tests — 137 passing.**

Reference implementation source: [github.com/imtrixy/BIO-ENCRYPTION](https://github.com/imtrixy/BIO-ENCRYPTION)

---

## ⚖️ License

This project uses two licenses to protect both the protocol specification and any associated code:

### Whitepaper (Protocol Specification)
The whitepaper (`BEP_Whitepaper_v2.0.md`) and all protocol documentation in this repository are licensed under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.

[![CC BY 4.0](https://licensebuttons.net/l/by/4.0/88x31.png)](https://creativecommons.org/licenses/by/4.0/)

**You are free to:**
- Share — copy and redistribute in any medium or format
- Adapt — remix, transform, and build upon the material for any purpose

**Under the following terms:**
- **Attribution** — You must give appropriate credit to **IAMTRIXY / Bio Encryption Protocol**, provide a link to this repository, and indicate if changes were made.
- You may not imply that IAMTRIXY endorses you or your use.

### Code Samples
Any code samples included in this repository are licensed under the **MIT License**.

See [LICENSE-MIT](LICENSE-MIT) for full terms.

---

## ⚠️ Important Notices

### Trademark
**"Bio Encryption Protocol"** and **"BEP"** (in the context of this messaging protocol) are identifiers associated with this project. Any use of these names in a competing product or service without permission is prohibited.

### No Warranty
This specification is provided "as is" without warranty of any kind. The authors make no representations about the suitability of this protocol for any particular purpose. Cryptographic protocols should undergo independent third-party audit before deployment in production systems handling sensitive data.

### Export Control
Cryptographic software may be subject to export control regulations in certain jurisdictions. Users are responsible for compliance with applicable laws.

---

## 📬 Contact

- **Author**: IAMTRIXY
- **GitHub**: [@imtrixy](https://github.com/imtrixy)
- **Protocol Repository**: [github.com/imtrixy/BIO-ENCRYPTION](https://github.com/imtrixy/BIO-ENCRYPTION)

To report security vulnerabilities in the protocol specification, please open a **private security advisory** via GitHub's security tab. Do not disclose security issues publicly until they have been reviewed.

---

## 📖 Citation

If you reference this work in academic or technical writing, please cite as:

```
IAMTRIXY. (2026). Bio Encryption Protocol (BEP): A Post-Quantum End-to-End
Encrypted Messaging Protocol with Biologically-Inspired Key Evolution.
Version 2.0. https://github.com/imtrixy/THE-BIOENCRYPTION-whitepaper
```

---

<div align="center">

*Built for a world where privacy is not optional.*

**🧬 Bio Encryption Protocol — Every message. Every time. No exceptions.**

</div>
