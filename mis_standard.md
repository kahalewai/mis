# MCP Integrity Standard

**Status:** Public Draft for Community Review

**Date:** February 24, 2026

**Version:** v1.0.0

**License:** Apache License 2.0

**Audience:** MCP server developers, MCP client developers, MCP host application developers, security architects, AI platform builders, registry operators, security tool vendors

<br>

### Begin MCP Integrity Standard v1.0.0

<br>

## Abstract

The MCP Integrity Standard defines a specification for verifying the integrity, provenance, and behavioral declarations of tools and servers operating over the Model Context Protocol (MCP). It provides a standardized document format (the Sealed Manifest), verification procedures, discovery conventions, and graduated trust levels that enable MCP clients, hosts, and registries to determine whether a tool is what it claims to be, whether it has been tampered with, and whether it has changed without the knowledge or consent of the user.

The MCP Integrity Standard is to MCP tools what code-signing and SBOM verification are to software packages. It does not replace or duplicate MCP's transport security, authentication, or authorization mechanisms. It adds an integrity layer that those mechanisms do not provide.

<br>

## Table of Contents

1. Introduction
2. Goals, Scope, and Non-Goals
3. Normative Language
4. Terminology
5. Threat Model
6. Cryptographic Requirements
7. Control Domain 1: Sealed Manifest
8. Control Domain 2: Tool Interface Fingerprinting
9. Control Domain 3: Artifact Integrity
10. Control Domain 4: Side-Effect Declarations
11. Control Domain 5: Build Provenance
12. Control Domain 6: Version Lineage
13. Control Domain 7: Multi-Party Attestation
14. Trust Levels
15. Key Management
16. MCP Integration
17. Discovery
18. Verification
19. Client Behavior and Policy Enforcement
20. Publisher Requirements
21. Conformance Requirements
22. Error Codes
23. Privacy Considerations
24. Security Considerations
25. References

<br>

## 1. Introduction

### 1.1 Purpose

The MCP Integrity Standard defines requirements for verifying the integrity, provenance, and behavioral declarations of MCP tools and servers. It specifies a signed document format (the Sealed Manifest), discovery conventions, verification procedures, and graduated trust levels.

### 1.2 Relationship to MCP

The MCP Integrity Standard operates on top of the Model Context Protocol as an MCP Extension. This standard:

- SHALL NOT require modifications to the MCP JSON-RPC message format
- SHALL NOT require modifications to MCP transports (stdio, Streamable HTTP)
- SHALL NOT replace, duplicate, or interfere with MCP's OAuth 2.1 authorization flows
- SHALL NOT modify or intercept `tools/call` request or response payloads
- SHALL operate as metadata alongside MCP operations, not as a gatekeeper within them
- SHALL degrade gracefully when either client or server does not support it

### 1.3 Relationship to OWASP MCP Top 10

This standard directly addresses five of the ten risks in the OWASP MCP Top 10 (2025):

| OWASP Risk | Mitigation Mechanism |
|---|---|
| MCP02: Excessive Permissions & Scope Creep | Side-effect and capability declarations make permission scope visible, auditable, and enforceable |
| MCP03: Tool Poisoning | Tool interface fingerprinting detects any modification to tool descriptions, schemas, or annotations |
| MCP04: Insecure Tool Discovery | Signed manifests with verified publisher identity prevent tool spoofing, shadowing, and name collision attacks |
| MCP05: Rug Pull / Server Compromise | Version lineage with hash chaining detects and flags malicious or undisclosed updates |
| MCP08: Insufficient Logging & Monitoring | Structured verification events provide auditable integrity data |

### 1.4 Design Principles

1. **Integrity, not authorization.** This standard answers "is this tool what it claims to be?" It does not answer "is this user allowed to use it?"
2. **Graduated adoption.** Implementations SHALL support incremental adoption through defined trust levels. Partial adoption SHALL provide partial value.
3. **Publisher simplicity.** Sealing a tool SHALL require no more than a signing key and a single automated step.
4. **Client-side value without publisher adoption.** Clients SHALL be able to derive integrity value through TOFU fingerprinting even when publishers have not adopted this standard.
5. **Non-interference.** This standard SHALL NOT interfere with any other MCP capability, extension, or authorization mechanism.
6. **Fail-safe by default.** Verification results SHALL be advisory by default. Host applications MAY enforce blocking policies.

<br>

## 2. Goals, Scope, and Out of Scope

### 2.1 Goals

This standard SHALL:

1. Provide a standardized format for declaring and signing tool integrity metadata
2. Enable cryptographic verification that a tool's interface has not been modified from the publisher's declared version
3. Enable cryptographic verification that a tool's underlying artifact matches the publisher's declared digest
4. Provide machine-readable declarations of tool side effects and required capabilities
5. Enable detection of undisclosed or unauthorized tool updates through version lineage
6. Support multi-party attestation so that publishers, registries, scanners, and auditors can independently vouch for a tool
7. Define graduated trust levels with clear requirements for each level
8. Integrate with MCP as a standard extension with graceful degradation
9. Define conformance requirements sufficient for independent implementations to interoperate

