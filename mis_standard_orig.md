# MCP Integrity Standard

## Begin MCP Integrity Standard

**Status:** Public Draft for Community Review

**Date:** February 24, 2026

**Version:** v1.0.0

**License:** Apache License 2.0

**Audience:** MCP server developers, MCP client developers, MCP host application developers, security architects, AI platform builders, registry operators, security tool vendors

<br>

## Table of Contents

1. Introduction
2. Goals, Scope, and Non-Goals
3. Normative Language
4. Terminology
5. Threat Model
6. Architecture Overview
7. Cryptographic Requirements
8. Sealed Manifest Format
9. Tool Interface Fingerprinting
10. Artifact Integrity Verification
11. Side-Effect Declarations
12. Build Provenance
13. Version Lineage and Change Classification
14. Multi-Party Attestation
15. Trust Levels
16. Key Discovery and Management
17. MCP Integration
18. Discovery Mechanisms
19. Verification Procedures
20. Client Behavior and Policy Enforcement
21. Publisher Guide
22. Conformance Requirements
23. Error Codes
24. Privacy Considerations
25. Security Considerations
26. Data Structures and JSON Schemas
27. Test Vectors
28. References
29. Appendix A: OWASP MCP Top 10 Coverage Matrix
30. Appendix B: Example Sealed Manifests

<br>

## 1. Introduction

### 1.1 Purpose

The MCP Integrity Standard defines a specification for verifying the integrity, provenance, and behavioral declarations of tools and servers operating over the Model Context Protocol (MCP). It provides a standardized document format (the Sealed Manifest), verification procedures, discovery conventions, and graduated trust levels that enable MCP clients, hosts, and registries to determine whether a tool is what it claims to be, whether it has been tampered with, and whether it has changed without the knowledge or consent of the user.

The MCP Integrity Standard is to MCP tools what code-signing and SBOM verification are to software packages. It does not replace or duplicate MCP's transport security, authentication, or authorization mechanisms. It adds an integrity layer that those mechanisms do not provide.

### 1.2 Relationship to MCP

The MCP Integrity Standard operates on top of the Model Context Protocol. It is designed as an MCP Extension as defined in the MCP specification (version 2025-11-25 and later). The MCP Integrity Standard:

- SHALL NOT require modifications to the MCP JSON-RPC message format.
- SHALL NOT require modifications to MCP transports (stdio, Streamable HTTP).
- SHALL NOT replace, duplicate, or interfere with MCP's OAuth 2.1 authorization flows.
- SHALL NOT modify or intercept `tools/call` request or response payloads.
- SHALL operate as metadata alongside MCP operations, not as a gatekeeper within them.
- SHALL degrade gracefully when either client or server does not support it.

### 1.3 Relationship to OWASP MCP Top 10

The MCP Integrity Standard directly addresses five of the ten risks identified in the OWASP MCP Top 10 (2025):

| OWASP Risk | How This Standard Addresses It |
|---|---|
| MCP02: Excessive Permissions & Scope Creep | Side-effect declarations and capability declarations make permission scope visible, auditable, and enforceable by policy |
| MCP03: Tool Poisoning | Tool interface fingerprinting detects any modification to tool descriptions, input schemas, or annotations |
| MCP04: Insecure Tool Discovery | Signed manifests with verified publisher identity prevent tool spoofing, shadowing, and name collision attacks |
| MCP05: Rug Pull / Server Compromise | Version lineage with hash chaining and change classification detects and flags malicious or undisclosed updates |
| MCP08: Insufficient Logging & Monitoring | Verification events (pass, fail, level, digest comparisons) provide structured, auditable integrity data |

A detailed coverage matrix is provided in Appendix A.

### 1.4 Design Principles

The MCP Integrity Standard is designed according to the following principles:

1. Integrity, not authorization. This standard answers "is this tool what it claims to be?" It does not answer "is this user allowed to use it?" or "is this tool behaving correctly right now?"

2. Graduated adoption. Implementations SHALL support incremental adoption through defined trust levels (0 through 3). Partial adoption SHALL provide partial value.

3. Publisher simplicity. Sealing a tool SHALL require no more than a signing key and a single CLI command or CI/CD step.

4. Client-side value without publisher adoption. Clients SHALL be able to derive integrity value (Trust On First Use fingerprinting, change detection) even when publishers have not adopted this standard.

5. Non-interference. This standard SHALL NOT interfere with any other MCP capability, extension, tool, or authorization mechanism.

6. Fail-safe by default, enforceable by policy. Verification results SHALL be advisory by default. Host applications MAY enforce blocking policies based on verification results.

<br>

## 2. Goals, Scope, and Out of Scope

### 2.1 Goals

The MCP Integrity Standard SHALL:

1. Provide a standardized format for declaring and signing tool integrity metadata (Sealed Manifests).
2. Enable cryptographic verification that a tool's interface (description, input schema, annotations) has not been modified from the publisher's declared version.
3. Enable cryptographic verification that a tool's underlying artifact (code, binary, package) matches the publisher's declared digest.
4. Provide machine-readable declarations of tool side effects and required capabilities.
5. Enable detection of undisclosed or unauthorized tool updates through version lineage and hash chaining.
6. Support multi-party attestation so that publishers, registries, scanners, and auditors can independently vouch for a tool's integrity.
7. Define graduated trust levels with clear requirements for each level.
8. Integrate with MCP as a standard MCP Extension with graceful degradation.
9. Define conformance requirements sufficient for independent implementations to interoperate.

### 2.2 Scope

The MCP Integrity Standard SHALL specify:

- The Sealed Manifest document format, including all required and optional fields.
- Cryptographic algorithms for signing and hashing.
- Tool interface fingerprint computation procedures, including canonicalization rules.
- Artifact digest computation and verification procedures.
- Side-effect declaration schema and vocabulary.
- Build provenance metadata format.
- Version lineage and change classification format.
- Multi-party attestation format and roles.
- Trust level definitions and requirements.
- Key discovery mechanisms.
- MCP Extension negotiation.
- Discovery mechanisms (`.well-known`, registry, local file).
- Verification procedures (install-time, list-time, invoke-time, update-time).
- Client behavior recommendations and policy enforcement points.
- Conformance requirements and test vectors.
- Error codes for verification failures.

### 2.3 Out of Scope

The MCP Integrity Standard SHALL NOT specify:

- Authentication or identity provisioning of users or agents.
- Authorization or access control decisions.
- OAuth token issuance, validation, or management.
- Transport layer security (TLS, mTLS).
- Runtime behavioral monitoring or anomaly detection.
- Reputation scoring algorithms or trust network topologies.
- Prompt injection detection or defense.
- Sandbox implementation or enforcement mechanisms.
- Logging storage formats or retention policies.
- Registry implementation details beyond the interface for hosting Sealed Manifests.

<br>

## 3. Normative Language

The key words "SHALL", "SHALL NOT", "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, RFC 2119, and RFC 8174, when, and only when, they appear in all capitals, as shown here.

<br>

## 4. Terminology

Sealed Manifest: A signed JSON document published by a tool publisher that declares the tool's identity, interface fingerprints, artifact digests, capability requirements, side-effect declarations, build provenance, and version lineage.

Tool Interface: The combination of a tool's description, input schema, output schema (if declared), and annotations as returned by the MCP `tools/list` method.

Interface Fingerprint: A cryptographic hash computed over the canonicalized tool interface, used to detect modifications to tool descriptions, schemas, or annotations.

Artifact Digest: A cryptographic hash of the tool's underlying code, binary, or package, used to detect modifications to the tool's executable content.

Publisher: The entity that authors and signs a Sealed Manifest. A publisher is identified by a namespace (e.g., `io.github.acme`) and is responsible for the accuracy of the manifest contents.

Attestation: A signed statement by an independent party (registry, scanner, auditor) vouching for a specific property of a tool or its manifest.

Trust Level: A graduated classification (Level 0 through Level 3) that represents increasing assurance about a tool's integrity, provenance, and verification status.

Sealing: The act of generating and signing a Sealed Manifest for a tool or set of tools.

Verification: The act of checking a Sealed Manifest's signature, comparing interface fingerprints and artifact digests against observed values, and evaluating trust level requirements.

Version Lineage: A hash-linked chain of Sealed Manifests across tool versions, enabling detection of broken chains or unauthorized version insertions.

TOFU (Trust On First Use): A verification mode where the client fingerprints a tool's interface on first encounter, caches the fingerprint, and alerts on subsequent changes. This mode provides integrity value without publisher adoption.

Side-Effect Declaration: A machine-readable statement within a Sealed Manifest describing what system resources a tool reads, writes, or accesses.

Change Classification: A machine-readable description of what changed between tool versions, enabling automated policy decisions about updates.

---

## 5. Threat Model

### 5.1 Actors

This standard considers the following threat actors:

- Malicious or compromised publishers who release tools containing malicious functionality, or whose signing keys are compromised by an external attacker.
- Supply-chain attackers who compromise tool distribution channels (package registries, CDNs, hosting infrastructure) to replace legitimate tools with tampered versions.
- Tool poisoners who modify tool descriptions, input schemas, or annotations to inject instructions that manipulate LLM behavior without the user's knowledge.
- Rug-pull attackers who publish a legitimate tool, gain trust and adoption, and then push a malicious update.
- Tool shadowers who publish tools with names or descriptions designed to impersonate or intercept calls to legitimate tools on other servers.

### 5.2 Attack Vectors and Mitigations

