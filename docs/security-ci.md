# Security CI

This repository implements security controls that other applications may rely on. The repository and its CI/CD pipeline are therefore part of the product's trusted computing base.

The CI design uses defense in depth and assumes that:

- pull request contents are untrusted;
- dependency metadata is untrusted;
- GitHub Actions configuration can itself contain injection vulnerabilities;
- third-party Actions and downloaded tools are supply-chain dependencies;
- secrets may be committed accidentally;
- a scanner can miss a vulnerability;
- a scanner itself can be compromised.

---

## 1. Active workflows

### `Security CI`

File:

```text
.github/workflows/security-ci.yml
```

Runs on:

- pull requests;
- pushes to `main`;
- weekly schedule;
- manual dispatch.

Merge-gate jobs:

- `workflow-hardening`
- `secret-scan`
- `filesystem-security`
- `dependency-review` on pull requests
- `protocol-contract`

### `Security Posture`

File:

```text
.github/workflows/security-posture.yml
```

Runs OpenSSF Scorecard against repository-level security posture.

This is a posture/monitoring control rather than the primary code gate.

---

## 2. GitHub Actions hardening

### Default token permissions

Workflows start with:

```yaml
permissions: {}
```

Each job grants only the permission it requires.

Most jobs use:

```yaml
permissions:
  contents: read
```

No pull-request job receives write permission, OIDC permission, or repository secrets by design.

### No `pull_request_target`

Security test workflows use `pull_request`, not `pull_request_target`.

A privileged workflow must not check out and execute untrusted PR code.

### Immutable Action references

Every `uses:` reference must use a full commit SHA, with the human-readable release version in a comment.

Example:

```yaml
uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

Tags and branches are not accepted for executable Actions.

### Checkout credentials

```yaml
persist-credentials: false
```

is required unless a reviewed workflow has a documented need to push.

### Timeouts

Every job has a `timeout-minutes` value.

No security job is allowed to run indefinitely.

### Concurrency

Repeated runs for the same workflow/ref cancel earlier in-progress runs.

This reduces resource abuse and stale results.

---

## 3. Scanner supply-chain model

For security scanners that provide standalone binaries, CI avoids adding an Action wrapper.

Instead:

1. download a specific official release asset over HTTPS;
2. verify its pinned SHA-256 digest;
3. extract;
4. run the scanner with no secrets beyond the minimum read token where required.

Currently pinned:

| Tool | Version |
|---|---|
| actionlint | 1.7.12 |
| zizmor | 1.30.1 |
| Gitleaks | 8.30.1 |
| Trivy | 0.75.0 |
| OpenSSF Scorecard | 5.5.0 |

The checksum is part of the reviewed workflow.

Dependabot updates GitHub Actions references. Scanner binary updates remain explicit security-sensitive changes so release asset digests are reviewed with the version bump.

---

## 4. Workflow security scanning

### actionlint

Checks GitHub Actions workflow syntax and expression problems.

### zizmor

Runs with:

- strict collection;
- pedantic persona;
- low-or-higher findings as failures;
- online GitHub metadata using only a read token.

It is intended to catch issues such as:

- unpinned Actions;
- template injection;
- excessive token permissions;
- dangerous workflow patterns;
- risky cache behavior;
- suspicious Action references;
- other GitHub Actions security mistakes.

---

## 5. Secret scanning

Gitleaks scans repository history with redaction enabled.

The CI intentionally fetches complete history for this job.

This is a backstop, not a replacement for GitHub Secret Scanning / Push Protection.

Repository settings should also enable:

- Secret Scanning;
- Push Protection;
- review/resolution of detected secrets before merge where supported.

A leaked credential must be rotated even if it is later deleted from Git history.

---

## 6. Filesystem / dependency / configuration scanning

Trivy scans the repository for:

- dependency vulnerabilities;
- configuration weaknesses;
- secrets.

The gate currently fails on all reported vulnerability severities.

This is intentionally strict during the early project phase. The policy can later move to a documented risk-acceptance mechanism, but should not silently add broad ignores.

---

## 7. Dependency review

Pull requests run GitHub's Dependency Review Action pinned to a full commit SHA.

Current policy:

- fail on `low` or higher known vulnerability severity;
- inspect runtime, development, and unknown scopes;
- check license metadata;
- surface OpenSSF package score information.

Any future allowlist must identify a specific advisory/package and include a reason plus expiration/review date.

---

## 8. Protocol contract guard

CI verifies that the common request/response schemas remain strict and that the input/output security contract retains core invariants.

This does not replace implementation tests. It makes accidental weakening of the documented boundary more visible.

Once the Core is implemented, this job should additionally run:

- schema conformance tests;
- parser differential tests;
- negative corpus;
- fuzz regression corpus;
- stdout/stderr separation tests;
- fail-closed tests;
- resource-budget tests.

---

## 9. CodeQL

The repository currently contains design documents and no final Core implementation language.

When the first supported-language source files are introduced, enable GitHub CodeQL default setup immediately.

Recommended initial setting:

- default setup;
- Extended / `security-extended` query suite;
- pull request and default branch scanning;
- code scanning merge protection.

GitHub default setup can automatically add newly detected supported languages.

For a high-risk Core, switch to advanced/manual build configuration where it materially improves analysis fidelity, and add custom CodeQL models for project-specific input sources / dangerous sinks when useful.

---

## 10. Language-specific gates

When the Core language is selected, add mandatory native tooling rather than relying only on generic scanners.

Examples:

### Rust

- `cargo fmt --check`
- `cargo clippy -- -D warnings`
- `cargo test`
- `cargo audit` / RustSec
- fuzzing via cargo-fuzz / libFuzzer
- sanitizer jobs where supported
- Miri for unsafe-sensitive components where practical
- deny unsafe code by default and isolate required `unsafe`

### Go

- `go test ./...`
- `go vet ./...`
- `govulncheck ./...`
- built-in fuzz tests
- race detector for concurrent code

### Python

- unit/property tests
- type checking
- pip-audit
- Bandit / Semgrep as supplemental checks
- Hypothesis fuzz/property tests

### TypeScript / Node.js

- strict TypeScript
- test suite
- package audit / OSV
- ESLint security rules as supplemental checks
- property/fuzz testing for parser boundaries

The implementation language determines the final stack.

---

## 11. Required repository settings

File-based CI is not sufficient because a pull request can attempt to change CI files themselves.

The repository should use a branch ruleset or branch protection on `main`.

Recommended settings:

- require pull requests before merging;
- require all Security CI status checks;
- block force pushes;
- block branch deletion;
- require conversation resolution;
- require branch to be up to date before merge if operationally acceptable;
- require code scanning results once CodeQL is enabled;
- require secret scanning alerts to be resolved where available;
- do not allow bypass except for explicitly documented emergency recovery.

`.github/CODEOWNERS` marks CI and security contracts as security-sensitive.

If the repository gains a second maintainer, enable required Code Owner review and require approval of the most recent push for security-critical changes.

For a single-maintainer repository, mandatory independent approval cannot be meaningfully satisfied; status checks and platform protections should still be enforced.

---

## 12. GitHub repository security features

Recommended platform features:

- Dependency Graph: enabled
- Dependabot alerts: enabled
- Dependabot security updates: enabled
- Secret Scanning: enabled
- Push Protection: enabled
- CodeQL default setup: enable when supported source code lands
- private vulnerability reporting: enabled
- Actions default workflow permissions: read-only
- allow GitHub Actions to create/approve PRs: disabled unless explicitly needed
- restrict allowed Actions where practical
- require Actions to be pinned to a full commit SHA if the repository setting is available

---

## 13. Release security

When binaries begin shipping, release CI must be separate from untrusted PR CI.

Release jobs should:

- run only from trusted tags/refs;
- have minimal write permissions;
- build in ephemeral hosted runners;
- not execute artifacts produced by untrusted PR workflows;
- generate SBOMs;
- generate GitHub artifact attestations / provenance;
- publish checksums;
- sign releases where practical;
- verify release inputs before publication.

Attestations establish provenance; they do not replace vulnerability testing.

---

## 14. Security test categories for the product itself

Before v1.0, CI should cover:

### Parser / protocol

- fuzzing;
- duplicate-key rejection;
- Unicode edge cases;
- frame desynchronization;
- oversized payloads;
- numeric boundaries;
- unknown-field rejection.

### Resource exhaustion

- parser nesting;
- large arrays/maps;
- many connections;
- slow sender;
- output amplification;
- cancellation races.

### Isolation

- UDS / named pipe permissions;
- peer identity;
- localhost binding;
- socket path attacks;
- stale socket handling.

### Information disclosure

- logs;
- error messages;
- crash/panic paths;
- stack traces;
- environment leakage;
- CI logs.

### Blocker bypass

Each blocker gets its own adversarial corpus and property tests.

---

## 15. No blanket suppression

Security findings should not be disabled globally because they are inconvenient.

Any suppression should be:

- as narrow as possible;
- documented;
- reviewed;
- tied to a concrete false positive or accepted risk;
- periodically revisited.

The target state is that the repository stays clean enough for strict CI to remain usable.