### 2.2 Scope

This standard SHALL specify: the Sealed Manifest format; cryptographic algorithms; interface fingerprint computation; artifact digest computation and verification; side-effect declaration schema; build provenance metadata; version lineage and change classification; multi-party attestation; trust level definitions; key discovery; MCP extension negotiation; discovery mechanisms; verification procedures; client behavior and policy enforcement; conformance requirements; and error codes.

### 2.3 Out of Scope

This standard SHALL NOT specify: user or agent authentication or identity provisioning; authorization or access control decisions; OAuth token management; transport layer security; runtime behavioral monitoring; reputation scoring; prompt injection detection; sandbox implementation; logging retention policies; or registry implementation details beyond manifest hosting interfaces.

<br>

## 3. Normative Language

The key words "SHALL", "SHALL NOT", "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, RFC 2119, and RFC 8174 when, and only when, they appear in all capitals.

<br>

## 4. Terminology

**Sealed Manifest** — A signed JSON document published by a tool publisher that declares the tool's identity, interface fingerprints, artifact digests, capability requirements, side-effect declarations, build provenance, and version lineage.

**Tool Interface** — The combination of a tool's description, input schema, output schema (if declared), and annotations as returned by the MCP `tools/list` method.

**Interface Fingerprint** — A cryptographic hash computed over the canonicalized tool interface, used to detect modifications to tool descriptions, schemas, or annotations.

**Artifact Digest** — A cryptographic hash of the tool's underlying code, binary, or package, used to detect modifications to executable content.

**Publisher** — The entity that authors and signs a Sealed Manifest. A publisher is identified by a namespace (e.g., `io.github.acme`) and is responsible for the accuracy of the manifest.

**Attestation** — A signed statement by an independent party (registry, scanner, auditor) vouching for a specific property of a tool or its manifest.

**Trust Level** — A graduated classification (Level 0 through Level 3) representing increasing assurance about a tool's integrity, provenance, and verification status.

**Sealing** — The act of generating and signing a Sealed Manifest for a tool or set of tools.

**Verification** — The act of checking a Sealed Manifest's signature, comparing interface fingerprints and artifact digests against observed values, and evaluating trust level requirements.

**Version Lineage** — A hash-linked chain of Sealed Manifests across tool versions, enabling detection of unauthorized version insertions.

**TOFU (Trust On First Use)** — A verification mode where the client fingerprints a tool's interface on first encounter, caches the fingerprint, and alerts on subsequent changes.

**Side-Effect Declaration** — A machine-readable statement within a Sealed Manifest describing what system resources a tool reads, writes, or accesses.

**Change Classification** — A machine-readable description of what changed between tool versions, enabling automated policy decisions about updates.

<br>

## 5. Threat Model

### 5.1 Actors

This standard considers the following threat actors: malicious or compromised publishers; supply-chain attackers who compromise distribution channels; tool poisoners who modify tool descriptions or schemas; rug-pull attackers who push malicious updates after gaining adoption; and tool shadowers who impersonate legitimate tools.

### 5.2 Attack Vectors

| ID | Attack Vector | Standard Mitigation |
|---|---|---|
| T1 | Tool Description Poisoning | Interface fingerprinting (Section 8) |
| T2 | Input Schema Manipulation | Interface fingerprinting (Section 8) |
| T3 | Annotation Tampering | Interface fingerprinting (Section 8) |
| T4 | Artifact Replacement | Artifact digest verification (Section 9) |
| T5 | Rug Pull via Malicious Update | Version lineage (Section 12) |
| T6 | Silent Permission Escalation | Side-effect declarations + change classification (Sections 10, 12) |
| T7 | Tool Name Spoofing / Shadowing | Signed manifests with publisher namespace binding (Section 7) |
| T8 | Manifest Forgery | Cryptographic signatures with external key discovery (Section 15) |
| T9 | Key Substitution | Keys resolved from external trust sources, not embedded in manifests (Section 15) |
| T10 | Unauditable Tool Usage | Structured verification event logging (Section 19) |

### 5.3 Out-of-Scope Threats

The following are acknowledged but not addressed: prompt injection via tool output; behavioral drift without interface change; compromised build infrastructure beyond SLSA coverage; and social engineering of human reviewers.

<br>

## 6. Cryptographic Requirements

### 6.1 Signing Algorithms

| Algorithm | Requirement | Identifier |
|---|---|---|
| Ed25519 (EdDSA over Curve25519) | REQUIRED | `Ed25519` |
| ES256 (ECDSA over P-256) | RECOMMENDED | `ES256` |

Implementations SHALL NOT use RSA, MD5, SHA-1, DSA, or any algorithm not listed above.

### 6.2 Hash Algorithms

| Algorithm | Requirement | Identifier |
|---|---|---|
| SHA-256 | REQUIRED | `sha256` |
| SHA-512 | RECOMMENDED | `sha512` |

All digest values SHALL be represented as lowercase hexadecimal strings prefixed with the algorithm identifier and a colon (e.g., `sha256:0f1e2d3c...`).

### 6.3 Canonicalization

