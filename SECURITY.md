# Security Policy

This repository implements security enforcement components and therefore treats vulnerabilities in the project itself as high priority.

## Reporting a vulnerability

Please do not publish exploitable vulnerabilities, proof-of-concept payloads, credentials, tokens, private keys, or sensitive production data in a public issue.

Use GitHub's private vulnerability reporting feature for this repository when available. If that feature is unavailable, contact the repository maintainer privately before public disclosure.

A useful report includes:

- affected version or commit;
- affected blocker / transport / SDK;
- attack preconditions;
- minimal reproduction steps;
- expected security invariant and observed behavior;
- impact;
- suggested mitigation, if known.

## High-priority vulnerability classes

Reports receive especially high priority when they involve:

- bypass of a Blocker decision;
- fail-open behavior after parser, policy, timeout, or internal errors;
- parser differentials or ambiguous request interpretation;
- duplicate-key / Unicode / framing inconsistencies;
- command, SQL, path, URL, template, or header injection in this project itself;
- authentication or local IPC boundary bypass;
- secret leakage through stdout, stderr, logs, crashes, telemetry, or CI;
- unbounded input, recursion, decompression, memory, CPU, disk, concurrency, or output amplification;
- GitHub Actions / release pipeline compromise;
- dependency or artifact provenance compromise.

## Disclosure

Please allow time to investigate, fix, test, and release a remediation before public disclosure. Security fixes should include a regression test whenever practical.

## Security invariants

The implementation must preserve the rules documented in:

- [Input / Output Security](docs/input-output-security.md)
- [Security CI](docs/security-ci.md)
- [Common Protocol](docs/common-protocol.md)

A change that weakens one of these invariants should be treated as a security-sensitive design change, not as an ordinary refactor.
