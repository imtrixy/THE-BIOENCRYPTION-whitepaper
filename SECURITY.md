# Security Policy — Bio Encryption Protocol Whitepaper

## Reporting a Vulnerability in the Protocol Specification

If you discover a cryptographic weakness, logical flaw, or security issue in
the **Bio Encryption Protocol specification** described in this whitepaper,
please report it responsibly.

### Please DO NOT open a public GitHub issue for security vulnerabilities.

Instead, use one of the following:

1. **GitHub Private Security Advisory** (preferred):
   Go to the **Security** tab of this repository → **Report a vulnerability**

2. **Contact the author directly** via GitHub:
   [@imtrixy](https://github.com/imtrixy)

---

## What We Consider a Valid Report

Valid security reports include:

- A flaw in the Hybrid PQXDH handshake that allows session key recovery
- A weakness in the DNA Ratchet construction that breaks forward secrecy
- A flaw in the Sealed Sender construction that reveals sender identity
- A weakness in the Merkle Key Transparency Log that allows undetected key substitution
- An attack on the Proof-of-Work construction that allows relay flooding bypass
- A misuse of any cryptographic primitive that reduces security below claimed levels

---

## What We DO NOT Consider a Valid Report

- Device-level exploits (malware, physical access) — outside protocol scope
- Social engineering attacks — outside protocol scope
- Attacks that require breaking AES-256-GCM, SHA-256, or ML-KEM-768 directly
  (we acknowledge these are security assumptions, not claims)

---

## Response Timeline

| Milestone | Target |
|---|---|
| Acknowledgement of report | Within 48 hours |
| Initial assessment | Within 7 days |
| Coordinated disclosure | 90 days from report (or sooner if fix is ready) |

---

## Responsible Disclosure Policy

We follow coordinated vulnerability disclosure. We will:
- Acknowledge your report promptly
- Work with you to understand and validate the issue
- Credit you in the fix (unless you prefer anonymity)
- Not take legal action against good-faith security researchers

We ask that you:
- Give us reasonable time to respond before public disclosure
- Not exploit any vulnerability for purposes other than demonstrating the issue
- Not share details of the vulnerability with third parties during the embargo period

---

## Scope

This repository contains the **protocol specification** only. For vulnerabilities
in the reference implementation code, please report to:
[github.com/imtrixy/BIO-ENCRYPTION](https://github.com/imtrixy/BIO-ENCRYPTION)