Before hashing, all JSON inputs SHALL be canonicalized according to RFC 8785 (JSON Canonicalization Scheme, JCS). Implementations MUST parse the input JSON, serialize according to RFC 8785, encode as UTF-8, and apply the hash algorithm to the resulting byte sequence.

### 6.4 Signature Format

Signatures SHALL be encoded as base64url strings (RFC 4648 §5) without padding. Signed payloads SHALL use the DSSE (Dead Simple Signing Envelope) format:

```
payloadType: "application/vnd.mcp-integrity.manifest+json"
payload: <base64url-encoded canonical manifest>
signatures: [ { keyid, sig } ]
```

<br>

## 7. Control Domain 1: Sealed Manifest

### 7.1 Purpose

The Sealed Manifest is the core data structure of this standard. It is a signed JSON document that declares integrity metadata for one or more tools published by a single MCP server.

### 7.2 Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `seal_version` | string | REQUIRED | Version of this standard. SHALL be `"1.0"` for this version. |
| `server` | object | REQUIRED | Server identity and metadata. |
| `tools` | array | REQUIRED | Array of tool integrity declarations. |
| `attestations` | array | REQUIRED | Array of one or more attestations (signatures). |
| `issued_at` | string | REQUIRED | ISO 8601 timestamp of manifest creation. |
| `expires_at` | string | OPTIONAL | ISO 8601 timestamp after which this manifest SHOULD be re-fetched. |

### 7.3 Server Object

| Field | Type | Required | Description |
|---|---|---|---|
| `server.name` | string | REQUIRED | Server name as registered in `serverInfo.name`. |
| `server.version` | string | REQUIRED | Server version as registered in `serverInfo.version`. |
| `publisher.id` | string | REQUIRED | Publisher identifier (e.g., email or domain). |
| `publisher.namespace` | string | REQUIRED | Registry namespace (e.g., `io.github.acme`). |
| `publisher.url` | string | OPTIONAL | Publisher's website or profile URL. |

### 7.4 Tool Object

Each entry in the `tools` array SHALL contain:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | REQUIRED | Tool name as returned by `tools/list`. |
| `version` | string | REQUIRED | Tool version. |
| `interface` | object | REQUIRED | Interface fingerprints. See Section 8. |
| `artifact` | object | OPTIONAL | Artifact integrity information. See Section 9. |
| `side_effects` | object | REQUIRED | Side-effect declarations. See Section 10. |
| `capabilities_required` | array | REQUIRED | Required and optional capabilities. See Section 7.5. |
| `provenance` | object | OPTIONAL | Build provenance metadata. See Section 11. |
| `lineage` | object | OPTIONAL | Version lineage information. See Section 12. |

### 7.5 Capability Declarations

Each entry in the `capabilities_required` array SHALL contain:

| Field | Type | Required | Description |
|---|---|---|---|
| `resource` | string | REQUIRED | Resource category. |
| `actions` | array | REQUIRED | Permitted actions on the resource. |
| `scope` | string | OPTIONAL | Restriction on the resource (e.g., a path pattern, hostname, or variable name). |
| `required` | boolean | REQUIRED | Whether this capability is required for basic tool operation. |

The following resource categories are defined by this standard:

| Category | Description |
|---|---|
| `filesystem` | File system read or write access |
| `network` | Network communication |
| `environment` | Environment variable access |
| `database` | Database read or write access |
| `clipboard` | System clipboard access |
| `process` | Ability to spawn child processes |
| `system` | Operating system state modification |

Custom categories SHOULD be prefixed with a reverse-domain namespace.

<br>

## 8. Control Domain 2: Tool Interface Fingerprinting

### 8.1 Purpose

Tool interface fingerprinting provides cryptographic detection of modifications to a tool's description, input schema, output schema, and annotations. This is the primary defense against tool poisoning (OWASP MCP03).

### 8.2 Interface Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `description_digest` | string | REQUIRED | Hash of the canonicalized tool description string. |
| `input_schema_digest` | string | REQUIRED | Hash of the canonicalized JSON input schema object. |
| `annotations_digest` | string | OPTIONAL | Hash of the canonicalized JSON annotations object. REQUIRED if the tool declares annotations. |
| `output_schema_digest` | string | OPTIONAL | Hash of the canonicalized JSON output schema object. |
| `composite_digest` | string | REQUIRED | Hash of the concatenation of all individual digests. Used as the single value for quick verification. |

### 8.3 Fingerprint Requirements

Implementations SHALL compute individual digests by hashing each interface component according to Section 6.3. The description string SHALL be hashed as raw UTF-8, not as a JSON value. The composite digest SHALL be computed by hashing the colon-delimited concatenation of all individual digests in the order listed above.

### 8.4 Verification Requirements

Implementations SHALL compare the computed `composite_digest` against the manifest value. On mismatch, individual digests SHALL be compared to identify which component changed. Verification results SHALL indicate match or mismatch per component.

### 8.5 Handling Missing Fields

- If the manifest declares an `annotations_digest` but the tool returns no annotations: the verification SHALL report a mismatch.
- If the tool returns annotations but the manifest declares no `annotations_digest`: the verification SHALL report an `interface_extension` advisory.
- If the manifest version predates the tool's current version: the verification SHALL report a `version_mismatch` advisory.