| ID | Attack Vector | Description | Standard Mitigation | OWASP Ref |
|----|--------------|-------------|-------------------|-----------|
| T1 | Tool Description Poisoning | Attacker modifies a tool's description to include hidden instructions that cause the LLM to exfiltrate data, execute unauthorized commands, or override user intent | Interface fingerprinting (Section 9) detects any modification to the tool description | MCP03 |
| T2 | Input Schema Manipulation | Attacker adds, removes, or modifies input schema fields to enable injection of malicious parameters or to silently expand the tool's accepted inputs | Interface fingerprinting (Section 9) includes the input schema in the fingerprint computation | MCP03 |
| T3 | Annotation Tampering | Attacker modifies tool annotations (e.g., changing `destructiveHint` from `true` to `false`) to suppress safety warnings | Interface fingerprinting (Section 9) includes annotations in the fingerprint computation | MCP03 |
| T4 | Artifact Replacement | Attacker replaces the tool's underlying code or binary with a malicious version while keeping the interface unchanged | Artifact digest verification (Section 10) detects any modification to the executable content | MCP03, MCP05 |
| T5 | Rug Pull via Malicious Update | Publisher (or attacker who compromises publisher) pushes an update that introduces malicious functionality | Version lineage (Section 13) detects broken hash chains; change classification flags unexpected changes to interface, capabilities, or side effects | MCP05 |
| T6 | Silent Permission Escalation | Tool update adds new capabilities or side effects without disclosing the change | Side-effect declarations (Section 11) combined with change classification (Section 13) make permission changes visible and auditable | MCP02 |
| T7 | Tool Name Spoofing / Shadowing | Attacker publishes a tool with the same name as a legitimate tool on a different server to intercept calls | Signed manifests bind tool identity to a verified publisher namespace; clients can verify publisher identity matches expected source | MCP04 |
| T8 | Manifest Forgery | Attacker creates a fake Sealed Manifest for a tool they do not control | Cryptographic signatures using publisher's private key; key discovery via independent trust sources (Section 16) | MCP04 |
| T9 | Key Substitution | Attacker replaces the public key used for verification to make forged signatures appear valid | Keys are resolved from external trust sources (registry JWKS, `.well-known` endpoint), not embedded in the manifest itself | MCP04 |
| T10 | Unauditable Tool Usage | Organization cannot determine which tools were invoked, whether they were verified, or what integrity level they met | Verification event logging (Section 20) provides structured audit data for every verification decision | MCP08 |

### 5.3 Out-of-Scope Threats

The following threats are acknowledged but not addressed by this standard:

- Prompt injection via tool output: Malicious content returned by a tool in its execution result. This requires output sanitization and is orthogonal to pre-execution integrity verification.
- Behavioral drift without interface change: A tool that returns different results depending on context without modifying its interface. This requires runtime behavioral monitoring.
- Compromised build infrastructure: If the build system itself is compromised, provenance attestations may be falsified. SLSA Level 3 addresses this; this standard references SLSA but does not redefine it.
- Social engineering of human reviewers: An attacker who convinces an auditor to attest a malicious tool. Multi-party attestation reduces but does not eliminate this risk.

---

## 6. Architecture Overview

### 6.1 Components

The MCP Integrity Standard defines the following components:

```
┌──────────────────────────────────────────────────────────────┐
│                   MCP Integrity Standard                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  Sealed Manifest                       │  │
│  │                                                        │  │
│  │  ┌──────────┐ ┌───────────┐ ┌───────────────────────┐  │  │
│  │  │ Tool     │ │ Artifact  │ │ Side-Effect           │  │  │
│  │  │ Interface│ │ Digest    │ │ Declarations          │  │  │
│  │  │ Finger-  │ │           │ │                       │  │  │
│  │  │ prints   │ │           │ │                       │  │  │
│  │  └──────────┘ └───────────┘ └───────────────────────┘  │  │
│  │  ┌──────────┐ ┌───────────┐ ┌───────────────────────┐  │  │
│  │  │ Build    │ │ Version   │ │ Capability            │  │  │
│  │  │ Prove-   │ │ Lineage   │ │ Declarations          │  │  │
│  │  │ nance    │ │           │ │                       │  │  │
│  │  └──────────┘ └───────────┘ └───────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ Multi-Party  │  │ Trust Levels │  │ Verification     │    │
│  │ Attestation  │  │ (0-3)        │  │ Procedures       │    │
│  └──────────────┘  └──────────────┘  └──────────────────┘    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ Key          │  │ Discovery    │  │ MCP Extension    │    │
│  │ Management   │  │ (.well-known │  │ Integration      │    │
│  │              │  │  + registry) │  │                  │    │
│  └──────────────┘  └──────────────┘  └──────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Roles

| Role | Responsibility |
|------|---------------|
| **Publisher** | Authors tools, generates signing keys, creates and signs Sealed Manifests, publishes manifests to discovery endpoints |
| **Client** | Fetches Sealed Manifests, performs verification, computes interface fingerprints, enforces or advises based on policy |
| **Host Application** | Presents verification results to users, enforces trust level policies, manages TOFU fingerprint cache |
| **Registry** | Hosts Sealed Manifests alongside tool metadata, verifies publisher namespace ownership, provides key discovery |
| **Scanner** | Performs automated analysis of tools (prompt injection scanning, malware detection, side-effect verification) and produces attestations |
| **Auditor** | Performs human review of tools and produces attestations for high-assurance environments |

### 6.3 Data Flow

```
 Publisher                    Registry / Discovery          Client / Host
 ────────                    ────────────────────          ─────────────
     │                              │                           │
     │  1. Seal tool                │                           │
     │  (sign manifest)             │                           │
     │──────────────────────────────>                           │
     │  2. Publish manifest         │                           │
     │                              │                           │
     │                              │   3. Fetch manifest       │
     │                              │<──────────────────────────│
     │                              │──────────────────────────>│
     │                              │   4. Return manifest      │
     │                              │                           │
     │                              │                           │ 5. Verify signature
     │                              │                           │ 6. Connect to MCP server
     │                              │                           │ 7. Call tools/list
     │                              │                           │ 8. Compute interface
     │                              │                           │    fingerprints
     │                              │                           │ 9. Compare against manifest
     │                              │                           │ 10. Evaluate trust level
     │                              │                           │ 11. Enforce policy / advise
     │                              │                           │ 12. Log verification event
```

---

## 7. Cryptographic Requirements

### 7.1 Signing Algorithms

Implementations SHALL support the following algorithms for manifest and attestation signing:

| Algorithm | Requirement | Identifier | Use |
|-----------|------------|------------|-----|
| Ed25519 (EdDSA over Curve25519) | REQUIRED | `Ed25519` | Manifest signing, attestation signing |
| ES256 (ECDSA over P-256) | RECOMMENDED | `ES256` | Environments with existing ECDSA infrastructure |

Implementations SHALL NOT use RSA for manifest or attestation signing.

Implementations SHALL NOT use algorithms with known weaknesses, including but not limited to: MD5, SHA-1, DSA, RSA key sizes below 2048 bits, or any algorithm not listed above.

### 7.2 Hash Algorithms

Implementations SHALL support the following hash algorithms for fingerprint and digest computation:

| Algorithm | Requirement | Identifier | Use |
|-----------|------------|------------|-----|
| SHA-256 | REQUIRED | `sha256` | Interface fingerprints, artifact digests, manifest digests |
| SHA-512 | RECOMMENDED | `sha512` | High-security deployments |

All digest values SHALL be represented as lowercase hexadecimal strings prefixed with the algorithm identifier and a colon (e.g., `sha256:0f1e2d3c4b5a...`).

### 7.3 Canonicalization

Before hashing, all JSON inputs SHALL be canonicalized according to RFC 8785 (JSON Canonicalization Scheme, JCS). This ensures that semantically identical JSON documents produce identical hashes regardless of key ordering, whitespace, or formatting differences.

Implementations MUST:

1. Parse the input JSON into a native data structure.
2. Serialize the data structure according to RFC 8785 (sorted keys, no whitespace, Unicode normalization).
3. Encode the serialized output as UTF-8 bytes.
4. Apply the hash algorithm to the resulting byte sequence.

### 7.4 Signature Format

Signatures SHALL be encoded as base64url strings (RFC 4648, Section 5) without padding.

Signed payloads SHALL use the DSSE (Dead Simple Signing Envelope) format, defined by the in-toto project:

```json
{
  "payloadType": "application/vnd.mcp-integrity.manifest+json",
  "payload": "<base64url-encoded canonical manifest>",
  "signatures": [
    {
      "keyid": "<key identifier>",
      "sig": "<base64url-encoded signature>"
    }
  ]
}
```

This format separates the signed content from the signature, supports multiple signers (for multi-party attestation), and aligns with the SLSA provenance ecosystem.

---

## 8. Sealed Manifest Format

### 8.1 Overview

A Sealed Manifest is a signed JSON document that declares the integrity metadata for one or more tools published by a single MCP server. It is the core data structure of the MCP Integrity Standard.

### 8.2 Top-Level Structure

A Sealed Manifest SHALL contain the following top-level fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `seal_version` | string | REQUIRED | Version of this standard. SHALL be `"1.0"` for this version. |
| `server` | object | REQUIRED | Server identity and metadata. |
| `tools` | array | REQUIRED | Array of tool integrity declarations. |
| `attestations` | array | REQUIRED | Array of one or more attestations (signatures). |
| `issued_at` | string | REQUIRED | ISO 8601 timestamp of manifest creation. |
| `expires_at` | string | OPTIONAL | ISO 8601 timestamp after which this manifest SHOULD be re-fetched. |

### 8.3 Server Object

The `server` object SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | REQUIRED | Server name as registered in the MCP server's `serverInfo.name`. |
| `version` | string | REQUIRED | Server version as registered in the MCP server's `serverInfo.version`. |
| `publisher` | object | REQUIRED | Publisher identity. |
| `publisher.id` | string | REQUIRED | Publisher identifier (e.g., email or domain). |
| `publisher.namespace` | string | REQUIRED | Registry namespace (e.g., `io.github.acme`). |
| `publisher.url` | string | OPTIONAL | Publisher's website or profile URL. |

### 8.4 Tool Object

Each entry in the `tools` array SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | REQUIRED | Tool name as returned by `tools/list`. |
| `version` | string | REQUIRED | Tool version. |
| `interface` | object | REQUIRED | Interface fingerprints. See Section 9. |
| `artifact` | object | OPTIONAL | Artifact integrity information. See Section 10. |
| `side_effects` | object | REQUIRED | Side-effect declarations. See Section 11. |
| `capabilities_required` | array | REQUIRED | Required and optional capabilities. See Section 8.5. |
| `provenance` | object | OPTIONAL | Build provenance metadata. See Section 12. |
| `lineage` | object | OPTIONAL | Version lineage information. See Section 13. |

### 8.5 Capability Declarations

Each entry in the `capabilities_required` array SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resource` | string | REQUIRED | Resource category (e.g., `filesystem`, `network`, `environment`, `clipboard`, `database`). |
| `actions` | array of strings | REQUIRED | Permitted actions on the resource (e.g., `["read"]`, `["read", "write"]`, `["execute"]`). |
| `scope` | string | OPTIONAL | Restriction on the resource (e.g., a filesystem path pattern, a network hostname, an environment variable name). |
| `required` | boolean | REQUIRED | Whether this capability is required for basic tool operation (`true`) or used only for optional features (`false`). |

