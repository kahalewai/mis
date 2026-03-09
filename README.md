<div align="center">

# MCP Integrity Standard

Welcome to the **MCP Integrity Standard (MIS)** Security Specification Landing

</div>

<br>

## Intro

As MCP-based tools move into production and assume critical roles in AI systems, they introduce new and poorly understood integrity risks that extend far beyond transport security and authorization. Tool descriptions, input schemas, annotations, artifact code, side effects, and version updates can all silently change — often without the knowledge of the user, host application, or downstream system. Existing MCP security mechanisms were not designed to detect these changes or bind a tool to the publisher who created it. MCP Integrity Standard (MIS) is a vendor-neutral security standard that defines minimum, enforceable requirements for MCP tool integrity, provenance, behavioral transparency, and continuous verification.

MIS establishes a clear, cryptographically-anchored trust model for MCP tools by requiring:

* Explicit declaration of all tool interface fingerprints, side effects, and capabilities
* Cryptographically verifiable integrity and publisher authenticity guarantees
* Continuous verification at install-time, list-time, and update-time

The goal of MIS is to make MCP tools inspectable, verifiable, and governable throughout their entire lifecycle.

<br>

## MIS Control Model

MIS defines a deterministic security and governance model for MCP tool integrity. Rather than relying on transport-layer trust alone, MIS enforces trust through signed declarations and machine-verifiable fingerprints.

The standard is organized around seven control domains, all carried in a single signed document called the **Sealed Manifest**:

* Tool Interface Fingerprinting
* Artifact Integrity
* Side-Effect Declarations
* Build Provenance
* Version Lineage
* Multi-Party Attestation
* Capability Declarations

Together, these domains ensure that every MCP tool influencing AI system behavior is declared, fingerprinted, verified, and continuously enforced.

<br>

| Control Domain | Name | Responsibility | Security Guarantee Provided |
|---|---|---|---|
| 1 | Tool Interface Fingerprinting | Cryptographic fingerprints of tool descriptions, schemas, and annotations | Detects tool poisoning, description tampering, and annotation manipulation |
| 2 | Artifact Integrity | Cryptographic digest of the tool's underlying code or package | Detects silent artifact replacement and supply-chain tampering |
| 3 | Side-Effect Declarations | Machine-readable declaration of all resource access and system behavior | Makes tool permissions visible, auditable, and enforceable by policy |
| 4 | Build Provenance | Links a tool to its source repository and build process | Enables full traceability from running tool back to source commit |
| 5 | Version Lineage | Hash-chained manifest history across tool versions | Detects rug-pulls, broken chains, and unauthorized version insertions |
| 6 | Multi-Party Attestation | Independent signatures from publishers, registries, scanners, and auditors | Provides defense in depth beyond a single publisher's word |
| 7 | Capability Declarations | Machine-readable declaration of required system resources and permitted actions | Enables least-privilege enforcement and permission scope auditing |

<br>

This model is implementation-agnostic and applies to:

* MCP-integrated LLM applications
* Agentic systems using MCP tool servers
* Multi-agent orchestration platforms
* Enterprise AI platforms with tool marketplaces
* MCP registries and hosting services
* Security tooling and compliance platforms

Security, trust, and governance are verification-enforced properties, not aspirational controls.

<br>

## MIS Trust Levels

MIS defines four graduated trust levels, enabling incremental adoption while providing measurable security value at every stage:

| Trust Level | Name | What It Provides |
|---|---|---|
| Level 0 | Unsealed | No manifest exists. TOFU (Trust On First Use) fingerprinting detects future changes. |
| Level 1 | Publisher-Sealed | Publisher has signed a manifest with interface fingerprints, side effects, and source provenance. |
| Level 2 | Attested | Level 1 plus registry attestation, artifact digest, build provenance, and version lineage. |
| Level 3 | Hardened | Level 2 plus independent scanner or auditor attestation and SLSA provenance. |

Even at Level 0, MIS clients provide value through TOFU mode — detecting any change to a tool's interface since it was first observed, without requiring any publisher adoption.

<br>

## View the MIS Security Specification

For the complete formal specification — including definitions, requirements, and conformance criteria — see the full reference document:

[MCP-Integrity-Standard-v1.0.0.md](./MCP-Integrity-Standard-v1.0.0.md)

For implementation guidance, JSON schemas, computation procedures, publisher workflows, test vectors, and example Sealed Manifests, see the supplemental document:

[MCP-Integrity-Standard-Supplemental-v1.0.0.md](./MCP-Integrity-Standard-Supplemental-v1.0.0.md)

**Status:** Public Draft for Community Review
**Version:** v1.0.0
**License:** Apache License 2.0
**Date:** 2026-02-24

<br>

## Why MIS Exists

MCP tools differ fundamentally from traditional software components:

* Behavior is shaped by tool descriptions that are read and acted on by LLMs
* A single change to a tool's description can redirect an entire AI workflow
* Tools can be silently updated after adoption without user knowledge
* Tool annotations that control safety behavior (e.g., `destructiveHint`) can be tampered with
* Publisher identity is not cryptographically bound to tool identity in the base MCP protocol
* Permission scope is not declared in a machine-readable, enforceable format

MIS directly addresses these risks by closing gaps that transport security alone cannot cover, including:

* Tool description poisoning and prompt injection via tool metadata
* Silent tool updates that introduce malicious behavior (rug pulls)
* Unauthorized artifact replacement without interface changes
* Undisclosed side-effect or permission expansion across versions
* Tool spoofing and name shadowing attacks
* Lack of auditable records for tool verification decisions

MIS is aligned with the OWASP MCP Top 10 (2025), directly addressing MCP02, MCP03, MCP04, MCP05, and MCP08.

<br>

## Conformance Levels

MIS supports incremental adoption through defined conformance levels for clients, publishers, and registries:

* **Client Level 1** — Parse and verify Sealed Manifests, compute interface fingerprints, support TOFU mode, enforce basic policy modes, and log verification events

* **Client Level 2** — Full conformance including artifact verification, lineage chain validation, all discovery mechanisms, and persistent fingerprint caching

* **Publisher Conformance** — Generate well-formed Sealed Manifests, sign with Ed25519 or ES256, publish through at least one discovery mechanism, and accurately declare side effects and capabilities

* **Registry Conformance** — Validate and store Sealed Manifests, verify publisher signatures, serve manifests with correct content types, and provide a JWKS key discovery endpoint

Partial adoption is allowed, but implemented controls must not weaken the security guarantees of the controls that are in place.

<br>

## Contribute

MIS is an open security standard in public draft. This repository is the canonical home for the specification and its evolution through transparent, community-driven collaboration.

We invite participation from:

* Security architects and researchers
* MCP server and client developers
* AI platform and host application builders
* Registry operators and tool marketplace builders
* Standards authors and compliance professionals
* Auditors and scanner tool vendors

**Ways to contribute:**

* Clarify requirements or terminology
* Propose extensions that preserve existing security guarantees
* Review changes for correctness and security impact
* Improve interoperability and adoption readiness
* Contribute test vectors or reference implementation feedback

**Why contribute?**

* Help define the future of MCP tool security
* Reduce systemic risk in the growing MCP ecosystem
* Enable auditable, policy-enforced AI tool deployments
* Establish cryptographically-grounded trust boundaries for AI-integrated tools

**How to get started:**

1. Fork the repository
2. Review the MIS specification and supplemental documents
3. Open a Pull Request with proposed changes
4. Participate in technical discussion and review

<br>

By working together, we can make MIS a foundational security standard for trustworthy, production-grade MCP tool ecosystems.

<br>
