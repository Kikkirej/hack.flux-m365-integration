# Hack.flux Integration - Microsoft 365

[Hack.flux](https://sr.ht/~chirpcel/hack.flux/) is a tool for organizing hackathons. To integrate in existing infrastructure it enables integrating by reacting to events. This integration is meant for integrating in Microsoft 365 Ecosystem.

Features:

* Sending Mails
* Sending events
* Create Teams Channel
* Add User to Team
* Add User to Teams-Channel
* Remove User from Teams-Channel

## Events

### User registered

This event is triggered when a user registers for the hackathon.

The following steps happen:

1. Mail to the User from a template
2. Add user to Hackathon Team
3. Add User to Event-Blockers from shared mailbox, defined in the settings. 

### User added to topic

1. Create Teams Channel (Private Teams Channel) - if it not exists
2. Add User to Teams Channel.
3. Add Organization Team to Teams Channel.

Teams Channel is identified by UUID of the topic, written in the description field)

Prefix can be defined via an option.

### User removed from topic

1. Remove User from Teams Channel

## Mail Templates

* User Registration Mail


## Microsoft 365 Integration

> **Open decision — not yet made.** See [docs/microsoft365-integration-overview.md](docs/microsoft365-integration-overview.md)
> for the full comparison. Summarized here so it isn't missed.

Large enterprise tenants should be supported, where permission governance tends to be strict. Two of the six
features (managing Teams channel membership and Team membership) have no way to be scoped down
to just the hackathon's own Team using Microsoft Graph today, which turns "how do we call M365"
into a genuine architecture fork rather than an implementation detail. Two options are fully
documented, and one needs to be chosen before implementation starts:

- **[Option A — direct Microsoft Graph API](docs/microsoft365-option-a-graph-api.md):** the
  service calls Graph directly (Microsoft Graph SDK for Java + Azure Identity, app-only
  client-credentials OAuth2). Simple, single-codebase, mail/calendar permissions scoped via an
  Exchange Application Access Policy and channel creation scoped via Resource-Specific Consent
  (RSC) — but channel/team *membership* management has no scoped Graph alternative today, so
  those two permissions are necessarily tenant-wide (mitigated with compensating controls:
  dedicated app registration, application-layer allow-listing, audit-log alerting,
  certificate-based auth).
- **[Option B — Power Automate relay](docs/microsoft365-option-b-power-automate.md):** the
  service POSTs to a Power Automate flow, which acts as a dedicated, narrowly-scoped service
  account (owner of the specific Teams, delegate on the specific mailboxes) — genuinely smaller
  blast radius via ordinary Exchange/Teams governance, at the cost of a premium Power Automate
  license, splitting logic across two systems, and some ongoing operational maintenance
  (reconnecting the service account when Conditional Access forces re-auth).

## Configuration

| Setting | Description | Example | Default |
| ---- | ---- | ----| ---- |
| `spring.kafka.bootstrap-servers` | Endpoint(s) of the Kafka broker(s) to connect to | `localhost:9092` | `localhost:9092` |
| `spring.kafka.consumer.group-id` | Consumer group id used when reading from Kafka | `hackflux-m365-integration` | `hackflux-m365-integration` |
| `hackflux.kafka.topic` | Kafka topic to consume Hack.flux events from | `hackflux-events` | - |

### Microsoft 365 — common to both options

| Setting | Description | Example | Default |
| ---- | ---- | ----| ---- |
| `hackflux.m365.hackathon-team-id` | Team users are added to on registration | `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee` | - |
| `hackflux.m365.organization-team-id` | Team added as a shared-channel member on topic channels | `bbbbbbbb-cccc-dddd-eeee-ffffffffffff` | - |
| `hackflux.m365.channel-prefix` | Prefix for Teams channel names created per topic | `topic-` | `` |
| `hackflux.m365.sender-mailbox` | Shared mailbox used as sender for template mails | `hackathon@contoso.com` | - |
| `hackflux.m365.event-blocker-mailbox` | Shared mailbox containing the Event-Blockers calendar | `blockers@contoso.com` | - |
| `hackflux.m365.event-blocker-event-id` | ID of the calendar event registered users are silently added to | `AAMkAG...` | - |

### Option A — direct Graph API only

See [docs/microsoft365-option-a-graph-api.md](docs/microsoft365-option-a-graph-api.md) for how to obtain these.

| Setting | Description | Example | Default |
| ---- | ---- | ----| ---- |
| `hackflux.m365.tenant-id` | Entra ID tenant ID | `contoso.onmicrosoft.com` | - |
| `hackflux.m365.client-id` | App registration (client) ID | `11111111-2222-3333-4444-555555555555` | - |
| `hackflux.m365.client-secret` | App registration client secret (prefer a certificate credential in production) | *(secret, externalize)* | - |

### Option B — Power Automate relay only

See [docs/microsoft365-option-b-power-automate.md](docs/microsoft365-option-b-power-automate.md) for how to obtain these.

| Setting | Description | Example | Default |
| ---- | ---- | ----| ---- |
| `hackflux.m365.flow-webhook-url` | URL of the Power Automate flow's HTTP trigger | `https://prod-00.westeurope.logic.azure.com/workflows/...` | - |
| `hackflux.m365.flow-webhook-secret` | Shared secret for authenticating to the flow's HTTP trigger (if not using Entra ID auth on the trigger) | *(secret, externalize)* | - |

## Dev Start

````
./gradlew bootRun
````