Resource categories are an extensible vocabulary. The following categories are defined by this standard:

| Category | Description | Example Scopes |
|----------|-------------|---------------|
| `filesystem` | File system read or write access | `/home/user/docs`, `/tmp/*` |
| `network` | Network communication | `api.example.com`, `*.example.com`, `*` (any) |
| `environment` | Environment variable access | `API_KEY`, `HOME`, `*` |
| `database` | Database read or write access | `postgres://host/db`, `*` |
| `clipboard` | System clipboard access | — |
| `process` | Ability to spawn child processes | `git`, `npm`, `*` |
| `system` | Operating system state modification | `registry`, `services`, `cron` |

Implementations MAY define additional resource categories. Custom categories SHOULD be prefixed with a reverse-domain namespace (e.g., `com.example.custom-resource`).

<br>

## 9. Tool Interface Fingerprinting

### 9.1 Purpose

Tool interface fingerprinting provides cryptographic detection of modifications to a tool's description, input schema, output schema, and annotations. This is the primary defense against tool poisoning (OWASP MCP03).

### 9.2 Interface Object

Each tool's `interface` object in the Sealed Manifest SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description_digest` | string | REQUIRED | Hash of the canonicalized tool description string. |
| `input_schema_digest` | string | REQUIRED | Hash of the canonicalized JSON input schema object. |
| `annotations_digest` | string | OPTIONAL | Hash of the canonicalized JSON annotations object. Required if the tool declares annotations. |
| `output_schema_digest` | string | OPTIONAL | Hash of the canonicalized JSON output schema object, if the tool declares one. |
| `composite_digest` | string | REQUIRED | Hash of the concatenation of all individual digests above, in the order listed. This is the single value used for quick verification. |

### 9.3 Fingerprint Computation Procedure

To compute the interface fingerprints for a tool, implementations SHALL perform the following steps:

**Step 1: Extract interface components from the `tools/list` response.**

For each tool in the response, extract:
- `description` (string)
- `inputSchema` (JSON object)
- `annotations` (JSON object, if present)

**Step 2: Compute individual digests.**

```
description_digest = HASH(UTF8_ENCODE(description))
input_schema_digest = HASH(JCS_CANONICALIZE(inputSchema))
annotations_digest = HASH(JCS_CANONICALIZE(annotations))    // if present
output_schema_digest = HASH(JCS_CANONICALIZE(outputSchema))  // if present
```

Where:
- `HASH` is SHA-256 (REQUIRED) or SHA-512 (OPTIONAL).
- `JCS_CANONICALIZE` is JSON Canonicalization per RFC 8785.
- `UTF8_ENCODE` is UTF-8 encoding of the string.
- The description is hashed as a raw UTF-8 string, not as a JSON value (i.e., without surrounding quotes or escaping), because MCP returns it as a plain string field.

**Step 3: Compute composite digest.**

```
composite_input = description_digest || ":" ||
                  input_schema_digest || ":" ||
                  annotations_digest || ":" ||     // empty string if absent
                  output_schema_digest              // empty string if absent

composite_digest = HASH(UTF8_ENCODE(composite_input))
```

The composite digest provides a single value for fast comparison. If the composite digest matches, all individual digests match. If it does not match, clients SHOULD compare individual digests to identify which component changed.

**Step 4: Format the digest string.**

All digest values SHALL be formatted as `algorithm:hex_digest`, for example:

```
sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

### 9.4 Verification Procedure

To verify a tool's interface integrity, implementations SHALL:

1. Obtain the Sealed Manifest for the server (via discovery, cache, or prior fetch).
2. Locate the tool entry in the manifest's `tools` array by matching the `name` field.
3. Call `tools/list` on the MCP server (or use the cached result if the `tools/list` response is already available from the current session).
4. Compute the interface fingerprints for the tool as described in Section 9.3.
5. Compare the computed `composite_digest` against the manifest's `interface.composite_digest`.
6. If the composite digest does not match, compare individual digests to identify which component changed.
7. Return a verification result (see Section 20) indicating match or mismatch, and which specific components changed if applicable.

### 9.5 Handling Missing Fields

- If the manifest declares an `annotations_digest` but the tool does not return annotations: the verification SHALL report a mismatch.
- If the tool returns annotations but the manifest does not include an `annotations_digest`: the verification SHALL report an `interface_extension` advisory (the tool has more interface components than the manifest covers).
- If the manifest is for a version that predates the tool's current version: the verification SHALL report a `version_mismatch` advisory and SHOULD attempt to fetch an updated manifest.

<br>

## 10. Artifact Integrity Verification

### 10.1 Purpose

Artifact integrity verification provides cryptographic detection of modifications to a tool's underlying code, binary, or package. This defends against supply-chain tampering (OWASP MCP03, MCP05).

### 10.2 Artifact Object

Each tool's `artifact` object in the Sealed Manifest SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | Artifact type. One of: `npm`, `pypi`, `crate`, `wasm`, `binary`, `docker`, `mcpb`, `source`. |
| `identifier` | string | REQUIRED | Package identifier (e.g., `@acme/search-files`, `acme-search-files`, OCI image reference). |
| `version` | string | REQUIRED | Package version (e.g., `1.2.4`). |
| `digest` | string | REQUIRED | Hash of the artifact content. |
| `download_url` | string | OPTIONAL | URL where the artifact can be obtained. |
| `digest_scope` | string | OPTIONAL | What the digest covers. One of: `package` (the entire package archive), `entrypoint` (the main executable file), `bundle` (a self-contained bundle). Default: `package`. |

### 10.3 Verification Procedure

Artifact integrity verification applies to scenarios where the client has access to the tool's artifact (e.g., local MCP servers installed as packages, MCPB bundles, or WASM modules). For remote MCP servers where the client does not have access to the server's code, artifact verification is limited to checking the manifest's artifact digest against registry-published digests.

For local artifacts, implementations SHALL:

1. Retrieve the artifact from the local filesystem or package cache.
2. Compute the hash of the artifact according to the `digest_scope`:
   - `package`: Hash the entire package archive file (e.g., `.tgz`, `.whl`, `.mcpb`).
   - `entrypoint`: Hash the main executable file (e.g., `index.js`, `main.py`).
   - `bundle`: Hash the self-contained bundle (e.g., MCPB `.zip`).
3. Compare the computed digest against the manifest's `artifact.digest`.
4. Return a verification result indicating match or mismatch.

For remote artifacts, implementations SHOULD:

1. Fetch the artifact digest from the registry or package manager API.
2. Compare the registry-reported digest against the manifest's `artifact.digest`.
3. Return a verification result indicating match, mismatch, or `unverifiable` (if the registry does not provide digest information).

<br>

## 11. Side-Effect Declarations

### 11.1 Purpose

Side-effect declarations provide a machine-readable description of what system resources a tool accesses, enabling permission prompts, policy enforcement, and detection of undeclared behavior (OWASP MCP02).

### 11.2 Side-Effect Object

Each tool's `side_effects` object in the Sealed Manifest SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reads` | array of strings | REQUIRED | Resource categories the tool reads from. |
| `writes` | array of strings | REQUIRED | Resource categories the tool writes to. |
| `network_egress` | boolean | REQUIRED | Whether the tool makes outbound network connections. |
| `network_endpoints` | array of strings | OPTIONAL | Specific hostnames or URL patterns the tool connects to. REQUIRED if `network_egress` is `true`. |
| `executes_code` | boolean | REQUIRED | Whether the tool executes dynamic code (e.g., eval, subprocess, shell commands). |
| `execution_context` | string | OPTIONAL | Execution environment. One of: `native`, `sandboxed-wasm`, `sandboxed-container`, `sandboxed-vm`. RECOMMENDED if `executes_code` is `true`. |
| `accesses_env_vars` | boolean | REQUIRED | Whether the tool reads environment variables. |
| `env_vars_accessed` | array of strings | OPTIONAL | Specific environment variable names the tool reads. RECOMMENDED if `accesses_env_vars` is `true`. |
| `spawns_processes` | boolean | REQUIRED | Whether the tool spawns child processes. |
| `modifies_system_state` | boolean | REQUIRED | Whether the tool modifies operating system state (e.g., registry, services, scheduled tasks). |
| `persists_data` | boolean | REQUIRED | Whether the tool stores data persistently beyond the current session. |
| `human_readable_summary` | string | OPTIONAL | A plain-language summary of what the tool does, suitable for display to end users. |

### 11.3 Resource Category Values

The values used in `reads` and `writes` arrays SHALL use the resource category vocabulary defined in Section 8.5.

### 11.4 Completeness Requirement

Publishers SHALL declare all side effects of the tool to the best of their knowledge. A side-effect declaration that omits known behavior is considered inaccurate.

Scanners (see Section 14) MAY verify side-effect declarations by static analysis, dynamic analysis, or sandboxed execution and produce attestations of the form `side_effects_verified` or `side_effects_discrepancy_found`.

### 11.5 Side-Effect Change Detection

When a tool update modifies side-effect declarations, the change classification (Section 13) SHALL indicate `side_effects_changed: true`. Clients SHOULD treat side-effect expansion (e.g., adding `network_egress: true` where it was previously `false`) as a significant change requiring user notification or approval.