<br>

## 9. Control Domain 3: Artifact Integrity

### 9.1 Purpose

Artifact integrity verification provides cryptographic detection of modifications to a tool's underlying code, binary, or package, defending against supply-chain tampering (OWASP MCP03, MCP05).

### 9.2 Artifact Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | REQUIRED | Artifact type. One of: `npm`, `pypi`, `crate`, `wasm`, `binary`, `docker`, `mcpb`, `source`. |
| `identifier` | string | REQUIRED | Package identifier. |
| `version` | string | REQUIRED | Package version. |
| `digest` | string | REQUIRED | Hash of the artifact content. |
| `download_url` | string | OPTIONAL | URL where the artifact can be obtained. |
| `digest_scope` | string | OPTIONAL | Scope of the digest: `package`, `entrypoint`, or `bundle`. Default: `package`. |

### 9.3 Verification Requirements

For local artifacts, implementations SHALL compute the artifact hash according to `digest_scope` and compare against the manifest value. For remote artifacts, implementations SHOULD compare the registry-reported digest against the manifest value and report `artifact_unverifiable` if the registry does not provide digest information.

<br>

## 10. Control Domain 4: Side-Effect Declarations

### 10.1 Purpose

Side-effect declarations provide machine-readable descriptions of what system resources a tool accesses, enabling permission prompts, policy enforcement, and detection of undeclared behavior (OWASP MCP02).

### 10.2 Side-Effect Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `reads` | array of strings | REQUIRED | Resource categories the tool reads from. |
| `writes` | array of strings | REQUIRED | Resource categories the tool writes to. |
| `network_egress` | boolean | REQUIRED | Whether the tool makes outbound network connections. |
| `network_endpoints` | array of strings | OPTIONAL | Specific hostnames or URL patterns. REQUIRED if `network_egress` is `true`. |
| `executes_code` | boolean | REQUIRED | Whether the tool executes dynamic code. |
| `execution_context` | string | OPTIONAL | One of: `native`, `sandboxed-wasm`, `sandboxed-container`, `sandboxed-vm`. RECOMMENDED if `executes_code` is `true`. |
| `accesses_env_vars` | boolean | REQUIRED | Whether the tool reads environment variables. |
| `env_vars_accessed` | array of strings | OPTIONAL | Specific variable names accessed. RECOMMENDED if `accesses_env_vars` is `true`. |
| `spawns_processes` | boolean | REQUIRED | Whether the tool spawns child processes. |
| `modifies_system_state` | boolean | REQUIRED | Whether the tool modifies operating system state. |
| `persists_data` | boolean | REQUIRED | Whether the tool stores data persistently beyond the current session. |
| `human_readable_summary` | string | OPTIONAL | A plain-language summary suitable for display to end users. |

### 10.3 Completeness Requirement

Publishers SHALL declare all known side effects. A declaration that omits known behavior is considered inaccurate. Clients SHOULD treat side-effect expansion across versions as a significant change requiring user notification.

<br>

## 11. Control Domain 5: Build Provenance

### 11.1 Purpose

Build provenance links a tool to its source code and build process, enabling auditors to trace from a running tool back to the source repository and commit that produced it.

### 11.2 Provenance Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `source_repo` | string | REQUIRED at Level 1+ | URL of the source code repository. |
| `commit` | string | REQUIRED at Level 1+ | Full commit hash. |
| `branch` | string | OPTIONAL | Branch name. |
| `build_system` | string | REQUIRED at Level 2+ | Build system identifier (e.g., `github-actions`). |
| `build_workflow` | string | OPTIONAL | Path or identifier of the build workflow. |
| `builder_id` | string | REQUIRED at Level 2+ | URI identifying the builder per SLSA specification. |
| `build_timestamp` | string | REQUIRED at Level 2+ | ISO 8601 timestamp of the build. |
| `slsa_level` | integer | OPTIONAL | SLSA Build Level achieved (0–3). |
| `build_attestation_uri` | string | OPTIONAL | URI of the SLSA provenance attestation. |
| `sbom_uri` | string | OPTIONAL | URI of the Software Bill of Materials (SPDX or CycloneDX). |
| `reproducible` | boolean | OPTIONAL | Whether the build is reproducible. |

### 11.3 Provenance Level Requirements

| Trust Level | Requirements |
|---|---|
| Level 0 | No provenance required. |
| Level 1 | `source_repo` and `commit` SHALL be present. |
| Level 2 | Level 1 requirements plus `build_system`, `builder_id`, and `build_timestamp`. Build provenance SHOULD be signed by the build system. |
| Level 3 | Level 2 requirements plus `build_attestation_uri` referencing a verifiable SLSA attestation. `sbom_uri` SHOULD be present. |

<br>

## 12. Control Domain 6: Version Lineage

### 12.1 Purpose

Version lineage creates a cryptographic chain across tool versions, enabling detection of broken chains, unauthorized version insertion, and undisclosed changes.

