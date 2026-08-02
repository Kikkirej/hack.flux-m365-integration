# Option A — Direct Microsoft Graph API Integration

> See [microsoft365-integration-overview.md](microsoft365-integration-overview.md) — this is one
> of two options under an open architecture decision, not a confirmed direction.

The service calls Microsoft Graph directly, authenticated app-only (OAuth2 client-credentials —
no signed-in user, which fits a headless Kafka consumer). Recommended client: the
**Microsoft Graph SDK for Java** (`com.microsoft.graph:microsoft-graph`) authenticated via
**Azure Identity**'s `ClientSecretCredential` or, preferably, `ClientCertificateCredential`
(`com.azure:azure-identity`). The SDK handles token acquisition/caching, retry-on-throttling
(HTTP 429), and pagination, which would otherwise have to be hand-rolled on top of a plain
`WebClient`.

## Prerequisites

- Entra ID **Application Administrator** access (to register the app and grant admin consent).
- **Exchange Administrator** access (to create the Application Access Policy).
- Ownership of the Hackathon Team (to install the RSC-consented Teams app package).

## 1. Register the application

1. In the [Entra admin center](https://entra.microsoft.com) → **App registrations** → **New registration**.
2. Single tenant (`Accounts in this organizational directory only`) — this integration has no
   reason to be multi-tenant.
3. No redirect URI is needed (client-credentials flow only).
4. Note the **Application (client) ID** and **Directory (tenant) ID** — these become
   `hackflux.m365.client-id` and `hackflux.m365.tenant-id`.

## 2. Add tenant-wide Application permissions

Under **API permissions** → **Add a permission** → **Microsoft Graph** → **Application permissions**,
add:

| Permission | Why |
|---|---|
| `Mail.Send` | Send template mails from the sender shared mailbox |
| `Calendars.ReadWrite` | Silently add a registered user as an attendee on the existing Event-Blockers calendar event |
| `ChannelMember.ReadWrite.All` | Add/remove users from a Teams channel; associate the Organization Team with a shared channel |
| `GroupMember.ReadWrite.All` | Add a user to the Hackathon Team (Teams are backed by M365 Groups) |
| `User.ReadBasic.All` | Resolve a user's Graph object ID from their email address — the least-privileged read permission for basic profile properties |

Click **Grant admin consent** — this requires a Global Administrator or Privileged Role
Administrator.

`Channel.Create.Group` is **not** added here — see step 4.

## 3. Create a credential

Prefer a **certificate** over a client secret, backed by a managed key vault, for a tenant at
this scale. Under **Certificates & secrets**:

- **Certificates** (preferred): upload the public key of a certificate whose private key is
  stored in a vault (e.g. Azure Key Vault) and never leaves it.
- **Client secrets** (acceptable, but rotate on a defined schedule — e.g. every 6 months —
  and track the expiry date): generates the value for `hackflux.m365.client-secret`.

## 4. Grant `Channel.Create.Group` via Resource-Specific Consent (RSC), scoped to the Hackathon Team only

Unlike the permissions above, channel creation can be scoped to *just* the Hackathon Team
instead of every team in the tenant, using RSC:

1. Create a minimal Teams app manifest (`manifest.json`) declaring:
   - `webApplicationInfo.id` = the Application (client) ID from step 1.
   - `authorization.permissions.resourceSpecificApplicationPermissions` = `["Channel.Create.Group"]`.
2. Package it as a custom/LOB Teams app (zip with the manifest + icons) and upload it to the
   tenant's Teams app catalog (or sideload it directly, depending on tenant app-upload policy).
3. Have a **Hackathon Team owner** add the app to the Hackathon Team from the Teams client —
   this both installs it and grants the RSC consent, scoped to that team only. No tenant admin
   action is required for this step.

If the Organization Team also needs `Channel.Create.Group` (e.g. for the shared-channel
association), repeat step 3 for that team.

## 5. Restrict `Mail.Send` / `Calendars.ReadWrite` with an Exchange Application Access Policy

Without this step, `Mail.Send` and `Calendars.ReadWrite` let the app act on **every** mailbox in
the tenant. Scope it to only the two mailboxes this integration actually needs:

```powershell
Connect-ExchangeOnline

New-DistributionGroup -Name "hackflux-m365-integration-mailboxes" -Type Security
Add-DistributionGroupMember -Identity "hackflux-m365-integration-mailboxes" -Member "hackathon@contoso.com"
Add-DistributionGroupMember -Identity "hackflux-m365-integration-mailboxes" -Member "blockers@contoso.com"

New-ApplicationAccessPolicy `
  -AppId "<client-id-from-step-1>" `
  -PolicyScopeGroupId "hackflux-m365-integration-mailboxes" `
  -AccessRight RestrictAccess `
  -Description "hackflux-m365-integration: restrict to sender + event-blocker mailboxes"

# Verify:
Test-ApplicationAccessPolicy -AppId "<client-id-from-step-1>" -Identity "hackathon@contoso.com"
Test-ApplicationAccessPolicy -AppId "<client-id-from-step-1>" -Identity "someone-else@contoso.com"
```

The second `Test-ApplicationAccessPolicy` call should report access denied.

## 6. Collect the remaining configuration values

| Value | Where to find it |
|---|---|
| `hackflux.m365.hackathon-team-id` | Team → **Get link to team** in Teams, or the underlying group's Object ID in Entra ID → Groups |
| `hackflux.m365.organization-team-id` | Same as above, for the Organization Team |
| `hackflux.m365.channel-prefix` | Chosen by the team running the hackathon (e.g. `topic-`) |
| `hackflux.m365.sender-mailbox` | The shared mailbox's primary SMTP address |
| `hackflux.m365.event-blocker-mailbox` | The Event-Blockers shared mailbox's primary SMTP address |
| `hackflux.m365.event-blocker-event-id` | `GET /users/{event-blocker-mailbox}/events` via Graph Explorer, find the target event, copy its `id` |

Map these onto the config keys documented in the [README](../README.md#configuration).

## 7. Verify the setup

Using [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) or a quick
`curl`, confirm a client-credentials token can be minted and used:

```bash
curl -X POST "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token" \
  -d "client_id=<client-id>" \
  -d "client_secret=<client-secret>" \
  -d "scope=https://graph.microsoft.com/.default" \
  -d "grant_type=client_credentials"

curl -H "Authorization: Bearer <access-token>" \
  "https://graph.microsoft.com/v1.0/teams/<hackathon-team-id>/channels"
```

## Privilege & compliance notes

Verified against current Microsoft documentation: `ChannelMember.ReadWrite.All` and
`GroupMember.ReadWrite.All` have **no Resource-Specific Consent or "Selected" scoped
alternative today** — Microsoft's own RSC permissions reference lists the write variant for
channel members as not yet supported for application access, and there is no `Group.Selected`
permission (that pattern exists for SharePoint sites, not groups). This means these two
permissions are unavoidably tenant-wide: the app registration *could* add/remove members on any
team or group in the organization, not just the hackathon's own.

This should be presented to a security review as a **documented, monitored, accepted risk**,
not an oversight, backed by the following compensating controls:

1. **Single-purpose app registration.** This app registration is used for nothing else, so its
   permission grants are easy to reason about in isolation.
2. **Application-layer allow-listing.** The service only ever calls `ChannelMember`/`GroupMember`
   operations against the configured `hackathon-team-id` / `organization-team-id` — never an ID
   taken unvalidated from a Kafka event payload.
3. **Audit-log alerting.** Configure alerting (Entra ID audit logs, or a SIEM such as Microsoft
   Sentinel) to flag any action by this service principal against a group/channel ID outside the
   allow-list as a security incident — this turns a broad static grant into an effectively
   monitored, narrow one.
4. **Certificate-based credential**, ideally backed by a managed key vault, in preference to a
   client secret (see step 3), with a documented rotation cadence.
5. **Optional, if licensed:** Conditional Access for workload identities, restricting where the
   app's client-credentials token can be used from (e.g. a known egress IP range).

If Microsoft later ships RSC or `.Selected` support for channel/team membership management, the
tenant-wide grants here should be replaced with the scoped equivalent and this section updated.
