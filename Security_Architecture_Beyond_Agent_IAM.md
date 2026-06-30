# Security Architecture — Application Security, Cryptography, Supply Chain & Threat Modeling

**Principal Engineer / AI Architect / FDE Reference · Gunasekar Jabbala**

> The IAM & Zero-Trust document covers agent-specific identity and access. This document is the broader security architect surface every Principal Engineer is expected to reason about regardless of whether AI agents are involved at all — the application-level vulnerability classes, the cryptographic primitives underlying every secure claim in this entire series, and the software supply chain risks that have become a primary attack vector industry-wide.

---

## Table of Contents

1. [The Security Architecture Maturity Model](#1-the-security-architecture-maturity-model)
2. [Application Security - The Vulnerability Classes That Recur](#2-application-security--the-vulnerability-classes-that-recur)
3. [Cryptography Fundamentals - What You Actually Need to Reason About Correctly](#3-cryptography-fundamentals--what-you-actually-need-to-reason-about-correctly)
4. [Software Supply Chain Security](#4-software-supply-chain-security)
5. [Threat Modeling - A Repeatable Process, Not an Art](#5-threat-modeling--a-repeatable-process-not-an-art)
6. [Complexity Reduction for Security Architecture Specifically](#6-complexity-reduction-for-security-architecture-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The Security Architecture Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Reactive Security | Security addressed after incidents, no systematic review process |
| **2** | Checklist Security | Standard vulnerability scanning, basic security reviews, OWASP-aware development |
| **3** | Threat-Modeled Security | Systematic threat modeling per system, supply chain verification, security built into the design phase |
| **4** | Security-as-Platform | Security controls as enforced, automated platform capabilities (policy-as-code, automated SBOM generation, continuous compliance verification) |

The IAM & Zero-Trust document operates at Level 3-4 specifically for agent identity. This document extends that same maturity target across the broader application and infrastructure security surface.

---

## 2. Application Security - The Vulnerability Classes That Recur

### 2.1 Injection Vulnerabilities - The Pattern Underlying SQL Injection, Command Injection, and Prompt Injection

**The unifying mechanism, worth stating precisely:** injection vulnerabilities all share the same root cause — untrusted input is interpreted as code or instructions rather than treated strictly as data, because the system failed to maintain a clear boundary between the two.

```json
{
  "injection_vulnerability_class": {
    "sql_injection": "untrusted input concatenated into a SQL query string, interpreted as query syntax",
    "command_injection": "untrusted input passed to a shell command, interpreted as shell syntax",
    "prompt_injection": "untrusted retrieved content or user input interpreted by the model as instructions rather than data to reason about"
  }
}
```

**Principal-level note, the connective insight worth making explicit in an interview:** prompt injection (covered as a specific concern in the RAG document and the Agent Orchestration document) is not a fundamentally new vulnerability class invented by LLMs — it's the exact same injection pattern security engineers have dealt with for decades, with the model substituting for the SQL interpreter or shell. The fix follows the same general principle as parameterized SQL queries: maintain a strict, structural boundary between trusted instructions and untrusted data, never relying on the data simply behaving because it usually does.

### 2.2 The OWASP Top 10, and Which Ones Matter Most for AI-Adjacent Systems

| OWASP Category | Why It Matters Specifically for AI/Agent Systems |
|---|---|
| Broken Access Control | Directly the IAM document's least-privilege agent identity problem — an agent with excessive standing permissions is a broken access control finding by definition |
| Injection | Prompt injection, covered above, plus traditional injection in any tool an agent calls |
| Security Misconfiguration | Default-open agent tool permissions, overly permissive service mesh policies |
| Vulnerable & Outdated Components | Directly Section 4 of this document — the AI ecosystem's rapid dependency churn makes this acutely relevant |
| Server-Side Request Forgery (SSRF) | An agent with a fetch-URL tool that doesn't validate or restrict target URLs can be manipulated into making requests to internal-only services |

**Principal-level note on SSRF specifically, since it's the least obviously AI-relevant on this list:** any tool that lets an agent fetch arbitrary URLs (a common and useful capability) is a potential SSRF vector if the target URL isn't validated against an allowlist — a manipulated agent could be directed to fetch an internal metadata service endpoint instead of the external URL it was ostensibly asked to retrieve, exfiltrating internal secrets through a tool that looks completely benign in its intended use.

### 2.3 Authentication vs. Authorization - The Distinction Worth Stating Precisely

Authentication answers "who are you," verifying identity. Authorization answers "what are you allowed to do," evaluating permissions for an already-authenticated identity. A surprising number of real vulnerabilities come from conflating these — systems that authenticate correctly but then fail to re-check authorization for every individual action, such as checking authorization once at login, then trusting the session indefinitely for increasingly sensitive actions without re-verification.

**Principal-level note:** this directly connects to the IAM document's continuous re-verification, not just session start, principle — that principle is the AI-agent-specific instance of this much older, more general authentication-versus-authorization discipline.

---

## 3. Cryptography Fundamentals - What You Actually Need to Reason About Correctly

### 3.1 Symmetric vs. Asymmetric Encryption - The Tradeoff That Explains TLS's Design

Symmetric encryption (AES) uses the same key to encrypt and decrypt — fast, but requires both parties to already share a secret key, which creates a key-distribution problem. Asymmetric encryption (RSA, elliptic curve) uses a public and private key pair — solves key distribution since the public key can be shared openly, but is computationally much more expensive than symmetric encryption.

```json
{
  "tls_handshake_summary": {
    "phase_1": "asymmetric encryption used briefly to securely establish a shared symmetric session key",
    "phase_2": "symmetric encryption such as AES used for the bulk of the actual data transfer, for performance",
    "why_hybrid": "gets asymmetric's key-distribution benefit without paying asymmetric's full computational cost for every byte transferred"
  }
}
```

**Principal-level note:** this hybrid pattern, using the expensive-but-distribution-friendly mechanism briefly to establish a shared secret, then switching to the cheap-but-shared-secret-requiring mechanism for the bulk of actual work, is worth recognizing as a general pattern, not just a TLS-specific detail. The same logic appears in mutual TLS service mesh setup and is conceptually similar to using an expensive precise model briefly to validate a decision that a cheap model then executes repeatedly.

### 3.2 Hashing vs. Encryption - A Distinction Frequently Confused

Encryption is reversible with the right key; hashing is designed to be one-way — you cannot recover the original input from a hash. This is why passwords should be hashed, with a slow, salted algorithm like bcrypt or Argon2, never encrypted — encryption implies an intended way to recover the original password, which is precisely what you don't want for stored credentials.

**Principal-level note:** the audit log hash-chaining pattern in the AI Governance document relies specifically on hashing's one-way, tamper-evident property, not encryption — a chained hash lets you detect if a past entry was altered, without needing to ever decrypt anything, which is the correct primitive for that specific integrity-verification goal.

### 3.3 Key Management - Where Theoretical Cryptography Meets Real Operational Risk

**Principal-level note, the honest practical reality:** the cryptographic algorithms themselves (AES, RSA, modern elliptic curve schemes) are essentially never the weak point in a real security failure — they're mathematically sound and well-vetted. Key management — how keys are generated, stored, rotated, and revoked — is where almost all real-world cryptographic failures actually occur. This is precisely why the IAM document's credential lifecycle schemas and HSM-backed key management discussion matter more in practice than deep cryptographic algorithm knowledge for most Principal-level security architecture decisions.
---

## 4. Software Supply Chain Security

### 4.1 Why This Became a Primary Attack Vector

Modern software depends on enormous dependency trees — a typical application may have hundreds or thousands of transitive dependencies, most never directly reviewed by the application's own developers. This makes the supply chain (the dependencies themselves, the build process, the artifact registry) an attractive attack vector — compromising one widely-used package can affect every downstream consumer simultaneously, a far higher-leverage attack than targeting one application directly.

### 4.2 Software Bill of Materials (SBOM)

```json
{
  "sbom_entry": {
    "component": "fastapi",
    "version": "0.115.0",
    "license": "MIT",
    "known_vulnerabilities": [],
    "source": "pypi",
    "dependency_depth": 1,
    "introduced_by": "direct_dependency"
  }
}
```

**Principal-level note:** an SBOM is the supply-chain equivalent of the AI Governance document's data lineage requirement (Section 3 there) — you cannot assess your actual exposure to a newly disclosed vulnerability in some deeply transitive dependency if you don't have a complete, queryable inventory of what's actually in your software, the same "you can't classify or audit what you haven't catalogued" principle as the AI system inventory record (AI Governance document, Section 2.1).

### 4.3 Dependency Pinning and the Tradeoff It Represents

Pinning exact dependency versions (rather than accepting a version range) guarantees reproducible builds and prevents an unexpected, potentially malicious update from silently entering your build — at the cost of not automatically receiving security patches, which now requires a deliberate, tracked update process rather than happening implicitly.

**Principal-level note:** this is a direct philosophical echo of the Fine-Tuning Workflow document's emphasis on pinning model and dataset versions for reproducibility (Section 2.2 there) — the same tradeoff between automatic-but-unpredictable updates and pinned-but-requires-deliberate-maintenance applies whether you're versioning a software dependency, a training dataset, or a deployed model.

### 4.4 Verifying Artifact Integrity — Signing and Provenance

```json
{
  "artifact_provenance": {
    "artifact": "container_image_fraud_detection_v14",
    "built_by": "ci_pipeline_run_8821",
    "source_commit": "a1b2c3d",
    "signed": true,
    "signature_verified_at_deploy": true,
    "build_provenance_attestation": "SLSA_level_3"
  }
}
```

**Principal-level note:** `signature_verified_at_deploy: true` is the enforcement mechanism that actually matters — generating a signature is necessary but not sufficient; the deployment pipeline must actually *verify* that signature before running the artifact, or an attacker who compromises the artifact registry (even without compromising the signing process) can simply substitute an unsigned or differently-signed malicious artifact that an unverifying deployment pipeline would run anyway.

### 4.5 The Specific Risk in AI/ML Supply Chains

**Principal-level note, the domain-specific extension:** the supply chain concept extends directly to model weights and training data, not just traditional software dependencies — downloading a pretrained model from a public hub carries analogous risk to installing an unvetted package, since model weights can theoretically be crafted to produce malicious behavior under specific trigger conditions (a still-emerging research area sometimes called model backdooring), and training data provenance (Fine-Tuning Workflow document, Section 3.2) is itself a supply chain integrity question for anything trained on externally-sourced data.

---

## 5. Threat Modeling — A Repeatable Process, Not an Art

### 5.1 STRIDE — A Structured Way to Enumerate Threats Systematically

Rather than relying on unstructured brainstorming (which reliably misses categories of threat), STRIDE provides a checklist of threat categories to systematically consider for any system component: **S**poofing (impersonating identity), **T**ampering (modifying data/code), **R**epudiation (denying having performed an action), **I**nformation Disclosure (exposing data to unauthorized parties), **D**enial of Service, **E**levation of Privilege.

```json
{
  "threat_model_entry": {
    "component": "agent_tool_invocation_endpoint",
    "stride_category": "elevation_of_privilege",
    "threat": "a compromised agent could attempt to invoke a tool outside its granted scope",
    "existing_mitigation": "policy decision point enforces per-task tool grants (IAM document Section 4.1)",
    "residual_risk": "low, contingent on PDP availability",
    "additional_mitigation_needed": "PDP itself needs high availability design, since its failure mode currently defaults to deny, which is safe but could cause availability issues at scale"
  }
}
```

**Principal-level note:** the `additional_mitigation_needed` field surfacing a second-order risk (what happens if the *security control itself* fails or becomes a bottleneck) is exactly the kind of follow-on thinking that distinguishes a thorough threat model from a superficial one — STRIDE gets you to the first-order threats systematically, but Principal-level threat modeling keeps asking "and what happens if this mitigation itself fails or doesn't scale."

### 5.2 Attack Trees — Modeling How Multiple Steps Combine Into a Successful Attack

While STRIDE enumerates threat categories per component, attack trees model how an attacker might chain multiple individually-insufficient steps into a successful overall attack — useful specifically for understanding compound risk that no single component's threat model would surface in isolation.

```json
{
  "attack_tree_root": "exfiltrate customer PII from RAG knowledge base",
  "paths": [
    {
      "steps": ["inject malicious instructions via an uploaded document", "manipulate agent into retrieving and including PII in its response", "exfiltrate via the response itself"],
      "mitigations_present_at_each_step": ["document validation (RAG doc Section 5.4)", "retrieval access-control filtering (RAG doc Section 5.4)", "output content scanning"]
    }
  ]
}
```

**Principal-level note:** the value of the attack tree format specifically is forcing explicit consideration of *defense in depth* — listing the mitigation present at each individual step shows whether a single mitigation failure collapses the entire defense (bad) or whether multiple independent layers would each need to fail for the attack to succeed (the actual goal of defense in depth, made visible and verifiable rather than just asserted).

### 5.3 When to Threat Model — Not Just Once at Launch

**Principal-level note:** threat modeling should be revisited at the same triggers as the AI Governance document's classification re-review triggers (Section 2.1 there) — any material change to a system's architecture, data sensitivity, or exposure surface should trigger a threat model update, not just a one-time exercise before initial launch that's never revisited as the system evolves.

---

## 6. Complexity Reduction for Security Architecture Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Number of distinct authentication mechanisms | One centralized identity provider and auth flow, not different mechanisms per service that each need independent security review |
| Dependency version sprawl | Pinned versions with a deliberate, tracked, scheduled update process, not unconstrained automatic updates across the dependency tree |
| Threat modeling scope per review | Focus deeply on the highest-risk components (data access, external-facing surfaces) rather than attempting uniform shallow coverage across every component |
| Cryptographic primitive choices | Use well-vetted standard libraries and protocols (TLS, established hashing algorithms); never implement custom cryptography |

---

## 7. Decision Framework

1. For any system accepting external or model-generated input, is there a clear, structural boundary preventing that input from being interpreted as instructions rather than data — or is the system implicitly trusting input to behave?
2. Do you have a complete, queryable inventory of dependencies (SBOM) sufficient to assess exposure when a new vulnerability is disclosed, or would assessing exposure require manual investigation under time pressure?
3. Is artifact integrity actually verified at deployment time, or only generated and never checked, which provides no real protection?
4. Has this system's threat model been revisited since its last material architecture change, or is it a one-time exercise from initial launch that no longer reflects the current system?
5. For each security mitigation in place, what's the second-order risk if that mitigation itself fails or becomes unavailable — and is that second-order risk itself being managed?

**The governing test:** security architecture should make the cost of a successful attack disproportionately high relative to the value gained — through defense in depth (Section 5.2), verified integrity (Section 4.4), and continuously re-evaluated threat models (Section 5.3) — rather than relying on any single control being perfect, since the consistent lesson across decades of real security incidents is that individual controls eventually fail; the architecture's resilience comes from not depending on any single one of them holding forever.

---

## Companion Documents

Part of the Principal Engineer / AI Architect / FDE core engineering series:

- `IAM_ZeroTrust_Agent_Architecture.md` — the agent-specific identity and access control that this document's broader application security principles extend
- `RAG_Architecture_Deep_Dive.md` — the prompt injection and retrieval poisoning concerns that Section 2.1's injection vulnerability class directly explains the mechanism behind
- `AI_Governance_Compliance_Schemas.md` — the audit trail hash-chaining and data lineage patterns that Sections 3.2 and 4.2 provide the cryptographic and supply-chain foundation for
- `Fine_Tuning_Workflow_Architecture.md` — the dataset and model versioning whose supply-chain risk is extended in Section 4.5
- `Cloud_Native_Kubernetes_Architecture.md` — the mutual TLS infrastructure that Section 3.1's hybrid cryptography pattern underlies