### 12.2 Lineage Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `previous_version` | string | REQUIRED at Level 2+ | Version string of the previous release. |
| `previous_manifest_digest` | string | REQUIRED at Level 2+ | Digest of the complete previous Sealed Manifest. |
| `changes` | array | REQUIRED at Level 2+ | Array of change classification entries. |

### 12.3 Change Classification Fields

Each entry in the `changes` array SHALL contain:

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | REQUIRED | One of: `bugfix`, `feature`, `security`, `refactor`, `dependency`, `breaking`. |
| `description` | string | REQUIRED | Human-readable description of the change. |
| `interface_changed` | boolean | REQUIRED | Whether the tool's interface fingerprints changed. |
| `capabilities_changed` | boolean | REQUIRED | Whether the tool's required capabilities changed. |
| `side_effects_changed` | boolean | REQUIRED | Whether the tool's side-effect declarations changed. |

### 12.4 Chain Integrity Requirements

Implementations SHALL compute the digest of the previous Sealed Manifest and compare it against `lineage.previous_manifest_digest`. A mismatch SHALL be reported as a `chain_broken` error. The first version of a tool SHALL omit the `lineage` object or set `previous_version` to `null`. The absence of a `lineage` object on a known non-first version SHALL be reported as a `chain_absent` advisory.

<br>

## 13. Control Domain 7: Multi-Party Attestation

### 13.1 Purpose

Multi-party attestation enables independent parties to vouch for specific properties of a tool, providing defense in depth beyond a single publisher's signature.

### 13.2 Attestation Object Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `role` | string | REQUIRED | Attestation role. See Section 13.3. |
| `key_id` | string | REQUIRED | Identifier of the signing key used. |
| `algorithm` | string | REQUIRED | Signing algorithm identifier. |
| `signature` | string | REQUIRED | Base64url-encoded signature over the canonical manifest payload. |
| `timestamp` | string | REQUIRED | ISO 8601 timestamp of the attestation. |
| `attestation_type` | string | OPTIONAL | Specific property being attested. See Section 13.4. |
| `details` | object | OPTIONAL | Additional attestation-specific metadata. |
| `expires_at` | string | OPTIONAL | ISO 8601 timestamp after which this attestation is no longer valid. |

### 13.3 Attestation Roles

| Role | Description |
|---|---|
| `publisher` | The tool author or authorized maintainer. REQUIRED on all manifests. |
| `registry` | An MCP registry or package registry. |
| `scanner` | An automated security analysis tool. |
| `auditor` | A human security reviewer or audit organization. |

### 13.4 Attestation Types

| Value | Role(s) | Meaning |
|---|---|---|
| `manifest_signed` | publisher | Publisher has signed this manifest. Default if omitted on a publisher attestation. |
| `namespace_verified` | registry | Registry has verified publisher ownership of the declared namespace. |
| `schema_validated` | registry | Registry has validated the manifest against this standard's schema. |
| `no_prompt_injection_detected` | scanner | Automated scanning found no prompt injection patterns. |
| `side_effects_verified` | scanner | Analysis confirmed side-effect declarations match observed behavior. |
| `side_effects_discrepancy_found` | scanner | Analysis found discrepancies between declared and observed side effects. |
| `malware_scan_clean` | scanner | Automated malware scanning found no known malicious patterns. |
| `code_review_passed` | auditor | A human reviewer has reviewed and approved the tool's code. |
| `security_audit_passed` | auditor | A security audit has been completed for this tool. |

### 13.5 Signature Scope

All attestation signatures SHALL be computed over the canonical manifest payload. The manifest payload SHALL NOT include the `attestations` array itself; attestations are carried in the DSSE envelope alongside the payload. All attestors sign the same payload.

<br>

## 14. Trust Levels

### 14.1 Purpose

Trust levels communicate the assurance level of a tool's integrity to users, host applications, and policy engines.

### 14.2 Level 0: Unsealed

No Sealed Manifest exists for this tool. No integrity guarantees. The client MAY operate in TOFU mode to detect future changes.

### 14.3 Level 1: Publisher-Sealed

The publisher has signed a Sealed Manifest containing interface fingerprints.

| Requirement | Status |
|---|---|
| Well-formed Sealed Manifest | REQUIRED |
| Publisher attestation with valid signature | REQUIRED |
| `interface.composite_digest` for each tool | REQUIRED |
| `side_effects` for each tool | REQUIRED |
| `capabilities_required` for each tool | REQUIRED |
| `provenance.source_repo` and `provenance.commit` | REQUIRED |
| Interface fingerprints match `tools/list` response at verification time | REQUIRED |

Defends against: T1, T2, T3, T7, T8.

### 14.4 Level 2: Attested

All Level 1 requirements, plus build provenance, version chaining, and registry attestation.

| Requirement | Status |
|---|---|
| All Level 1 requirements | REQUIRED |
| Registry attestation with `namespace_verified` type | REQUIRED |
| `artifact` object with digest for each tool | REQUIRED |
| `provenance.build_system`, `builder_id`, `build_timestamp` | REQUIRED |
| `lineage` object with `previous_version` and `previous_manifest_digest` | REQUIRED (except first version) |
| `lineage.changes` array | REQUIRED (except first version) |
| Artifact digest matches artifact at verification time | REQUIRED |
| Version lineage chain is valid at verification time | REQUIRED |