<br>

## 12. Build Provenance

### 12.1 Purpose

Build provenance links a tool to its source code and build process, enabling auditors to trace from a running tool back to the source repository and commit that produced it.

### 12.2 Provenance Object

Each tool's `provenance` object in the Sealed Manifest MAY contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source_repo` | string | REQUIRED at Level 1+ | URL of the source code repository. |
| `commit` | string | REQUIRED at Level 1+ | Full commit hash. |
| `branch` | string | OPTIONAL | Branch name. |
| `build_system` | string | REQUIRED at Level 2+ | Build system identifier (e.g., `github-actions`, `gitlab-ci`, `circleci`). |
| `build_workflow` | string | OPTIONAL | Path or identifier of the build workflow. |
| `builder_id` | string | REQUIRED at Level 2+ | URI identifying the builder (per SLSA specification). |
| `build_timestamp` | string | REQUIRED at Level 2+ | ISO 8601 timestamp of the build. |
| `slsa_level` | integer | OPTIONAL | SLSA Build Level achieved (0, 1, 2, or 3). |
| `build_attestation_uri` | string | OPTIONAL | URI of the SLSA provenance attestation (e.g., Rekor transparency log entry). |
| `sbom_uri` | string | OPTIONAL | URI of the Software Bill of Materials (SPDX or CycloneDX format). |
| `reproducible` | boolean | OPTIONAL | Whether the build is reproducible (bit-for-bit identical output from the same inputs). |

### 12.3 Provenance Level Requirements

| Trust Level | Provenance Requirements |
|-------------|------------------------|
| Level 0 | No provenance required. |
| Level 1 | `source_repo` and `commit` SHALL be present. |
| Level 2 | Level 1 requirements, plus `build_system`, `builder_id`, and `build_timestamp` SHALL be present. Build provenance SHOULD be signed by the build system. |
| Level 3 | Level 2 requirements, plus `build_attestation_uri` SHALL be present and SHALL reference a verifiable SLSA provenance attestation. `sbom_uri` SHOULD be present. |

<br>

## 13. Version Lineage and Change Classification

### 13.1 Purpose

Version lineage creates a cryptographic chain across tool versions, enabling detection of broken chains (unauthorized version insertion), rug-pulls, and undisclosed changes. Change classification provides machine-readable descriptions of what changed between versions, enabling automated policy decisions about updates.

### 13.2 Lineage Object

Each tool's `lineage` object in the Sealed Manifest MAY contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `previous_version` | string | REQUIRED at Level 2+ | Version string of the previous release. |
| `previous_manifest_digest` | string | REQUIRED at Level 2+ | Digest of the complete previous Sealed Manifest. |
| `changes` | array | REQUIRED at Level 2+ | Array of change classification entries. |

### 13.3 Change Classification Entries

Each entry in the `changes` array SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | Change type. One of: `bugfix`, `feature`, `security`, `refactor`, `dependency`, `breaking`. |
| `description` | string | REQUIRED | Human-readable description of the change. |
| `interface_changed` | boolean | REQUIRED | Whether the tool's interface fingerprints changed. |
| `capabilities_changed` | boolean | REQUIRED | Whether the tool's required capabilities changed. |
| `side_effects_changed` | boolean | REQUIRED | Whether the tool's side-effect declarations changed. |

### 13.4 Chain Verification Procedure

To verify version lineage, implementations SHALL:

1. Obtain the current Sealed Manifest.
2. If `lineage.previous_version` is present, retrieve the previous Sealed Manifest (from cache or registry).
3. Compute the digest of the previous Sealed Manifest.
4. Compare the computed digest against `lineage.previous_manifest_digest`.
5. If the digests do not match, report a `chain_broken` error.
6. If the digests match, the lineage is valid. Implementations MAY recursively verify further back in the chain.

### 13.5 First Version

The first version of a tool (with no predecessor) SHALL omit the `lineage` object or include it with `previous_version` set to `null`. Implementations SHALL treat the absence of a `lineage` object on a version that is known not to be the first version as a `chain_absent` advisory.

<br>

## 14. Multi-Party Attestation

### 14.1 Purpose

Multi-party attestation enables independent parties to vouch for specific properties of a tool, providing defense in depth beyond a single publisher's signature.

### 14.2 Attestation Object

Each entry in the top-level `attestations` array SHALL contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `role` | string | REQUIRED | Attestation role. See Section 14.3. |
| `key_id` | string | REQUIRED | Identifier of the signing key used. |
| `algorithm` | string | REQUIRED | Signing algorithm identifier (e.g., `Ed25519`). |
| `signature` | string | REQUIRED | Base64url-encoded signature over the canonical manifest payload. |
| `timestamp` | string | REQUIRED | ISO 8601 timestamp of the attestation. |
| `attestation_type` | string | OPTIONAL | Specific property being attested. See Section 14.4. |
| `details` | object | OPTIONAL | Additional attestation-specific metadata. |
| `expires_at` | string | OPTIONAL | ISO 8601 timestamp after which this attestation is no longer valid. |

### 14.3 Attestation Roles

| Role | Description | What They Attest |
|------|-------------|-----------------|
| `publisher` | The tool author or authorized maintainer | "I authored this tool and this manifest accurately describes it." |
| `registry` | An MCP registry or package registry | "This publisher owns this namespace and the manifest conforms to the schema." |
| `scanner` | An automated security analysis tool | "Automated analysis found [no issues / specific issues] in this tool." |
| `auditor` | A human security reviewer or audit organization | "A human reviewer has examined this tool and verified [specific properties]." |

The `publisher` role is REQUIRED. All other roles are OPTIONAL.

Implementations SHALL accept manifests with any combination of attestation roles. Trust level evaluation (Section 15) determines which attestation combinations are sufficient for each level.

### 14.4 Attestation Types

The `attestation_type` field SHALL use one of the following values, or a custom value prefixed with a reverse-domain namespace:

| Value | Role(s) | Meaning |
|-------|---------|---------|
| `manifest_signed` | publisher | The publisher has signed this manifest. This is the default if `attestation_type` is omitted on a `publisher` attestation. |
| `namespace_verified` | registry | The registry has verified the publisher's ownership of the declared namespace. |
| `schema_validated` | registry | The registry has validated the manifest against the MCP Integrity Standard schema. |
| `no_prompt_injection_detected` | scanner | Automated scanning found no prompt injection patterns in tool descriptions or code. |
| `side_effects_verified` | scanner | Automated analysis confirmed the side-effect declarations match observed behavior. |
| `side_effects_discrepancy_found` | scanner | Automated analysis found discrepancies between declared and observed side effects. |
| `malware_scan_clean` | scanner | Automated malware scanning found no known malicious patterns. |
| `code_review_passed` | auditor | A human reviewer has reviewed the tool's code and approved it. |
| `security_audit_passed` | auditor | A security audit has been completed for this tool. |

### 14.5 Signature Scope

All attestation signatures SHALL be computed over the canonical manifest payload (the contents of the `payload` field in the DSSE envelope, after base64url decoding and before any attestation signatures are added).

This means that all attestors sign the same payload. The manifest payload SHALL NOT include the `attestations` array itself; the attestations are carried in the DSSE envelope alongside the payload.

<br>

## 15. Trust Levels

### 15.1 Purpose

Trust levels provide a graduated classification system that communicates the assurance level of a tool's integrity to users, host applications, and policy engines. Trust levels are determined by the verification results and the attestations present on a Sealed Manifest.

### 15.2 Level Definitions

#### Level 0: Unsealed

No Sealed Manifest exists for this tool.

- The tool has no integrity guarantees.
- The client may operate in TOFU mode (Section 20.2) to detect future changes.
- This is the default state for all MCP tools that have not adopted this standard.

**Requirements:** None.

#### Level 1: Publisher-Sealed

The publisher has signed a Sealed Manifest containing interface fingerprints.

**Requirements:**

| Requirement | Status |
|---|---|
| Sealed Manifest exists and is well-formed | REQUIRED |
| Publisher attestation (`role: publisher`) with valid signature | REQUIRED |
| `interface.composite_digest` present for each tool | REQUIRED |
| `side_effects` present for each tool | REQUIRED |
| `capabilities_required` present for each tool | REQUIRED |
| `provenance.source_repo` and `provenance.commit` present | REQUIRED |
| Interface fingerprints match observed `tools/list` response | REQUIRED (at verification time) |

**Defends against:** Tool poisoning (T1, T2, T3), tool spoofing (T7), manifest forgery (T8).

#### Level 2: Attested

Level 1 requirements, plus build provenance, version chaining, and registry attestation.

**Requirements:**

| Requirement | Status |
|---|---|
| All Level 1 requirements | REQUIRED |
| Registry attestation (`role: registry`, `attestation_type: namespace_verified`) | REQUIRED |
| `artifact` object with digest present for each tool | REQUIRED |
| `provenance.build_system`, `provenance.builder_id`, `provenance.build_timestamp` present | REQUIRED |
| `lineage` object with `previous_version` and `previous_manifest_digest` present (except for first version) | REQUIRED |
| `lineage.changes` array with change classification present (except for first version) | REQUIRED |
| Artifact digest matches observed artifact (for local tools) or registry-reported digest (for remote tools) | REQUIRED (at verification time) |
| Version lineage chain is valid | REQUIRED (at verification time) |

**Defends against:** All Level 1 threats, plus supply-chain tampering (T4), rug-pulls (T5), silent permission escalation (T6).

#### Level 3: Hardened

Level 2 requirements, plus independent scanner or auditor attestation, SLSA provenance, and SBOM.

**Requirements:**

| Requirement | Status |
|---|---|
| All Level 2 requirements | REQUIRED |
| At least one scanner or auditor attestation with valid signature | REQUIRED |
| `provenance.build_attestation_uri` present and pointing to a verifiable SLSA provenance attestation | REQUIRED |
| `provenance.slsa_level` of 2 or higher | REQUIRED |
| `provenance.sbom_uri` present | RECOMMENDED |
| Scanner attestation of type `side_effects_verified` or `malware_scan_clean` | RECOMMENDED |

