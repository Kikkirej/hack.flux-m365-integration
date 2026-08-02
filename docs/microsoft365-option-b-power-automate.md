# Option B — Power Automate Relay

> See [microsoft365-integration-overview.md](microsoft365-integration-overview.md) — this is one
> of two options under an open architecture decision, not a confirmed direction.

The Kotlin service keeps owning Kafka consumption, event routing, and mail templating, but
instead of calling Microsoft Graph directly, it POSTs the templated payload to a Power Automate
flow. The flow's Teams/Outlook connector actions run under a **dedicated, non-interactive
service account**, so the blast radius is bounded by that account's actual rights — no
tenant-wide Graph application permission is needed.

## Prerequisites

- A **premium or per-flow Power Automate license** — required only because the base/seeded
  Microsoft 365 license doesn't include the HTTP trigger connector. Confirm this is budgeted
  before committing to this option.
- Ability to provision a **dedicated service account** (a normal Entra ID user, not an app
  registration) that IT governance is comfortable owning long-term.
- Team-owner access to the Hackathon Team and Organization Team (to grant that service account
  ownership), and mailbox-admin access (to grant Send As / delegate rights).

## 1. Provision the service account

Create a standard, non-interactive Entra ID user account dedicated to this integration (e.g.
`svc-hackflux-m365@contoso.com`). This account should hold **no** directory roles and be a
member/owner of nothing beyond what's granted in the next two steps — its entire footprint
should be auditable at a glance.

## 2. Grant it exactly the rights each feature needs

| Right | Grant | How |
|---|---|---|
| Create/manage channels, add/remove channel members, add team members | **Owner** of the Hackathon Team | Teams client → Team → Manage team → Members → add as owner |
| Associate a shared channel with the Organization Team | **Owner** of the Organization Team (or whatever role that operation requires — verify at implementation time) | Same as above, on the Organization Team |
| Send template mails | **Send As** on the sender shared mailbox | Exchange Admin Center → Mailboxes → sender mailbox → delegation, or `Add-RecipientPermission "hackathon@contoso.com" -AccessRights SendAs -Trustee svc-hackflux-m365@contoso.com` |
| Silently add attendees to the Event-Blockers calendar event | **Editor** delegate on the event-blocker mailbox's calendar | Exchange Admin Center, or `Add-MailboxFolderPermission "blockers@contoso.com:\Calendar" -User svc-hackflux-m365@contoso.com -AccessRights Editor` |

Because this is ordinary Exchange/Teams delegation rather than a Graph application permission,
it's a request pattern most enterprise IT governance processes already have a fast path for.

## 3. Build the flow(s)

1. Create a new **Instant cloud flow** triggered by **"When an HTTP request is received"**.
   Define the expected JSON schema for the payload the Kotlin service will send (event type,
   user email, topic UUID, etc.).
2. Branch on event type (`User registered` / `User added to topic` / `User removed from topic`)
   and map each branch to the corresponding actions below, using connections signed in as the
   service account from step 1:

| Feature | Action | Connector |
|---|---|---|
| Sending Mails | **Send an email (V2)** | Office 365 Outlook (Standard) |
| Sending events (silent attendee add) | **Update event (V3)** on the event-blocker mailbox's event | Office 365 Outlook (Standard) — **verify at implementation time** whether this action updates attendees without triggering Exchange's "meeting updated" notification to existing attendees; if it does, this feature may need a different approach even under this option |
| Create Teams Channel | **Create a channel** | Microsoft Teams (Standard) |
| Add/Remove User to/from Teams-Channel | **Add member to channel** / **Remove member from channel** | Microsoft Teams (Standard) |
| Add User to Team | **Add a member to a team** | Microsoft Teams (Standard) |
| Add Organization Team to Teams Channel (shared channel) | **No native action found in the standard Teams connector as of this writing.** Likely requires an **HTTP** action calling `POST /teams/{id}/channels/{id}/sharedWithTeams` directly against Graph — which is itself a premium action, and would need its own delegated or app-only auth setup within the flow. Confirm at implementation time whether Microsoft has since added a connector action for this. |

3. Secure the HTTP trigger: either require Entra ID authentication on the trigger URL (the
   Kotlin service authenticates the outbound call with its own app registration, but only for
   *invoking the flow* — a much narrower ask than any Graph permission, since it doesn't grant
   access to any M365 data by itself), or use the trigger's built-in SAS-style URL and treat it
   as a secret (`hackflux.m365.flow-webhook-secret`), embedded in the callback URL itself
   (`hackflux.m365.flow-webhook-url`).

## 4. Operational notes

- Power Automate manages the service account connection's OAuth token refresh automatically.
  When Conditional Access eventually forces re-authentication (expected periodically in a tenant
  this size), the connection will show as broken in the flow's run history / owner
  notifications — this needs a documented runbook step (who gets notified, who reconnects).
- Because business logic is split between the Kotlin service and this flow, treat the flow
  definition as production infrastructure: use Power Platform solutions for export/import
  between environments, and keep a change log, even though it isn't in this git repository.

## Collect the remaining configuration values

Same `hackflux.m365.hackathon-team-id`, `organization-team-id`, `channel-prefix`,
`sender-mailbox`, `event-blocker-mailbox`, and `event-blocker-event-id` values as
[Option A](microsoft365-option-a-graph-api.md#6-collect-the-remaining-configuration-values) are
needed here too — they're used to template the payload sent to the flow rather than to call
Graph directly. Map these, plus `flow-webhook-url` / `flow-webhook-secret`, onto the config keys
documented in the [README](../README.md#configuration).
