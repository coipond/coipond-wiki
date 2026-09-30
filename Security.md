# Security

How CoiPond contains agent sessions, keeps host secrets out of containers, and guards cross-node traffic, plus how to report a vulnerability.

## Security Properties

- Session containers are isolated by coi; network access is constrained per session by the coi profile's `[network]` policy. See [Network Profiles](Network-Profiles).
- Host secrets never blanket-cross into containers - only profile-allow-listed variable names do, with values from CoiPond's own scoped store. See [Configuration](Configuration).
- Cross-node terminal streaming/input is gated by short-lived, session-scoped, signed capability tokens.
- CI includes Brakeman, bundler-audit, and a gitleaks secret scan.

## Reporting a Vulnerability

To report a vulnerability, see SECURITY.md in the repository.