Defends against: All Level 1 threats plus T4, T5, T6.

### 14.5 Level 3: Hardened

All Level 2 requirements, plus independent scanner or auditor attestation and SLSA provenance.

| Requirement | Status |
|---|---|
| All Level 2 requirements | REQUIRED |
| At least one scanner or auditor attestation with valid signature | REQUIRED |
| `provenance.build_attestation_uri` pointing to verifiable SLSA attestation | REQUIRED |
| `provenance.slsa_level` of 2 or higher | REQUIRED |
| `provenance.sbom_uri` | RECOMMENDED |
| Scanner attestation of type `side_effects_verified` or `malware_scan_clean` | RECOMMENDED |

Defends against: All Level 2 threats plus T4 and T5 from compromised publishers.

### 14.6 Level Determination

Implementations SHALL evaluate trust level requirements from Level 3 down to Level 0 and return the highest level for which all REQUIRED requirements are met.

<br>

## 15. Key Management

### 15.1 Key Discovery

Public keys for signature verification SHALL be resolved through one of the following mechanisms:

| Mechanism | Requirement |
|---|---|
| Registry JWKS endpoint | RECOMMENDED |
| `.well-known/mcp-integrity-keys.json` hosted by the publisher | RECOMMENDED |
| Transparency Log (e.g., Sigstore Rekor) | OPTIONAL |
| Locally pinned public key (configured out of band) | OPTIONAL |

Public keys SHALL NOT be embedded in the Sealed Manifest itself.

### 15.2 Key Identifier Format

Key identifiers (`key_id`) SHALL uniquely identify a signing key within the scope of a publisher and SHOULD be human-readable (e.g., `acme-ed25519-prod-01`).

### 15.3 Key Rotation

Publishers SHOULD rotate signing keys at intervals not exceeding one year. When rotating, publishers SHALL maintain the old public key at discovery endpoints for at least 90 days after the last manifest signed with it.

### 15.4 Key Revocation

If a signing key is compromised, the publisher SHALL remove the compromised key from all discovery endpoints, publish a signed key revocation notice, generate a new key pair, re-seal all current manifests, and notify relevant registries. Implementations SHALL check for key revocation before accepting a signature. Manifests signed with revoked keys SHALL be treated as unverified.

### 15.5 Key Revocation Notice

A key revocation notice SHALL be a signed JSON document containing: the type identifier `"key_revocation"`; the revoked `key_id`; revocation timestamp; reason; replacement `key_id`; and a signature from the replacement key.

<br>

## 16. MCP Integration

### 16.1 Extension Negotiation

This standard SHALL be negotiated as an MCP Extension during the `initialize` handshake.

**Client capabilities:**

| Field | Type | Description |
|---|---|---|
| `supported_levels` | array of integers | Trust levels the client can evaluate. |
| `policy` | string | Client's default policy: `report_only`, `warn_below_level_N`, or `block_below_level_N`. |

**Server capabilities:**

| Field | Type | Description |
|---|---|---|
| `seal_level` | integer | The trust level the server claims to meet. Clients SHALL independently verify this claim. |
| `manifest_url` | string | URL where the Sealed Manifest can be fetched. |

### 16.2 Graceful Degradation

- If the server does not declare `mcp-integrity/v1`: the client SHALL treat the server as Level 0 and MAY operate in TOFU mode.
- If the client does not declare `mcp-integrity/v1`: the server SHALL operate normally with no behavior changes.
- If the manifest URL is unreachable: the client SHALL treat the server as Level 0 and SHOULD log a `manifest_fetch_failed` event.

### 16.3 Lifecycle Events

| MCP Event | Required Action |
|---|---|
| `initialize` | Negotiate extension; fetch manifest; perform initial verification. |
| `tools/list` | Compute interface fingerprints; compare against manifest. |
| `tools/call` | OPTIONAL: verify artifact digest for local tools if not verified in current session. |
| `notifications/tools/list_changed` | Re-fetch manifest; recompute all interface fingerprints; flag any changes. |

<br>

## 17. Discovery

### 17.1 Discovery Priority

Implementations SHALL use the following priority order:

1. Manifest URL provided via MCP extension negotiation (`manifest_url`)
2. Local file (`mcp-integrity.json` in package root)
3. `.well-known` endpoint on the server's HTTP host
4. MCP Registry API
5. TOFU mode (no manifest available)

### 17.2 `.well-known` Endpoint

For MCP servers accessible via Streamable HTTP, the Sealed Manifest SHOULD be available at:

```
GET https://<server-host>/.well-known/mcp-integrity.json
Content-Type: application/vnd.mcp-integrity.manifest+json
```

### 17.3 Registry Discovery

For servers registered in the MCP Registry, the Sealed Manifest SHOULD be available at the registry's API. Registries that host Sealed Manifests SHALL validate the manifest schema, verify the publisher attestation signature, and serve manifests in DSSE envelope format.

### 17.4 Local File Discovery

