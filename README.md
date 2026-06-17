# Agent Security Threat Model

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
![Status](https://img.shields.io/badge/status-public%20draft-blue)
![Version](https://img.shields.io/badge/version-v0.3-informational)

A practical threat model for AI agents that use tools, browsers, files, terminals, APIs, and cloud resources.

**A threat model for agents is a model of seams, not boxes.** The model is benign. The browser is benign. The cloud token is benign. The compromise lives at the seam where they meet. This document is organized around trust boundaries for exactly that reason, and it maps each threat to the OWASP Top 10 for Agentic Applications (the ASI list) and the OWASP Top 10 for LLM Applications.

## What this is

A working threat model, not a position paper. It carries a single through-line from asset to control: assets the attacker wants, the trust boundaries those assets sit behind, eleven concrete threats, a threat-to-boundary-to-control matrix, nine controls rated by what they stop and what they do not, and a pre-deployment checklist you can run before an agent reaches production.

Every threat is anchored in a real 2025 or 2026 incident (EchoLeak / CVE-2025-32711, the GitHub MCP confused-deputy exploit, the Replit production-database deletion, GitLab Duo, Unit 42's Agent Session Smuggling, CyberArk's full-schema poisoning). The intent is to make the model usable in review, not just readable.

## Who it is for

Engineers deciding what an agent is allowed to touch, where to put a human in the loop, and what to log. Security reviewers doing design review or pre-deployment review. Red teamers planning agent engagements.

## The document

The full threat model lives in [`agent-security-threat-model.md`](./agent-security-threat-model.md).

Structure:

1. Scope and method
2. Reference architecture (the canvas the threats attach to)
3. Assets
4. Trust boundaries, including the lethal-trifecta framing
5. Threats (4.1 through 4.11)
6. Threat-to-boundary-to-control matrix
7. Controls (what each stops, what each does not)
8. The unsolved core (why prompt injection is architectural, not patchable)
9. Pre-deployment review checklist
10. References

## How to use it

Start with the three trifecta questions in the checklist to scope blast radius: what untrusted content can the agent ingest, what private data can it reach, and what external actions can it take. Identify which trust boundaries the agent actually crosses. Then walk the control matrix to decide where deterministic mediation, scoping, logging, and human approval belong.

The matrix is meant to be forked and edited for your own system. No column in it is all primary-control, which is the honest position: defense here is layered containment, not a single fix.

## Frameworks referenced

- OWASP Agentic Security Initiative: Top 10 for Agentic Applications 2026 (ASI01 through ASI10)
- OWASP Top 10 for LLM Applications 2025 (LLM01, LLM02, LLM06)
- The lethal trifecta (Simon Willison, June 2025) and Meta's Agents Rule of Two (October 31, 2025)
- Dual-LLM / CaMeL design-pattern work (Google DeepMind, 2025)

Mapping names follow the published OWASP lists at time of writing. Verify against the current PDF before quoting a label in an assessment.

## Revision history

This started as a v0.1 draft and went through external review to v0.3. The version history is part of the artifact, not overhead: it shows where sourcing held under challenge and where it did not.

- **v0.1** Initial draft: reference architecture, assets, boundaries, ten threats, matrix, controls, checklist.
- **v0.2** Corrected Meta's Agents Rule of Two date to October 31, 2025. Softened the EchoLeak firstness claim to a sourced "widely described as first-of-its-kind." Removed an unresolved CaMeL uncertainty note. Added NVD and Fortune citations and named the Replit incident directly. Held the OWASP December 2025 release date as triple-sourced.
- **v0.3** Promoted MCP / supply-chain poisoning to a first-class threat (4.11) and matrix row. Added tool/connector schemas as an asset and a connector-registration item to the checklist, so the ASI04 thread runs end to end.

## How to cite

If you use or adapt this work, attribution is required under CC BY 4.0. A ready-to-paste credit line:

> "Agent Security Threat Model" by Larry Peseckis, licensed under CC BY 4.0. Source: https://github.com/larrypeseckis/agent-security-threat-model

A machine-readable [`CITATION.cff`](./CITATION.cff) is included so GitHub renders a "Cite this repository" button.

## License

Document content is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0). You may share and adapt it, including commercially, with attribution.

Add the `LICENSE` file via GitHub's license picker (New file, name it `LICENSE`, choose the Creative Commons Attribution 4.0 template) rather than pasting it by hand.

## Author

Larry Peseckis. More security work at [larrypeseckis.ai](https://larrypeseckis.ai).

---

*Agent Security Threat Model v0.3 | Public Draft | CC BY 4.0*
