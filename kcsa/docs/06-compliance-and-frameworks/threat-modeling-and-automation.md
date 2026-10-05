# Threat Modeling Frameworks, Supply Chain Compliance and Automation

Up: [KCSA README](../../README.md) · Domain 6 — Compliance and Security Frameworks (10%) · Prev: [Frameworks and regulations](frameworks-and-regulations.md)

Official curriculum topics covered here: threat modeling frameworks, supply chain compliance, and automation and tooling. Compliance frameworks are in [frameworks and regulations](frameworks-and-regulations.md).

## Threat modeling frameworks

- **STRIDE**: categorizes threats per component or data flow (spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege). Used in [threat model and attack paths](../04-kubernetes-threat-model/threat-model-and-attack-paths.md).
- **MITRE ATT&CK** (including its container matrix): a catalog of adversary tactics and techniques, useful for mapping detections to attacks.
- **Attack trees and data flow diagrams**: model how an attacker reaches a goal and where data crosses trust boundaries.

Frameworks are thinking tools. The output is a list of threats, each with a control and an owner.

## Supply chain compliance

- Know what is in each artifact (SBOM) and where it came from (provenance).
- Require signed artifacts and verified builds before deployment.
- Keep evidence: build logs, scan results, signatures and approvals, so an auditor can trace a running image to its source.
- SLSA provides levels for build integrity; check the current specification for the exact requirements.

## Automation and tooling

- Policy as code and scanning in CI: block risky changes before merge.
- Continuous compliance: admission policies and scheduled scans, with results stored and reviewed.
- Automation reduces human error, but its own credentials and pipelines become targets. Protect them like production systems.