For locally installed servers (npm, PyPI, MCPB bundles), the Sealed Manifest SHOULD be located at `mcp-integrity.json` in the package or bundle root. Implementations SHALL check the local path before falling back to remote discovery.

<br>

## 18. Verification

### 18.1 Verification Triggers

| Trigger | Name | Requirement |
|---|---|---|
| V1 | Install-time | REQUIRED — when an MCP server is first registered, installed, or connected |
| V2 | List-time | REQUIRED — when `tools/list` is called or its cached result is first used in a session |
| V3 | Update-time | REQUIRED — when `notifications/tools/list_changed` is received or server version changes |
| V4 | Session-periodic | OPTIONAL — at configurable intervals; NOT RECOMMENDED for intervals shorter than 30 minutes |
| V5 | Manual | OPTIONAL — when explicitly requested by a user or administrator |

### 18.2 Verification Steps

A full verification SHALL proceed in the following order:

1. **Manifest Retrieval** — Fetch the manifest per Section 17.1. If none is found, classify as Level 0 and enter TOFU mode.
2. **Envelope Validation** — Verify DSSE envelope structure; verify `payloadType`; parse payload; verify `seal_version`.
3. **Attestation Verification** — For each attestation: resolve key; check revocation; verify signature; check expiry. At least one valid `publisher` attestation MUST be present.
4. **Interface Fingerprint Verification** — For each tool in the manifest, locate the tool in `tools/list`, compute fingerprints, compare against manifest values.
5. **Artifact Verification** (if applicable) — Compute artifact digest per Section 9.3 and compare against manifest.
6. **Lineage Verification** (if applicable) — Verify the hash chain per Section 12.4.
7. **Trust Level Determination** — Evaluate per Section 14.6.
8. **Result Assembly and Logging** — Assemble the verification result; log the verification event.

### 18.3 Verification Result Fields

| Field | Type | Description |
|---|---|---|
| `server_name` | string | Name of the MCP server. |
| `tool_name` | string | Name of the tool. |
| `trust_level` | integer | Determined trust level (0–3). |
| `claimed_level` | integer | Trust level claimed by the server. |
| `interface_match` | boolean | Whether all interface fingerprints match. |
| `interface_details` | object | Per-component match results. |
| `artifact_match` | boolean or null | Whether the artifact digest matches. `null` if not verifiable. |
| `chain_valid` | boolean or null | Whether the version lineage chain is valid. `null` if lineage is absent. |
| `attestations_verified` | array of strings | Attestation roles with valid signatures. |
| `advisories` | array | Non-blocking warnings. |
| `errors` | array | Blocking errors. |
| `verified_at` | string | ISO 8601 timestamp of the verification. |

<br>

## 19. Client Behavior and Policy Enforcement

### 19.1 Policy Modes

Host applications SHALL support the following policy modes:

| Mode | Behavior |
|---|---|
| `report_only` | All tools are allowed. Verification results are logged but do not block execution. |
| `warn_below_level_N` | Tools below Trust Level N trigger a user-visible warning before execution. |
| `block_below_level_N` | Tools below Trust Level N are blocked from execution without policy change. |

The default policy for new implementations SHOULD be `warn_below_level_1`.

### 19.2 TOFU Mode

When no Sealed Manifest is available (Level 0), implementations SHOULD:

1. On first encounter, compute interface fingerprints from `tools/list` and store them in a local cache with a timestamp.
2. On subsequent encounters, recompute fingerprints and compare against the cached values.
3. On mismatch, alert the user that the tool's interface has changed and identify which component(s) changed.
4. Allow the user to accept the new fingerprints (updating the cache) or block the tool.

TOFU fingerprints SHALL be stored in a per-host-application cache including at minimum: server name, tool name, composite digest, individual digests, and timestamp of first observation.

### 19.3 Audit Logging

Implementations SHALL log verification events containing at minimum:

| Field | Type | Description |
|---|---|---|
| `event_type` | string | One of: `verification_pass`, `verification_fail`, `tofu_first_use`, `tofu_change_detected`, `manifest_fetch_failed`, `key_revoked`. |
| `timestamp` | string | ISO 8601 timestamp. |
| `server_name` | string | Name of the MCP server. |
| `tool_name` | string | Name of the tool, if applicable. |
| `trust_level` | integer | Determined trust level. |
| `interface_match` | boolean | Whether interface fingerprints matched. |
| `details` | object | Additional event-specific information. |

<br>

## 20. Publisher Requirements

Publishers SHALL:

1. Generate an Ed25519 or ES256 key pair for signing Sealed Manifests. Private keys SHALL NOT be committed to source control, embedded in published packages, or transmitted over unencrypted channels.
2. Generate a Sealed Manifest after each build, including all REQUIRED fields for the declared trust level.
3. Accurately declare all known side effects and capabilities.
4. Publish the manifest through at least one discovery mechanism defined in Section 17.
5. When updating a tool, accurately populate `lineage.previous_version`, `lineage.previous_manifest_digest`, `lineage.changes`, and the `interface_changed`, `capabilities_changed`, and `side_effects_changed` flags.
6. Automate manifest generation in CI/CD pipelines to ensure manifests remain current.