**Defends against:** All Level 2 threats, plus compromised publishers (as the scanner/auditor provides an independent check).

### 15.3 Level Determination Procedure

Implementations SHALL determine the trust level of a tool by evaluating the requirements in order from Level 3 down to Level 0 and returning the highest level for which all REQUIRED requirements are met.

<br>

## 16. Key Discovery and Management

### 16.1 Key Discovery

Public keys for signature verification SHALL be resolved through one of the following mechanisms:

| Mechanism | Requirement | Description |
|-----------|------------|-------------|
| Registry JWKS | RECOMMENDED | The MCP Registry publishes a JWKS (JSON Web Key Set) endpoint that maps `key_id` values to public keys. |
| `.well-known/mcp-integrity-keys.json` | RECOMMENDED | The publisher hosts a JWKS document at a `.well-known` URL on a domain they control. |
| Transparency Log | OPTIONAL | The public key is recorded in a transparency log (e.g., Sigstore Rekor) and the log entry is referenced by URI. |
| Pinned Key | OPTIONAL | The client has a locally pinned public key for a known publisher, configured out of band. |

Public keys SHALL NOT be embedded in the Sealed Manifest itself. This prevents the circular trust problem where an attacker replaces both the manifest and the verification key.

### 16.2 Key Identifier Format

Key identifiers (`key_id`) SHALL be strings that uniquely identify a signing key within the scope of a publisher. The format is not prescribed but SHOULD be human-readable (e.g., `acme-signing-2025`, `acme-ed25519-prod-01`).

### 16.3 Key Rotation

Publishers SHOULD rotate signing keys at intervals not exceeding one year.

When rotating keys, publishers SHALL:

1. Generate a new key pair.
2. Publish the new public key to all configured discovery endpoints.
3. Continue to make the old public key available for verification of previously signed manifests for a period of at least 90 days after the last manifest signed with the old key.
4. Sign new manifests with the new key.

### 16.4 Key Revocation

If a signing key is compromised, the publisher SHALL:

1. Remove the compromised public key from all discovery endpoints.
2. Publish a key revocation notice at the discovery endpoint, containing the `key_id` of the revoked key and the revocation timestamp.
3. Generate a new key pair and re-seal all current manifests with the new key.
4. Notify the relevant registry or registries of the revocation.

Implementations SHALL check for key revocation before accepting a signature. If a key is revoked, all manifests signed with that key SHALL be treated as unverified until re-signed with a valid key.

### 16.5 Key Revocation Notice Format

A key revocation notice SHALL be a signed JSON document with the following structure:

```json
{
  "type": "key_revocation",
  "revoked_key_id": "acme-signing-2025",
  "revocation_timestamp": "2026-02-15T10:00:00Z",
  "reason": "key_compromise",
  "replacement_key_id": "acme-signing-2026",
  "signed_by": {
    "key_id": "acme-signing-2026",
    "algorithm": "Ed25519",
    "signature": "..."
  }
}
```

If the revocation is signed by the replacement key, implementations SHOULD accept it if the replacement key is available through a trusted discovery mechanism. If the publisher's identity can be verified through the registry, the registry MAY also publish revocation notices on behalf of the publisher.

<br>

## 17. MCP Integration

### 17.1 Extension Negotiation

The MCP Integrity Standard SHALL be negotiated as an MCP Extension during the `initialize` handshake.

**Client capabilities (sent during `initialize` request):**

```json
{
  "capabilities": {
    "extensions": {
      "mcp-integrity/v1": {
        "supported_levels": [0, 1, 2, 3],
        "policy": "warn_below_level_1"
      }
    }
  }
}
```

| Field | Type | Description |
|---|---|---|
| `supported_levels` | array of integers | Trust levels the client can evaluate. |
| `policy` | string | Client's default policy. One of: `report_only`, `warn_below_level_N`, `block_below_level_N` (where N is 0, 1, 2, or 3). |

**Server capabilities (sent during `initialize` response):**

```json
{
  "capabilities": {
    "extensions": {
      "mcp-integrity/v1": {
        "seal_level": 2,
        "manifest_url": "https://example.com/.well-known/mcp-integrity.json"
      }
    }
  }
}
```

| Field | Type | Description |
|---|---|---|
| `seal_level` | integer | The trust level the server claims to meet. Clients SHALL independently verify this claim. |
| `manifest_url` | string | URL where the Sealed Manifest can be fetched. |

### 17.2 Graceful Degradation

- If the server does not include `mcp-integrity/v1` in its capabilities: The client SHALL treat the server as Level 0 and MAY operate in TOFU mode.
- If the client does not include `mcp-integrity/v1` in its capabilities: The server SHALL operate normally with no behavior changes.
- If the server declares `mcp-integrity/v1` but the manifest URL is unreachable: The client SHALL treat the server as Level 0 and SHOULD log a `manifest_fetch_failed` event.

### 17.3 Interaction with MCP Tools Lifecycle

The MCP Integrity Standard interacts with the following MCP lifecycle events:

| MCP Event | Integrity Standard Action |
|-----------|--------------------------|
| `initialize` | Negotiate extension support; fetch manifest; perform initial verification |
| `tools/list` | Compute interface fingerprints; compare against manifest |
| `tools/call` | (OPTIONAL) Verify artifact digest for local tools; verify interface fingerprint if not already verified in current session |
| `notifications/tools/list_changed` | Re-fetch manifest; re-compute all interface fingerprints; flag any changes |

<br>

## 18. Discovery Mechanisms

### 18.1 `.well-known` Discovery

For MCP servers accessible via HTTP (Streamable HTTP transport), the Sealed Manifest SHOULD be available at:

```
GET https://<server-host>/.well-known/mcp-integrity.json
```

The response SHALL be a JSON document conforming to the DSSE envelope format (Section 7.4) containing the Sealed Manifest as the payload.

The server SHOULD set the following HTTP headers:

```
Content-Type: application/vnd.mcp-integrity.manifest+json
Cache-Control: max-age=3600
```

### 18.2 Registry Discovery

For MCP servers registered in the MCP Registry (registry.modelcontextprotocol.io or compatible registries), the Sealed Manifest SHOULD be available at the registry's API:

```
GET https://registry.modelcontextprotocol.io/v0/servers/<server-id>/integrity
```

Registries that host Sealed Manifests SHALL:

1. Validate the manifest against the schema defined in Section 26.
2. Verify the publisher attestation signature.
3. Optionally add a registry attestation (`role: registry`, `attestation_type: namespace_verified`).
4. Serve the manifest in the DSSE envelope format.

### 18.3 Local File Discovery

For MCP servers installed as local packages (npm, PyPI, etc.) or distributed as MCPB bundles, the Sealed Manifest SHOULD be located at:

- `mcp-integrity.json` in the package root directory, or
- `mcp-integrity.json` in the MCPB bundle root.

Implementations SHALL search for the local manifest before falling back to registry or `.well-known` discovery.

### 18.4 Discovery Priority

Implementations SHALL use the following priority order for manifest discovery:

1. Manifest URL provided in MCP Extension negotiation (`manifest_url` field in server capabilities).
2. Local file (`mcp-integrity.json` in package root).
3. `.well-known` endpoint on the server's HTTP host.
4. MCP Registry API.
5. TOFU mode (no manifest available; fingerprint on first use).

<br>

## 19. Verification Procedures

### 19.1 Verification Triggers

Implementations SHALL perform verification at the following points:

| Trigger | Name | Requirement | Description |
|---------|------|------------|-------------|
| V1 | Install-time | REQUIRED | When an MCP server or tool is first registered, installed, or connected by the host application. |
| V2 | List-time | REQUIRED | When `tools/list` is called or its cached result is used for the first time in a session. |
| V3 | Update-time | REQUIRED | When a `notifications/tools/list_changed` event is received, or when the server's reported version changes. |
| V4 | Session-periodic | OPTIONAL | At a configurable interval during long-running sessions. NOT RECOMMENDED for intervals shorter than 30 minutes. |
| V5 | Manual | OPTIONAL | When a user or administrator explicitly requests re-verification. |

### 19.2 Full Verification Procedure

A full verification encompasses all checks applicable to the tool's declared trust level. Implementations SHALL perform the following steps in order:

**Step 1: Manifest Retrieval**

1. Fetch the Sealed Manifest using the discovery priority order (Section 18.4).
2. If no manifest is found, classify the tool as Level 0 and proceed to TOFU mode (Section 20.2) if enabled.

**Step 2: Envelope Validation**

1. Verify the DSSE envelope is well-formed.
2. Verify `payloadType` is `application/vnd.mcp-integrity.manifest+json`.
3. Decode the payload and parse as JSON.
4. Verify `seal_version` is a supported version (currently `"1.0"`).

**Step 3: Attestation Verification**

1. For each attestation in the `attestations` array:
   a. Resolve the public key for `key_id` via key discovery (Section 16.1).
   b. Check for key revocation (Section 16.4).
   c. Verify the signature over the canonical payload using the resolved key and declared algorithm.
   d. If `expires_at` is present, verify the attestation has not expired.
2. Verify that at least one `publisher` attestation is present and has a valid signature.
3. Record which attestation roles have valid signatures.

**Step 4: Interface Fingerprint Verification**

1. For each tool in the manifest's `tools` array:
   a. Locate the corresponding tool in the `tools/list` response by matching `name`.
   b. Compute the interface fingerprints as described in Section 9.3.
   c. Compare the computed `composite_digest` against the manifest's value.
   d. If mismatch, compare individual digests to identify changed components.
2. If any tool present in `tools/list` is not present in the manifest, record an `unknown_tool` advisory.

**Step 5: Artifact Verification** (if applicable)

1. If the `artifact` object is present and the artifact is accessible:
   a. Compute the artifact digest as described in Section 10.3.
   b. Compare against the manifest's `artifact.digest`.

**Step 6: Lineage Verification** (if applicable)

1. If the `lineage` object is present:
   a. Verify the chain as described in Section 13.4.

