# LSPxNuts Medicatie Overdracht Pilot 1 — documentation

Docs for Nuts vendors participating in the pilot. Read them in this order:

| # | Document | Read when | Audience                     |
|---|----------|-----------|------------------------------|
| 1 | [Participation guide](LSPxNuts-participation-guide.md) | Deciding whether and how to participate | Decision makers, tech leads  |
| 2 | [Deployment guide](deployment-guide.md) | Standing up the infrastructure | Ops / platform engineers     |
| 3 | [Provisioning guide](provisioning-guide.md) | Setting up SP and HCP identities and credentials | Software engineers           |
| 4 | [Data querying guide](data-querying-guide.md) | Querying the LSP for data | Software engineers |
| 5 | [UX considerations](ux-considerations.md) | Designing the issuance and query UX | Product / UX designers |

## Note conventions

Two kinds of callouts appear in these guides:

- `> **Note:** …` — guidance kept for the reader.
- `> 🚧 **TBD/TODO/Open question:** …` — editorial callouts to resolve before the
  docs are final. Inline gaps are marked `TODO`/`TBD`. Search for 🚧, `TODO`, or
  `TBD` to find everything still open.

## Glossary

| Term                                                              | Meaning                                                                                                                           |
|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| **Nuts node**                                                     | The software a vendor hosts; holds wallets, issues credentials, and requests access tokens.                                       |
| **Subject** (Nuts node subject)                                   | An identity managed in the Nuts node — the vendor's SP, or one per HCP customer. Each subject owns one or more DIDs and a wallet. |
| **Wallet**                                                        | The credential store associated with a subject.                                                                                   |
| **Service Provider (SP)**                                         | The vendor's own identity/role in the pilot; acts as OAuth requestor and client.                                                  |
| **Healthcare Provider (HCP)**                                     | A care-provider customer of the vendor. The vendor's Nuts node hosts one subject per HCP.                                         |
| **AET ZORG-ID SDK** (AET SDK)                                     | Vendor-hosted component that issues credentials through the UZI smartcard.                                                        |
| **AET ZorgID**                                                    | Desktop software installed on the workstation that performs the UZI smartcard crypto via the card reader.                         |
| **Central AET IDP**                                               | AET's existing identity provider; the SDK and workstation ZorgID communicate with it. Not deployed by the vendor.                 |
| **MEDGEG**                                                        | The medication-overview service queried at the LSP.                                                                               |
| **Verifiable Credential (VC)** / **Verifiable Presentation (VP)** | A signed credential, and the presentation that wraps one or more credentials for an access-token request.                         |
| **Two-VP flow**                                                   | The RFC 7523 jwt-bearer flow that presents two VPs (SP and HCP) to obtain a service access token.                                 |
| **DID** / **did:web**                                             | Decentralized Identifier. The pilot uses the `web` method, resolved over the node's public `.nl` URL.                             |