<br>

## 21. Conformance Requirements

### 21.1 Client Conformance

A conformant client implementation SHALL:

1. Parse and validate Sealed Manifests in DSSE envelope format
2. Verify Ed25519 signatures on publisher attestations
3. Compute interface fingerprints using SHA-256 and RFC 8785 canonicalization per Section 8.3
4. Perform verification at V1, V2, and V3 triggers
5. Determine trust levels per Section 14.6
6. Support TOFU mode for Level 0 tools
7. Support at least `report_only` and `warn_below_level_1` policy modes
8. Log verification events per Section 19.3
9. Produce identical interface fingerprints for identical `tools/list` inputs as all other conformant implementations

### 21.2 Publisher Conformance

A conformant publisher (sealing tool) SHALL:

1. Generate Sealed Manifests conforming to the schema defined in the MCP Integrity Standard Supplemental
2. Sign manifests using Ed25519 or ES256 in DSSE envelope format
3. Compute interface fingerprints per Section 8.3
4. Include all REQUIRED fields for the declared trust level
5. Publish manifests through at least one discovery mechanism defined in Section 17

### 21.3 Registry Conformance

A conformant registry implementation SHALL:

1. Accept and store Sealed Manifests in DSSE envelope format
2. Validate manifest schema conformance
3. Verify publisher attestation signatures before accepting a manifest
4. Serve manifests with the correct `Content-Type` header
5. Provide a JWKS endpoint for key discovery

### 21.4 Interoperability Requirement

All conformant client implementations SHALL produce identical interface fingerprints for the same `tools/list` input, given the same hash algorithm. RFC 8785 canonicalization is the critical interoperability requirement. Implementations MUST use a compliant JCS library and MUST NOT use custom serialization.

<br>

## 22. Error Codes

Error codes use the `-33xxx` range to avoid conflicts with MCP's reserved ranges.

| Code | Name | Severity | Description |
|---|---|---|---|
| -33001 | `MANIFEST_NOT_FOUND` | advisory | No Sealed Manifest found through any discovery mechanism. |
| -33002 | `MANIFEST_PARSE_ERROR` | error | The manifest is not valid JSON or does not conform to the schema. |
| -33003 | `MANIFEST_VERSION_UNSUPPORTED` | error | The `seal_version` is not supported by this implementation. |
| -33010 | `SIGNATURE_INVALID` | error | A publisher attestation signature is invalid. |
| -33011 | `SIGNATURE_KEY_NOT_FOUND` | error | The public key for `key_id` could not be resolved. |
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

---

## 23. Privacy Considerations

- Sealed Manifests are public documents and SHALL NOT contain private data, credentials, or internal infrastructure details.
- Publisher identifiers are publicly visible. Publishers SHOULD use identifiers intended for public use.
- TOFU fingerprint caches SHALL NOT be transmitted to external services without explicit user consent.
- Verification event logs MAY contain server names and tool names. Implementations SHOULD allow users to restrict logging.
- Implementations SHALL NOT transmit verification results to third parties without explicit user consent, except as required for key discovery.

<br>

## 24. Security Considerations

### 24.1 Trust Assumptions

- **Publisher signing keys are secure.** Compromised keys can produce forged manifests. Multi-party attestation reduces but does not eliminate this risk.
- **Hash algorithms are collision-resistant.** SHA-256 is assumed collision-resistant for the purposes of this standard.
- **Discovery endpoints are reachable.** If manifest discovery is prevented, the client falls back to TOFU mode. Implementations SHOULD cache manifests to reduce network dependence.
- **`tools/list` reflects the actual interface.** Interface fingerprinting depends on the `tools/list` response being accurate. A server that returns a benign interface at list time but a different interface at call time cannot be detected by this standard alone.

### 24.2 Limitations

- **Self-declared side effects.** Side-effect declarations rely on publisher accuracy. Scanner attestations provide independent verification but are optional at lower trust levels.
- **Interface fingerprinting is not prompt injection defense.** This standard detects changes to tool descriptions; it does not evaluate whether a description contains malicious instructions.
- **TOFU first-use vulnerability.** A tool that is poisoned before first client encounter will have its poisoned fingerprint cached as baseline. Publisher-sealed manifests (Level 1+) address this.

### 24.3 Recommendations

- Implementations SHOULD enforce `warn_below_level_1` as the minimum policy in production environments.
- Organizations deploying MCP in high-security environments SHOULD require Level 2 or Level 3 for all tools with write access or network egress.
- Publishers SHOULD integrate sealing into CI/CD pipelines to ensure manifests are always current.
- Registries SHOULD provide or integrate scanner attestation services to increase Level 3 availability.

<br>

## 25. References

### Normative References

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels
- **RFC 8174** — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words
- **RFC 8785** — JSON Canonicalization Scheme (JCS)
- **RFC 4648** — The Base16, Base32, and Base64 Data Encodings
- **RFC 8032** — Edwards-Curve Digital Signature Algorithm (EdDSA)
- **RFC 6979** — Deterministic Usage of DSA and ECDSA
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

### End MCP Integrity Standard v1.0.0