**Step 7: Trust Level Determination**

1. Evaluate trust level as described in Section 15.3.

**Step 8: Result Assembly**

1. Assemble the verification result (Section 19.3).
2. Log the verification event (Section 20.3).
3. Return the result to the host application.

### 19.3 Verification Result

A verification result SHALL contain:

| Field | Type | Description |
|---|---|---|
| `server_name` | string | Name of the MCP server. |
| `tool_name` | string | Name of the tool (for per-tool results). |
| `trust_level` | integer | Determined trust level (0, 1, 2, or 3). |
| `claimed_level` | integer | Trust level claimed by the server in extension negotiation. |
| `interface_match` | boolean | Whether all interface fingerprints match. |
| `interface_details` | object | Per-component match results (`description_match`, `input_schema_match`, `annotations_match`). |
| `artifact_match` | boolean or null | Whether the artifact digest matches. `null` if not verifiable. |
| `chain_valid` | boolean or null | Whether the version lineage chain is valid. `null` if lineage is not present. |
| `attestations_verified` | array of strings | List of attestation roles with valid signatures. |
| `advisories` | array of objects | Non-blocking warnings (e.g., `unknown_tool`, `interface_extension`, `version_mismatch`). |
| `errors` | array of objects | Blocking errors (e.g., `signature_invalid`, `chain_broken`, `key_revoked`). |
| `verified_at` | string | ISO 8601 timestamp of the verification. |

<br>

## 20. Client Behavior and Policy Enforcement

### 20.1 Policy Modes

Host applications SHALL support the following policy modes:

| Mode | Behavior |
|------|----------|
| `report_only` | All tools are allowed. Verification results are logged and available for display but do not block execution. |
| `warn_below_level_N` | Tools below Trust Level N trigger a user-visible warning before execution. The user may choose to proceed or cancel. |
| `block_below_level_N` | Tools below Trust Level N are blocked from execution. The user cannot override without changing the policy. |

The default policy for new implementations SHOULD be `warn_below_level_1`.

### 20.2 TOFU (Trust On First Use) Mode

When no Sealed Manifest is available for a tool (Level 0), implementations SHOULD operate in TOFU mode:

1. On first encounter with the tool, compute interface fingerprints from the `tools/list` response.
2. Store the fingerprints in a local cache, associated with the server name, tool name, and the current timestamp.
3. On subsequent encounters, recompute fingerprints and compare against the cached values.
4. If the fingerprints differ from the cached values, alert the user: "This tool's interface has changed since you last used it. Description/schema/annotations were modified."
5. The user may choose to accept the new fingerprints (updating the cache) or block the tool.

TOFU mode provides integrity value without any publisher adoption. It detects rug-pulls and tool poisoning from the client side alone.

TOFU fingerprints SHALL be stored in a per-host-application cache. The cache format is implementation-defined but SHALL include at minimum: server name, tool name, composite digest, individual digests, and the timestamp of first observation.

### 20.3 Audit Logging

Implementations SHALL log verification events containing at minimum the following fields:

| Field | Type | Description |
|---|---|---|
| `event_type` | string | One of: `verification_pass`, `verification_fail`, `tofu_first_use`, `tofu_change_detected`, `manifest_fetch_failed`, `key_revoked`. |
| `timestamp` | string | ISO 8601 timestamp. |
| `server_name` | string | Name of the MCP server. |
| `tool_name` | string | Name of the tool, if applicable. |
| `trust_level` | integer | Determined trust level. |
| `interface_match` | boolean | Whether interface fingerprints matched. |
| `details` | object | Additional event-specific information. |

The storage format and retention policy for audit logs is implementation-defined and out of scope for this standard. Implementations SHOULD expose audit logs through a structured interface (API, file, or event stream) for consumption by SIEM or observability tools.

<br>

## 21. Publisher Guide

### 21.1 Generating a Signing Key

Publishers SHALL generate an Ed25519 key pair for signing Sealed Manifests.

Reference CLI invocation:

```
mcp-integrity keygen \
  --algorithm Ed25519 \
  --key-id "my-publisher-key-2026" \
  --output ./signing-key.json
```

The key file SHALL contain the private key in JWK format and SHALL be stored securely. The private key SHALL NOT be committed to source control, embedded in published packages, or transmitted over unencrypted channels.

### 21.2 Sealing a Tool

Publishers SHALL generate a Sealed Manifest after building their MCP server.

Reference CLI invocation:

```
mcp-integrity seal \
  --server ./my-mcp-server \
  --key ./signing-key.json \
  --source-repo https://github.com/my-org/my-server \
  --commit $(git rev-parse HEAD) \
  --output ./mcp-integrity.json
```

The sealing process SHALL:

1. Start the MCP server in a temporary process.
2. Call `initialize` and `tools/list` to obtain the tool interfaces.
3. Compute interface fingerprints for each tool.
4. Compute artifact digests.
5. Assemble the Sealed Manifest with publisher-provided metadata.
6. Sign the manifest with the publisher's private key.
7. Output the signed manifest in DSSE envelope format.

### 21.3 Publishing the Manifest

Publishers SHALL make the Sealed Manifest available through one or more of the discovery mechanisms defined in Section 18:

- Include `mcp-integrity.json` in the package root (for npm, PyPI, MCPB, etc.).
- Host at `/.well-known/mcp-integrity.json` on the server's domain.
- Publish to the MCP Registry or a compatible registry.

### 21.4 CI/CD Integration

Publishers SHOULD automate manifest generation in their CI/CD pipeline.

Reference GitHub Actions workflow step:

```yaml
- name: Seal MCP Server
  uses: mcp-integrity/seal-action@v1
  with:
    server-path: ./dist/server
    signing-key: ${{ secrets.MCP_INTEGRITY_SIGNING_KEY }}
    slsa-provenance: true
    publish-to-registry: true
```

### 21.5 Updating a Sealed Tool

When publishing a new version, publishers SHALL:

1. Generate a new Sealed Manifest for the updated tool.
2. Set `lineage.previous_version` to the version string of the previous release.
3. Set `lineage.previous_manifest_digest` to the digest of the previous Sealed Manifest.
4. Populate `lineage.changes` with an accurate description of what changed.
5. Accurately update `interface_changed`, `capabilities_changed`, and `side_effects_changed` flags.
6. Sign and publish the new manifest.

<br>

## 22. Conformance Requirements

### 22.1 Conformance Levels

This standard defines three conformance levels for implementations:

#### 22.1.1 Client Conformance

A conformant client implementation SHALL:

1. Parse and validate Sealed Manifests in DSSE envelope format.
2. Verify Ed25519 signatures on publisher attestations.
3. Compute interface fingerprints per Section 9.3, using SHA-256 and RFC 8785 canonicalization.
4. Perform verification at install-time (V1), list-time (V2), and update-time (V3) triggers.
5. Determine trust levels per Section 15.3.
6. Support TOFU mode for Level 0 tools.
7. Support at least the `report_only` and `warn_below_level_1` policy modes.
8. Log verification events per Section 20.3.
9. Pass all client test vectors defined in Section 27.

A conformant client implementation SHOULD:

1. Support ES256 signatures in addition to Ed25519.
2. Support all discovery mechanisms defined in Section 18.
3. Support all policy modes defined in Section 20.1.
4. Verify artifact digests for locally installed tools.
5. Verify version lineage chains.
6. Cache Sealed Manifests and TOFU fingerprints persistently across sessions.

#### 22.1.2 Publisher Conformance

A conformant publisher implementation (sealing tool) SHALL:

1. Generate Sealed Manifests conforming to the schema in Section 26.
2. Sign manifests using Ed25519 or ES256 in DSSE envelope format.
3. Compute interface fingerprints per Section 9.3.
4. Include all REQUIRED fields for the declared trust level.
5. Publish manifests through at least one discovery mechanism defined in Section 18.
6. Pass all publisher test vectors defined in Section 27.

#### 22.1.3 Registry Conformance

A conformant registry implementation SHALL:

1. Accept and store Sealed Manifests in DSSE envelope format.
2. Validate manifest schema conformance.
3. Verify publisher attestation signatures before accepting a manifest.
4. Serve manifests via API with the correct `Content-Type`.
5. Provide a JWKS endpoint for key discovery.
6. Pass all registry test vectors defined in Section 27.

### 22.2 Interoperability Requirements

- All conformant client implementations SHALL produce identical interface fingerprints for the same `tools/list` input, given the same hash algorithm.
- All conformant publisher implementations SHALL produce Sealed Manifests that are parseable by all conformant client implementations.
- Canonicalization (RFC 8785) is the critical interoperability requirement. Implementations MUST use a compliant JCS implementation and MUST NOT use custom serialization.

<br>

## 23. Error Codes

The MCP Integrity Standard defines the following error codes for use in verification results. These codes follow the JSON-RPC error code convention and use the -33xxx range to avoid conflicts with MCP's reserved ranges.

| Code | Name | Severity | Description |
|------|------|----------|-------------|
| -33001 | `MANIFEST_NOT_FOUND` | advisory | No Sealed Manifest could be found through any discovery mechanism. |
| -33002 | `MANIFEST_PARSE_ERROR` | error | The Sealed Manifest is not valid JSON or does not conform to the expected schema. |
| -33003 | `MANIFEST_VERSION_UNSUPPORTED` | error | The `seal_version` is not supported by this implementation. |
| -33010 | `SIGNATURE_INVALID` | error | A publisher attestation signature is invalid. |
| -33011 | `SIGNATURE_KEY_NOT_FOUND` | error | The public key for `key_id` could not be resolved through any discovery mechanism. |
| -33012 | `SIGNATURE_KEY_REVOKED` | error | The signing key has been revoked. |
| -33013 | `SIGNATURE_EXPIRED` | error | An attestation's `expires_at` has passed. |
| -33020 | `INTERFACE_MISMATCH` | error | The computed interface fingerprint does not match the manifest. |
| -33021 | `INTERFACE_DESCRIPTION_CHANGED` | error | The tool's description has changed. |
| -33022 | `INTERFACE_SCHEMA_CHANGED` | error | The tool's input schema has changed. |
| -33023 | `INTERFACE_ANNOTATIONS_CHANGED` | error | The tool's annotations have changed. |
| -33024 | `INTERFACE_UNKNOWN_TOOL` | advisory | A tool in `tools/list` has no corresponding entry in the manifest. |
| -33030 | `ARTIFACT_MISMATCH` | error | The computed artifact digest does not match the manifest. |
| -33031 | `ARTIFACT_UNVERIFIABLE` | advisory | The artifact could not be accessed for verification. |
| -33040 | `CHAIN_BROKEN` | error | The version lineage chain digest does not match the previous manifest. |
| -33041 | `CHAIN_ABSENT` | advisory | No lineage information is present on a non-first version. |
| -33050 | `LEVEL_CLAIMED_NOT_MET` | advisory | The server's claimed trust level is not met by verification. |
| -33060 | `MANIFEST_FETCH_FAILED` | advisory | The manifest URL was unreachable or returned a non-200 response. |

