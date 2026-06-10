# Active Directory Authentication (LDAPS)

> **Delegating all identity to Active Directory over an encrypted channel**
> osTicket holds no local credentials. Every user and agent authenticates against `mednet.lab` through the osTicket LDAP plugin binding to `dc01` over LDAPS, validated against the internal certificate authority. This is the control that makes the service desk a HIPAA accountability artifact rather than just a ticket tracker.

---

## Design Intent

The service desk maintains no password database of its own. Authentication is delegated entirely to Active Directory, the same identity source used across the lab, so that every action taken in osTicket — a ticket opened, an assignment made, a status changed — is attributable to a named directory identity. There are no anonymous agents and no local fallback accounts.

Two properties were non-negotiable in the design:

- **Encryption in transit.** Directory binds carry credentials, and in a PHI-adjacent environment those must never cross the wire in cleartext. The integration uses LDAPS (LDAP over TLS) on port `636` exclusively — plain LDAP on `389` is not used.
- **Validated trust.** The TLS channel is only meaningful if the certificate it presents is verified. The Debian host is configured to trust the internal CA, `MedNet-RootCA`, so the certificate `dc01` presents is validated rather than blindly accepted.

---

## Authentication Model

osTicket separates two populations, and both are pointed at Active Directory:

| Population | Where they sign in | Backend |
|---|---|---|
| Users (clients) | User portal (site root) | Active Directory via LDAPS |
| Agents (staff) | Agent control panel (`/scp`) | Active Directory via LDAPS |

Identity, group membership, and authentication all originate from the directory. osTicket consumes those identities; it never owns them.

---

## Certificate Trust

LDAPS depends on the host trusting the issuer of the domain controller's certificate. `dc01` presents a certificate issued by the lab's internal authority, `MedNet-RootCA`, so that CA's certificate was exported from the domain controller and imported into the Debian host's trust configuration.

The CA certificate is placed into the system trust store and registered, and the LDAP client library is pointed at it so TLS validation succeeds:

```bash
# CA certificate exported from the DC (MedNet-RootCA), copied to the host
sudo cp mednet-rootca.crt /usr/local/share/ca-certificates/mednet-rootca.crt
sudo update-ca-certificates
```

```conf
# /etc/ldap/ldap.conf — point the LDAP client at the trusted CA
TLS_CACERT      /etc/ssl/certs/ca-certificates.crt
TLS_REQCERT     demand
```

`TLS_REQCERT demand` is the important line: the host requires a valid, trusted certificate and will refuse the connection otherwise. Relaxing this to `never` would "fix" a broken LDAPS bind by disabling the very protection LDAPS exists to provide — so it is left strict, and the trust problem is solved properly by importing the CA.

---

## Plugin Configuration

Authentication is provided by osTicket's LDAP / Active Directory authentication and lookup plugin, configured to bind to the domain controller over LDAPS:

| Setting | Value |
|---|---|
| Directory server | `dc01.mednet.lab` |
| Connection | `ldaps://dc01.mednet.lab:636` |
| Default domain | `mednet.lab` |
| Search base DN | `DC=mednet,DC=lab` |
| Bind account | `svc_osticket@mednet.lab` |
| Authentication backends | LDAP enabled for both clients and agents |

The plugin binds as the service account, locates the submitted identity beneath the search base, and authenticates it against the directory. Because the connection scheme is `ldaps://` on `636`, the entire exchange — bind, lookup, and credential verification — occurs inside the validated TLS channel established above.

---

## Service Account Design

`svc_osticket@mednet.lab` exists solely to perform directory reads on behalf of the service desk, and it is deliberately powerless beyond that:

- **Read-only.** The account can search and read directory objects; it cannot create, modify, or delete them, and it cannot reset passwords.
- **Single purpose.** It is dedicated to this one integration, so its activity in directory logs is unambiguous and its blast radius if compromised is limited to read exposure.
- **Minimum necessary, applied to a machine identity.** The same principle that scopes human agents to their queues scopes this service identity to read-only lookups — least privilege is enforced for non-human accounts as rigorously as for people.

---

## How a Login Flows

1. A user or agent submits their Active Directory credentials at the portal or `/scp`.
2. osTicket opens an LDAPS connection to `dc01.mednet.lab:636`; the host validates the presented certificate against `MedNet-RootCA`.
3. osTicket binds as `svc_osticket` and searches beneath `DC=mednet,DC=lab` for the submitted identity.
4. The directory verifies the supplied password, and osTicket grants the session as either a client or an agent according to the backend.

At no point is a password stored on `itsm01`, and at no point does a credential traverse the network unencrypted.

---

## HIPAA Alignment

This integration is where the service desk earns its compliance framing:

- **Audit Controls (§164.312(b))** — every osTicket action resolves to an authenticated directory identity, producing an attributable per-ticket trail.
- **Information Access Management (§164.308(a)(4))** — access is governed centrally in Active Directory; granting or revoking it is a directory operation, not a local edit on the service desk.
- **Transmission Security (§164.312(e))** — credentials and lookups are confined to a validated LDAPS channel; no cleartext directory traffic exists.
- **Accountability** — the absence of any local or anonymous account means there is no path to an unattributable action.

---

## Validation & Troubleshooting

The integration is confirmed by signing in at both the user portal and `/scp` with Active Directory accounts and observing the sessions resolve to the correct directory identities.

LDAPS integrations fail in a small number of predictable ways, and each was ruled out during bring-up:

- **Untrusted certificate** — if the CA is not imported, the TLS handshake fails outright. Resolved by importing `MedNet-RootCA` and pointing the LDAP client at the trust store rather than weakening `TLS_REQCERT`.
- **Bind failure** — an incorrect bind UPN or a directory account lacking read rights produces an authenticated-bind error. The service account UPN and its read scope were verified against the directory.
- **Empty lookups** — a mistyped search base returns no results even when credentials are valid. The base DN was confirmed against the live directory rather than assumed.
- **Blocked transport** — LDAPS on `636` must be reachable from `itsm01` to `dc01`; connectivity was confirmed before configuring the plugin.

| | |
|:---:|:---:|
| ![osTicket LDAP / Active Directory plugin configuration](../screenshots/ad-authentication_01.png) | ![Successful AD-authenticated agent login](../screenshots/ad-authentication_02.png) |
| *LDAP plugin bound to `dc01` over LDAPS* | *Agent session resolved to an Active Directory identity* |

---

*← Back to [MedNet IT Service Desk](../README.md)*
