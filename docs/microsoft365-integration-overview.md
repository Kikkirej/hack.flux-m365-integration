# Microsoft 365 Integration — Architecture Options

> **Status: open decision — not yet made.** This document exists to compare the two viable
> architectures side by side so the decision can be made deliberately (likely requiring input
> from a security review and/or licensing owner), not to recommend one over the other.

## Why there are two options

This integration should support large enterprise tenants, where permission governance tends to
be strict and every scope needs a clear justification. Two of the six features (managing Teams channel membership and Team membership)
have no way to be scoped down to just the hackathon's own Team using Microsoft Graph today —
see [Option A](microsoft365-option-a-graph-api.md) for details. That single constraint is what
makes this a genuine architecture fork rather than an implementation detail:

- **Option A — direct Microsoft Graph API calls** from the Kotlin service, authenticated
  app-only. Simple, single-codebase, but two of its permissions are necessarily tenant-wide.
- **Option B — a Power Automate flow** as a relay, driven by a dedicated service account that
  only owns the specific Teams/mailboxes involved. Genuinely smaller blast radius, at the cost
  of a second system, a license, and some ongoing operational maintenance.

## Comparison

| | Option A — Graph API (app-only) | Option B — Power Automate relay |
|---|---|---|
| Blast radius | 2 of 7 permissions are tenant-wide (all Teams/groups in the org), mitigated with compensating controls | Bounded to exactly the Hackathon/Organization Teams and two mailboxes, via account ownership/delegate rights |
| License cost | None beyond the base tenant | Premium/per-flow Power Automate plan (needed for the HTTP trigger only) |
| Where the logic lives | One version-controlled Kotlin/Spring Boot service | Split: Kotlin service (Kafka, templating) + a low-code flow in the Power Platform admin center |
| Operational maintenance | None — client-credentials tokens are unattended indefinitely | Service account's connection needs periodic human reconnect when Conditional Access forces re-auth |
| Feature coverage | All 6 features cleanly mapped to Graph endpoints | 5 of 6 features map to standard connector actions; the shared-channel feature likely needs a premium HTTP action too — see [Option B](microsoft365-option-b-power-automate.md) |
| What a security review needs to accept | A documented, monitored, tenant-wide grant for 2 permissions | A dedicated service account with elevated rights on specific resources; standard Exchange/Teams governance patterns |

## What would close this decision

- **For Option A:** get the security/architecture review to sign off on the tenant-wide
  `ChannelMember.ReadWrite.All` / `GroupMember.ReadWrite.All` grants, given the compensating
  controls documented in [Option A](microsoft365-option-a-graph-api.md#privilege--compliance-notes).
- **For Option B:** confirm Power Automate premium licensing is budgeted, and confirm the
  process for provisioning and owning a dedicated non-interactive service account is acceptable
  to IT governance.

Both options are documented in full so implementation can start immediately once one is chosen:

- [Option A — Microsoft Graph API setup](microsoft365-option-a-graph-api.md)
- [Option B — Power Automate relay setup](microsoft365-option-b-power-automate.md)
