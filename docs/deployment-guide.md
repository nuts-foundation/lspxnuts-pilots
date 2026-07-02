# LSPxNuts Medicatie Overdracht Pilot 1 deployment guide

This guide explains the infrastructure required for participating in the LSPxNuts pilot.

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
    - 
    - `storage.sql.connection`: configure, depending on SQL database
  - Files:
    - Mount policy file (TBD: add link) into `/nuts/config/policy/`

## 3. ZORG-ID SDK deployment

- Get the image from VZVZ by [registering as a software vendor](https://vzvz.atlassian.net/helpcenter/zorg-id/portal/11), via developer support.
- [Register OIDC client through VZVZ](https://vzvz.atlassian.net/helpcenter/zorg-id/portal/11/group/377/create/1703) for interacting with ZORG-ID SDK
- Running it alongside the node, HTTP (`:5003`) must be accessible from the Nuts node.
  - Configuration:
    - **Note:** if you alter `appsettings.json`, take it from [assets/appsettings.json](assets/appsettings.json) and mount it;
      - `./appsettings.json:/app/Assets/appsettings.json:ro`.
    - For DeveloperMode (enabling self-issued UZI certificates):
      - Mount PFX file of fake UZI CA (TBD: add link):
        - `./pki/fake-uzi-ca.pfx:/app/Assets/SoftCertificates/fake-uzi-ca.pfx:ro`
      - Enable DeveloperMode
        - Either in `Assets/appsettings.json` under `OpenID4VC`, set `DeveloperMode` to `true`
        - Or using environment variables:
          ```yaml
            environment:
              OpenID4VC__DeveloperMode: "true"
          ``` 
      - Configure fake UZI CA
        - Either in `Assets/appsettings.json` under `Sdk.SoftCertificates`:
          ```json
          {
            "FileName": "fake-uzi-ca.pfx",
            "Password": "",
            "Description": "Test Zorgverlener Lsp Demo (UZI type Z, signed by uzi-did-x509-issuer test_ca)"
          }
          ```
        - Or using environment variables:
          ```yaml
            environment:
              Sdk__SoftCertificates__1__FileName: "fake-uzi-ca.pfx"
              Sdk__SoftCertificates__1__Password: ""
              Sdk__SoftCertificates__1__Description: "Test Zorgverlener Lsp Demo (UZI type Z, signed by uzi-did-x509-issuer test_ca)"
          ``` 
    - Configure ZORG-ID SDK credential issuer metadata:
      - Take [assets/metadata_config.json](assets/metadata_config.json)
      - Replace `aet:5003` with the ZORG-ID SDK Docker container host/port, so it can be reached by the Nuts node (`credential_issuer`, `credential_endpoint`)
      - Mount it; `./metadata_config.json:/app/Assets/OpenId4VC/metadata_config.json:ro`

## 4. TLS / certificates

TBD