<br>

## 24. Privacy Considerations

- Sealed Manifests are public documents. They SHALL NOT contain private data, credentials, or internal infrastructure details.
- Publisher identifiers (email addresses, namespace strings) are part of the manifest and will be publicly visible. Publishers SHOULD use identifiers intended for public use.
- TOFU fingerprint caches are local to the host application and SHALL NOT be transmitted to external services without explicit user consent.
- Verification event logs MAY contain server names and tool names. Implementations SHOULD allow users to disable or restrict logging.
- Implementations SHALL NOT transmit verification results to third parties without explicit user consent, except to the extent required for key discovery (fetching public keys from a JWKS endpoint).

<br>

## 25. Security Considerations

### 25.1 Trust Assumptions

The MCP Integrity Standard makes the following trust assumptions:

- **Publisher signing keys are secure.** If a publisher's private key is compromised, the attacker can forge manifests. Key management (Section 16) and multi-party attestation (Section 14) mitigate this risk but cannot eliminate it.
- **Hash algorithms are collision-resistant.** SHA-256 is assumed to be collision-resistant. If a practical collision attack against SHA-256 is discovered, this standard will require revision.
- **Discovery endpoints are reachable.** If an attacker can prevent the client from fetching the Sealed Manifest (e.g., via DNS hijacking or network interception), the client will fall back to Level 0 / TOFU mode. Implementations SHOULD cache manifests to reduce dependence on network availability.
- **The MCP server correctly implements `tools/list`.** Interface fingerprinting depends on the `tools/list` response reflecting the actual tool interface. If the server returns a benign interface to `tools/list` but executes a different interface at `tools/call` time, this standard cannot detect the discrepancy. Runtime behavioral monitoring is needed for this case.

### 25.2 Limitations

- **Self-declared side effects.** Side-effect declarations are provided by the publisher. A malicious publisher may declare `network_egress: false` while the tool actually makes network calls. Scanner attestations (Section 14) mitigate this, but the base case relies on publisher honesty.
- **Interface fingerprinting is not prompt injection defense.** This standard detects changes to tool descriptions; it does not analyze whether a description contains malicious instructions. A publisher could sign a manifest for a tool with a poisoned description. Scanner attestations that check for prompt injection patterns address this.
- **TOFU mode has first-use vulnerability.** If a tool is poisoned before the client's first encounter, TOFU mode will cache the poisoned fingerprint as the baseline. Publisher-sealed manifests (Level 1+) are needed for protection against this scenario.

### 25.3 Recommendations

- Implementations SHOULD enforce a minimum trust level policy of `warn_below_level_1` in production environments.
- Organizations deploying MCP in high-security environments SHOULD require Level 2 or Level 3 for all tools with write access or network egress.
- Publishers SHOULD enable automated sealing in CI/CD pipelines to ensure manifests are always current and accurately reflect the published artifact.
- Registries SHOULD provide scanner attestation services or integrate with third-party scanner providers to increase the availability of Level 3 tools.

<br>

## 26. Data Structures and JSON Schemas

### 26.1 Sealed Manifest Schema

The following JSON Schema defines the Sealed Manifest format. Implementations SHALL validate manifests against this schema.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mcp-integrity.org/schemas/v1/sealed-manifest.json",
  "title": "MCP Integrity Sealed Manifest",
  "type": "object",
  "required": ["seal_version", "server", "tools", "issued_at"],
  "properties": {
    "seal_version": {
      "type": "string",
      "const": "1.0"
    },
    "server": {
      "type": "object",
      "required": ["name", "version", "publisher"],
      "properties": {
        "name": { "type": "string" },
        "version": { "type": "string" },
        "publisher": {
          "type": "object",
          "required": ["id", "namespace"],
          "properties": {
            "id": { "type": "string" },
            "namespace": { "type": "string" },
            "url": { "type": "string", "format": "uri" }
          }
        }
      }
    },
    "tools": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "version", "interface", "side_effects", "capabilities_required"],
        "properties": {
          "name": { "type": "string" },
          "version": { "type": "string" },
          "interface": {
            "type": "object",
            "required": ["description_digest", "input_schema_digest", "composite_digest"],
            "properties": {
              "description_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "input_schema_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "annotations_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "output_schema_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "composite_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" }
            }
          },
          "artifact": {
            "type": "object",
            "required": ["type", "identifier", "version", "digest"],
            "properties": {
              "type": { "type": "string", "enum": ["npm", "pypi", "crate", "wasm", "binary", "docker", "mcpb", "source"] },
              "identifier": { "type": "string" },
              "version": { "type": "string" },
              "digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "download_url": { "type": "string", "format": "uri" },
              "digest_scope": { "type": "string", "enum": ["package", "entrypoint", "bundle"], "default": "package" }
            }
          },
          "side_effects": {
            "type": "object",
            "required": ["reads", "writes", "network_egress", "executes_code", "accesses_env_vars", "spawns_processes", "modifies_system_state", "persists_data"],
            "properties": {
              "reads": { "type": "array", "items": { "type": "string" } },
              "writes": { "type": "array", "items": { "type": "string" } },
              "network_egress": { "type": "boolean" },
              "network_endpoints": { "type": "array", "items": { "type": "string" } },
              "executes_code": { "type": "boolean" },
              "execution_context": { "type": "string", "enum": ["native", "sandboxed-wasm", "sandboxed-container", "sandboxed-vm"] },
              "accesses_env_vars": { "type": "boolean" },
              "env_vars_accessed": { "type": "array", "items": { "type": "string" } },
              "spawns_processes": { "type": "boolean" },
              "modifies_system_state": { "type": "boolean" },
              "persists_data": { "type": "boolean" },
              "human_readable_summary": { "type": "string" }
            }
          },
          "capabilities_required": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["resource", "actions", "required"],
              "properties": {
                "resource": { "type": "string" },
                "actions": { "type": "array", "items": { "type": "string" } },
                "scope": { "type": "string" },
                "required": { "type": "boolean" }
              }
            }
          },
          "provenance": {
            "type": "object",
            "properties": {
              "source_repo": { "type": "string", "format": "uri" },
              "commit": { "type": "string" },
              "branch": { "type": "string" },
              "build_system": { "type": "string" },
              "build_workflow": { "type": "string" },
              "builder_id": { "type": "string" },
              "build_timestamp": { "type": "string", "format": "date-time" },
              "slsa_level": { "type": "integer", "minimum": 0, "maximum": 3 },
              "build_attestation_uri": { "type": "string", "format": "uri" },
              "sbom_uri": { "type": "string", "format": "uri" },
              "reproducible": { "type": "boolean" }
            }
          },
          "lineage": {
            "type": "object",
            "properties": {
              "previous_version": { "type": ["string", "null"] },
              "previous_manifest_digest": { "type": "string", "pattern": "^(sha256|sha512):[a-f0-9]+$" },
              "changes": {
                "type": "array",
                "items": {
                  "type": "object",
                  "required": ["type", "description", "interface_changed", "capabilities_changed", "side_effects_changed"],
                  "properties": {
                    "type": { "type": "string", "enum": ["bugfix", "feature", "security", "refactor", "dependency", "breaking"] },
                    "description": { "type": "string" },
                    "interface_changed": { "type": "boolean" },
                    "capabilities_changed": { "type": "boolean" },
                    "side_effects_changed": { "type": "boolean" }
                  }
                }
              }
            }
          }
        }
      }
    },
    "issued_at": { "type": "string", "format": "date-time" },
    "expires_at": { "type": "string", "format": "date-time" }
  }
}
```

### 26.2 DSSE Envelope Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mcp-integrity.org/schemas/v1/dsse-envelope.json",
  "title": "DSSE Envelope for MCP Integrity",
  "type": "object",
  "required": ["payloadType", "payload", "signatures"],
  "properties": {
    "payloadType": {
      "type": "string",
      "const": "application/vnd.mcp-integrity.manifest+json"
    },
    "payload": {
      "type": "string",
      "description": "Base64url-encoded canonical Sealed Manifest JSON."
    },
    "signatures": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["keyid", "sig"],
        "properties": {
          "keyid": { "type": "string" },
          "sig": { "type": "string" },
          "role": { "type": "string", "enum": ["publisher", "registry", "scanner", "auditor"] },
          "attestation_type": { "type": "string" },
          "timestamp": { "type": "string", "format": "date-time" },
          "expires_at": { "type": "string", "format": "date-time" },
          "details": { "type": "object" }
        }
      }
    }
  }
}
```

