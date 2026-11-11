# Authorization and Clean-Room Engineering

This guide defines authorization boundaries, clean-room development hygiene, and sensitive data handling for reverse engineering web targets.

> Technical guidance only; not legal advice. Reverse engineering, copyright, and trade secret laws differ across jurisdictions.

---

## 1. Scope and Authorization Gate

Proceed only when the target meets at least one criterion:
1. **User-owned software**: Developed or owned by the user.
2. **Open-source software (check its license before reusing any code)**: Distributed under an open-source license.
3. **Written authorization**: Explicit written consent from the rights holder.
4. **Interoperability or learning, where permitted by applicable law and the target's terms of use**: Conducted for protocol interoperability or study.

### Prohibited Activities (Hard Refusals)
Never participate in or facilitate:
- Bypassing DRM, anti-cheat systems, copy protection, or hardware encryption.
- Evading authentication, paywalls, license checks, rate limits, CAPTCHAs, or bot detection.
- Exfiltrating private user data, confidential databases, or third-party infrastructure.
- Generating automated attack tooling or exploit payloads.

---

## 2. Ideas vs. Expression and Legal Boundaries

Reverse engineering analyzes functional behavior, interface contracts, data formats, state transitions, and algorithms. In many legal frameworks, functional behavior and methods of operation are distinguished from protected creative expression (source code, binary layouts, UI graphics, and proprietary copy).

### Critical Legal Distinctions
- **Risk Reduction vs. Legal Guarantee**: Following clean-room procedures reduces copyright infringement and trade secret risks; it does not eliminate legal risk and is not a formal legal defense or legal advice.
- **Jurisdiction-Specific Standards**: The boundary between unprotectable functional behavior and protected expression depends on jurisdiction; this skill and toolchain do not make legal determinations or decide legality.
- **LLMs and Public Training Data**: When an AI agent re-implements an interface from a specification, the model may reproduce code patterns from its public training data. Enforce directory isolation, require human code review of all generated code, and never ingest target code into the clean room.

| Phase | Analyzed (Ideas & Behavior) | Never Copied (Protected Expression) |
| :--- | :--- | :--- |
| **Analysis** | HTTP routes, JSON schemas, state machines | Source files, minified bundles, AST fragments |
| **Specification** | Parameter names, types, validation rules | Proprietary function code, decompiled text |
| **Rebuild** | Functional logic, newly authored components | Proprietary icons, logos, brand fonts, copy |

---

## 3. Clean-Room Hygiene Rules

When reconstructing features from reverse engineering findings, enforce rigorous separation:
1. **Directory Isolation**: Keep the **analysis sandbox** (captures, HARs, dumps) in a dedicated directory. Build the **reconstruction** in an entirely separate directory containing no analysis files.
2. **Specification-Only Handoff**: Only `CLONE_SPEC.md`, `PARITY_CHECKLIST.md`, and anonymized test fixtures cross from analysis to rebuild. Never copy, paste, decompile, or transliterate proprietary source code.
3. **Human Review Required**: A human engineer must inspect generated code for copyright compliance, originality, and absence of target code artifacts.
4. **Black-Box Test Oracle**: Treat the target strictly as a black-box oracle. Send test inputs to observe return values, then execute identical inputs against the rebuild.
5. **Provenance Logging**: Cite exact Evidence IDs (`ev_<64hex>`) or verified offsets in the specification for every documented behavior.
6. **Original Brand and Media Assets**: Do not copy logos, trademarks, color palettes, or illustrations. Author original design tokens, placeholder assets, and UI styling.

---

## 4. Sensitive Data Handling and Sanitization

Web captures (HAR, SAZ, and CDP recordings) intercept sensitive credentials and personal data:
- **Authentication Headers**: Scrub `Authorization: Bearer <token>`, `Basic <credentials>`, and custom session headers before saving or sharing captures.
- **Session Cookies**: Redact `Cookie` and `Set-Cookie` headers containing session identifiers (`connect.sid`, `JSESSIONID`, JWTs).
- **Personal Data**: Mask email addresses, phone numbers, passwords, and PII observed in HTTP payloads.
- **Safe Storage**: Never commit raw, unscrubbed HAR files to version control or paste sensitive tokens into chat messages. Scrub captures prior to exporting evidence bundles (`export_evidence_bundle`).
