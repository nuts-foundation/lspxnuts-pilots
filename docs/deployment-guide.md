# LSPxNuts Medicatie Overdracht Pilot 1 deployment guide

This guide covers the infrastructure required for participating in the LSPxNuts pilot.

## 1. Architecture & components

Deployment view. Components in blue are deployed by the vendor for the pilot;
grey are existing systems the vendor already runs or that exist externally.

```mermaid
flowchart TB
  subgraph WS["Workstation — existing (AET ZorgID installed for the pilot)"]
    Browser["User browser<br/>[Browser]<br/>HCP staff doing credential issuance"]
    ZorgID["AET ZorgID<br/>[Installed software]<br/>Smartcard crypto via the card reader"]
    Reader["Smartcard reader<br/>[Hardware]<br/>e.g. HID OMNIKEY 3121"]
  end

  Inbound(["Inbound public traffic<br/>did:web resolution, OAuth2 token requests"])

  subgraph Vendor["Vendor deployment — to be deployed"]
    RP["Reverse proxy<br/>[Container]<br/>Serves the public .nl domain; forwards to the Nuts public API"]
    Node["Nuts node<br/>[Container]<br/>Hosts SP and per-HCP wallets; issues credentials, requests tokens"]
    AETSDK["AET ZORG-ID SDK<br/>[Container]<br/>Issues credentials through the smartcard"]
    DB[("SQL database<br/>[Database]<br/>Node storage")]
    Keys[("Key storage<br/>[Vault / Key Vault / disk]<br/>Signing keys")]
  end

  subgraph Systems["Vendor systems — existing"]
    EHR["EHR<br/>[Existing system]<br/>Requests access tokens, calls MEDGEG"]
    Admin["EHR admin interface<br/>[Existing system]<br/>Drives credential issuance"]
  end

  AETIDP["Central AET IDP<br/>[External, existing]<br/>Authenticates the UZI smartcard"]

  Inbound --> RP
  RP -->|public API| Node
  Node -->|request credential| AETSDK
  AETSDK -->|invoke signing operation| AETIDP
  Node --> DB
  Node --> Keys
  EHR -->|request access token| Node
  Admin -->|manage wallets, start credential issuance| Node
  Browser -->|uses| Admin
  ZorgID -->|reads card| Reader
  ZorgID -->|authenticate session| AETIDP

  subgraph Legend["Legend"]
    direction LR
    L1["To be deployed for the pilot"]
    L2["Existing / external"]
  end

  classDef deploy fill:#cfe3ff,stroke:#3b6ea5,color:#11233a;
  classDef existing fill:#e5e5e5,stroke:#888,color:#222;
  class RP,Node,AETSDK,DB,Keys,ZorgID,Reader,L1 deploy;
  class EHR,Admin,AETIDP,Browser,L2 existing;
```

Components:
- **Reverse proxy** — serves the public `.nl` domain (required for did:web
  resolution), terminates inbound traffic, forwards to the Nuts public API
  (OAuth2 + `.well-known`/did:web).
- **Nuts node** — central component; calls the AET SDK to acquire credentials,
  persists to the SQL database, signs with key storage.
- **AET ZORG-ID SDK** — deployed by the vendor; issues credentials through the
  smartcard, invoking signing operations at the central AET IDP.
- **SQL database**, **key storage** — node backing services.
- **EHR** (existing) — acquires access tokens from the node.
- **EHR admin interface** (existing) — manages wallets and starts credential
  issuance via the node.
- **Workstation** (existing) — runs the user's browser, **AET ZorgID** (the
  installed software that does the UZI smartcard crypto), and a **smartcard
  reader**. ZorgID must be installed on every workstation that performs issuance.
- **Central AET IDP** (external, existing) — workstation AET ZorgID authenticates
  the session against it, and the AET SDK invokes signing operations on it; not
  deployed by the vendor.

## 2. Nuts node deployment

- Docker image: TBD
- Configuration (also see [documentation](https://nuts-node.readthedocs.io/en/stable/pages/deployment/configuration.html)):  
  - Properties:
    - `url`: public base URL of the Nuts node
    - `crypto.storage`: configure, depending on key storage
    - `http.internal.address`: set to `0.0.0.0:8081` for internal API access from outside container
    - `auth.experimental.jwtbearerclient` set to `true` to enable two-VP bearer token flow
    - `storage.sql.connection`: configure, depending on SQL database
  - Files:
    - Mount policy file (TBD: add link) into `/nuts/config/policy/`

## 3. Key storage setup

- Vault / Azure Key Vault configuration
- On-disk option and its at-rest responsibilities
- Key rotation procedure + cadence (vendor decides).

## 4. AET ZORG-ID SDK deployment

> Participation guide B.1.
- Obtaining the image (AET licensing; not redistributed — #10)
- Running it alongside the node: stable HTTPS endpoint reachable from the node
- mTLS / certificate binding / network exposure → AET docs (link, don't copy)
- How the node authenticates to the SDK (#11, TBD)
- Soft-cert dev posture vs production (real UZI/HSM) — #12.

## 5. TLS / certificates

> Filled in once #2 is decided. Nuts convention (nuts-node#4156): OAuth2
> endpoints → public cert, data endpoints → PKIoverheid Private cert.
- Where each cert is configured
- CA trust (AET + others baked into the image).

## 6. Operations

> Participation guide A.4.
- Healthcheck endpoints + what to alert on
- Log routing and verbosity (pilot vs production)
- Backup / restore (DB, keys)
- Upgrade path within the pilot window.

## 7. Demo stack

- `docker compose up`: what it brings up (mock AS, mock issuers, AET dev mode, UI)
- How it differs from a real deployment
- Using it as a pre-flight sanity check.

## Appendix: pinned versions

Single table of every pinned tag/version, kept in sync with the demo stack and
the integration guide.