### 26.3 Key Revocation Notice Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://mcp-integrity.org/schemas/v1/key-revocation.json",
  "title": "Key Revocation Notice",
  "type": "object",
  "required": ["type", "revoked_key_id", "revocation_timestamp", "reason", "signed_by"],
  "properties": {
    "type": { "type": "string", "const": "key_revocation" },
    "revoked_key_id": { "type": "string" },
    "revocation_timestamp": { "type": "string", "format": "date-time" },
    "reason": { "type": "string", "enum": ["key_compromise", "key_rotation", "publisher_deactivation"] },
    "replacement_key_id": { "type": "string" },
    "signed_by": {
      "type": "object",
      "required": ["key_id", "algorithm", "signature"],
      "properties": {
        "key_id": { "type": "string" },
        "algorithm": { "type": "string" },
        "signature": { "type": "string" }
      }
    }
  }
}
```

<br>

## 27. Test Vectors

Conformant implementations SHALL produce results consistent with the following test vectors. Full test vector data (JSON files with inputs and expected outputs) SHALL be published alongside this standard at the canonical repository.

### 27.1 Interface Fingerprint Test Vector

**Input:**

Tool description (string):
```
Search files in a directory by keyword.
```

Tool input schema (JSON):
```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string", "description": "Search keyword" },
    "max_results": { "type": "integer", "default": 10 }
  },
  "required": ["query"]
}
```

Tool annotations (JSON):
```json
{
  "title": "Search Files",
  "readOnlyHint": true,
  "destructiveHint": false,
  "openWorldHint": false
}
```

**Procedure:**

1. `description_digest` = SHA-256 of the UTF-8 encoded description string.
2. `input_schema_digest` = SHA-256 of the JCS-canonicalized input schema.
3. `annotations_digest` = SHA-256 of the JCS-canonicalized annotations.
4. `composite_input` = `description_digest` + ":" + `input_schema_digest` + ":" + `annotations_digest` + ":"
5. `composite_digest` = SHA-256 of the UTF-8 encoded composite_input.

**Expected outputs:**

Implementations SHALL compute identical digest values for these inputs. The exact expected hex values SHALL be published in the test vector data files accompanying this standard.

### 27.2 Manifest Signature Test Vector

A test Ed25519 key pair and a signed manifest SHALL be published in the test vector data files. Conformant implementations SHALL:

1. Successfully verify the test manifest's signature using the test public key.
2. Report `SIGNATURE_INVALID` when any byte of the manifest payload is modified.
3. Report `SIGNATURE_INVALID` when the signature is modified.
4. Report `SIGNATURE_KEY_NOT_FOUND` when a different `key_id` is used.

### 27.3 Trust Level Determination Test Vectors

A set of test manifests with varying attestations and fields SHALL be published. For each test manifest, the expected trust level SHALL be specified. Conformant implementations SHALL produce the same trust level determination for each test case.

<br>

## 28. References

### Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **RFC 8174** — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- **RFC 8785** — JSON Canonicalization Scheme (JCS)
- **RFC 4648** — The Base16, Base32, and Base64 Data Encodings
- **RFC 8032** — Edwards-Curve Digital Signature Algorithm (EdDSA)
- **RFC 6979** — Deterministic Usage of the Digital Signature Algorithm (DSA) and Elliptic Curve Digital Signature Algorithm (ECDSA)
- **FIPS 180-4** — Secure Hash Standard (SHA-256, SHA-512)
- **DSSE** — Dead Simple Signing Envelope, in-toto project specification
- **JSON Schema** — JSON Schema: A Media Type for Describing JSON Documents (draft-bhutton-json-schema-01)

### Informative References

- **MCP Specification** — Model Context Protocol, version 2025-11-25
- **MCP Authorization** — MCP Authorization specification (OAuth 2.1)
- **SLSA** — Supply-chain Levels for Software Artifacts, version 1.0
- **in-toto** — in-toto Attestation Framework
- **Sigstore** — Sigstore Project (cosign, rekor, fulcio)
- **SPDX** — Software Package Data Exchange
- **CycloneDX** — CycloneDX SBOM Standard
- **RFC 8615** — Well-Known Uniform Resource Identifiers
- **RFC 7517** — JSON Web Key (JWK)
- **OWASP MCP Top 10** — OWASP Foundation, 2025 edition

<br>

## Appendix A: OWASP MCP Top 10 Coverage Matrix

| OWASP Risk | ID | MCP Integrity Standard Section(s) | Mitigation Mechanism |
|---|---|---|---|
| Excessive Permissions & Scope Creep | MCP02 | Section 8.5 (Capability Declarations), Section 11 (Side-Effect Declarations), Section 13 (Change Classification) | Machine-readable capability and side-effect declarations enable visibility into permission scope. Change classification flags capability creep across versions. |
| Tool Poisoning | MCP03 | Section 9 (Interface Fingerprinting), Section 14 (Multi-Party Attestation) | Cryptographic fingerprints detect any modification to tool descriptions, schemas, or annotations. Scanner attestations verify absence of prompt injection. |
| Insecure Tool Discovery | MCP04 | Section 8.3 (Server/Publisher Identity), Section 14 (Multi-Party Attestation), Section 16 (Key Management), Section 18 (Discovery) | Signed manifests with verified publisher namespaces prevent tool spoofing and shadowing. Registry attestations verify namespace ownership. |
| Rug Pull / Server Compromise | MCP05 | Section 13 (Version Lineage), Section 10 (Artifact Verification), Section 9 (Interface Fingerprinting) | Hash-chained version lineage detects broken chains. Interface fingerprinting and artifact digests detect unauthorized tool modifications across updates. |
| Insufficient Logging & Monitoring | MCP08 | Section 20.3 (Audit Logging), Section 19.3 (Verification Results) | Structured verification events with pass/fail results, trust levels, and digest comparisons provide auditable integrity data. |

<br>

## Appendix B: Example Sealed Manifest

The following is a complete example of a Sealed Manifest for an MCP server with two tools, at Trust Level 2.

```json
{
  "seal_version": "1.0",
  "server": {
    "name": "acme-file-tools",
    "version": "1.2.4",
    "publisher": {
      "id": "engineering@acme.example.com",
      "namespace": "io.github.acme",
      "url": "https://github.com/acme"
    }
  },
  "tools": [
    {
      "name": "searchFiles",
      "version": "1.2.4",
      "interface": {
        "description_digest": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
        "input_schema_digest": "sha256:c3ab8ff13720e8ad9047dd39466b3c8974e592c2fa383d4a3960714caef0c4f2",
        "annotations_digest": "sha256:2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae",
        "composite_digest": "sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
      },
      "artifact": {
        "type": "npm",
        "identifier": "@acme/file-tools",
        "version": "1.2.4",
        "digest": "sha256:0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5a69788796a5b4c3d2e1f0",
        "digest_scope": "package"
      },
      "side_effects": {
        "reads": ["filesystem"],
        "writes": [],
        "network_egress": false,
        "executes_code": false,
        "accesses_env_vars": false,
        "spawns_processes": false,
        "modifies_system_state": false,
        "persists_data": false,
        "human_readable_summary": "Searches files in a specified directory by keyword. Read-only access to the filesystem. No network calls, no code execution."
      },
      "capabilities_required": [
        {
          "resource": "filesystem",
          "actions": ["read"],
          "scope": "/home/*/docs",
          "required": true
        }
      ],
      "provenance": {
        "source_repo": "https://github.com/acme/file-tools",
        "commit": "abc123def456789abc123def456789abc123def4",
        "branch": "main",
        "build_system": "github-actions",
        "build_workflow": ".github/workflows/release.yml",
        "builder_id": "https://github.com/actions/runner",
        "build_timestamp": "2026-02-20T14:30:00Z",
        "slsa_level": 2
      },
      "lineage": {
        "previous_version": "1.2.3",
        "previous_manifest_digest": "sha256:aabbccdd11223344aabbccdd11223344aabbccdd11223344aabbccdd11223344",
        "changes": [
          {
            "type": "bugfix",
            "description": "Fixed pagination returning duplicate results when max_results exceeded available files.",
            "interface_changed": false,
            "capabilities_changed": false,
            "side_effects_changed": false
          }
        ]
      }
    },
    {
      "name": "writeFile",
      "version": "1.2.4",
      "interface": {
        "description_digest": "sha256:d7a8fbb307d7809469ca9abcb0082e4f8d5651e46d3cdb762d02d0bf37c9e592",
        "input_schema_digest": "sha256:ef2d127de37b942baad06145e54b0c619a1f22327b2ebbcfbec78f5564afe39d",
        "annotations_digest": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        "composite_digest": "sha256:b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2"
      },
      "artifact": {
        "type": "npm",
        "identifier": "@acme/file-tools",
        "version": "1.2.4",
        "digest": "sha256:0f1e2d3c4b5a69788796a5b4c3d2e1f00f1e2d3c4b5a69788796a5b4c3d2e1f0",
        "digest_scope": "package"
      },
      "side_effects": {
        "reads": [],
        "writes": ["filesystem"],
        "network_egress": false,
        "executes_code": false,
        "accesses_env_vars": false,
        "spawns_processes": false,
        "modifies_system_state": false,
        "persists_data": true,
        "human_readable_summary": "Writes content to a file at a specified path. Filesystem write access required. No network calls."
      },
      "capabilities_required": [
        {
          "resource": "filesystem",
          "actions": ["write"],
          "scope": "/home/*/docs",
          "required": true
        }
      ],
      "provenance": {
        "source_repo": "https://github.com/acme/file-tools",
        "commit": "abc123def456789abc123def456789abc123def4",
        "branch": "main",
        "build_system": "github-actions",
        "build_workflow": ".github/workflows/release.yml",
        "builder_id": "https://github.com/actions/runner",
        "build_timestamp": "2026-02-20T14:30:00Z",
        "slsa_level": 2
      },
      "lineage": {
        "previous_version": "1.2.3",
        "previous_manifest_digest": "sha256:aabbccdd11223344aabbccdd11223344aabbccdd11223344aabbccdd11223344",
        "changes": [
          {
            "type": "bugfix",
            "description": "Fixed file encoding issue when writing UTF-8 content with BOM.",
            "interface_changed": false,
            "capabilities_changed": false,
            "side_effects_changed": false
          }
        ]
      }
    }
  ],
  "issued_at": "2026-02-20T15:00:00Z",
  "expires_at": "2026-03-20T15:00:00Z"
}
```

The above manifest would be wrapped in a DSSE envelope with publisher and registry attestation signatures for delivery and verification.

<br>

## End MCP Integrity Standard
