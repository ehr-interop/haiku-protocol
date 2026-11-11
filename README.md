# haiku-protocol

Open documentation of the client–server wire protocol used by **Epic Haiku**
(Epic Systems' clinician-facing mobile EHR client) — the reference material for
building independent clients that speak the same protocol as the official app.

> **Not affiliated with Epic Systems Corporation.** This repository contains no
> Epic source code, binaries, or assets — only independently written protocol
> documentation and interface metadata. See `NOTICE`.

## What's in here

| Path | Contents |
|---|---|
| [docs/SPEC.md](docs/SPEC.md) | **Start here.** The consolidated wire specification: connection model (econfig provisioning), the two transports (SOAP facade + JSON Web API), headers, serialization rules, authentication, realtime, security posture |
| [docs/transport-protocol.md](docs/transport-protocol.md) | Deep dive: URL construction, SOAP envelope codec, multipart/JSON formats, faults, dispatch rules |
| [docs/auth-flow.md](docs/auth-flow.md) | Complete login sequence, all SSO variants (OIDC/SAML/Hyperspace facilitated login), token lifecycle, credential storage |
| [docs/org-discovery.md](docs/org-discovery.md) | econfig provisioning pipeline, environment configuration model, capability negotiation |
| [docs/security-webview-push.md](docs/security-webview-push.md) | TLS posture (no client pinning), Embedded Hyperspace WebView bridge, SignalR/FCM realtime channels |
| [catalog/message-spec.json](catalog/message-spec.json) | Machine-usable message catalog: 2,851 command records, 15,963 typed members with wire names |
| [catalog/capability-list.txt](catalog/capability-list.txt) | The 388 server capability wire-values used for feature gating |

## The protocol in one paragraph

Haiku speaks to each hospital's own Epic web servers over HTTPS. All work flows
through named commands: legacy commands are SOAP envelopes POSTed to a single
`/httplistener.ashx` dispatcher (action = body element name + `urn:epicsystems-com:EpicOnHand.*`
namespace, session in a `Guid` attribute and `Interconnect_SessionToken` header);
modern commands are flat-JSON bodies POSTed to `internal/Mobile/{Internal|Unauthenticated}/<CommandName>`
with optional `multipart/mixed+json` for binary properties. Login is a
PreLogin → (OAuth2) → multi-step wizard exchange ending in an Interconnect session
token; hospital connection info arrives as an encrypted, MDM- or deep-link-delivered
"econfig" document. Feature availability is gated per-server by a version +
capability-token model. Realtime is stock SignalR; push is FCM with AES-GCM payloads.

## Reading the message catalog

`catalog/message-spec.json` is keyed by record name (`Epic.Mobile.<Area>.<Op.Request|Response>`).
Each record lists its members with the final wire name, the source accessor, and a
readable type; `multipart: true` members are transmitted as binary parts. Wire names
follow the client's serialization metadata: the accessor name with the leading
`get`/`is` removed and first-letter case preserved, overridden per-member where the
metadata specifies an explicit transport name.

## Status

The specification is complete for the documented client version, with one open
category: `docs/SPEC.md` §9 lists the few items that require live observation to
confirm (wire-name casing tolerance, server-side path canonicalization, one cookie
name, multipart edge cases). None affect building against the documented surface.

## Legal notes

- This documentation was produced for **interoperability** purposes (17 U.S.C. §1201(f);
  EU Directive 2009/24/EC art. 6).
- "Epic", "Epic Haiku", "Hyperspace", "MyChart" are trademarks of Epic Systems
  Corporation; this project uses those names only to identify the analyzed
  software.
- Nothing here is legal advice. If you build on this work for production use,
  have your own counsel review your circumstances — especially access agreements
  with the health systems whose servers you connect to.
- **Health-data warning:** this protocol moves clinical data. Any client built
  on this spec is subject to HIPAA or equivalent health-privacy law if used
  against real patient data. Patient-safety review is required before using
  write operations.

## Contributing

Corrections and additional observations welcome — PRs should state the evidence
behind any protocol claim. All contributions are Apache-2.0.

## License

[Apache-2.0](LICENSE). Copyright 2026 ehr-interop contributors.
