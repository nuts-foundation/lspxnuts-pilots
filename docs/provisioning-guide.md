# LSPxNuts Medicatie Overdracht Pilot 1 provisioning guide

This guide explains how to set up your service provider and client healthcare provider identities and issue the credentials needed to query the LSP — and which APIs to invoke.

## Conventions used in this guide

- `<node>`: Nuts node internal API base URL (e.g. `http://nutsnode:8081`), as reachable by the EHR/admin UI.
- `<aet-sdk>`: AET SDK base URL (e.g. `http://aet:5003`), as reachable by the Nuts node.
- `<app-redirect-uri>`: URL to where the user browser will be redirected after successful credential issuance.
- `<sp-subject>`: Service Provider subject as registered in the Nuts node
- `<hcp-subject>`: Healthcare Provider subject as registered in the Nuts node
- `<sp-did>`: Service Provider's `did:web` DID (e.g. `did:web:example.nl:iam:serviceprovider`), required in some issuance requests
- `<hcp-did>`: Healthcare Provider's `did:web` DID, required in some issuance requests
- `<hcp-ura>`: Healthcare Provider's URA-identifier (e.g. `12345678`).
- `<patient-bsn>`: Patient's BSN (e.g. `12345678`).

> 🚧 **TBD:** the `gis-nl.example` `@context` URLs and the `authorizationRule`
> value in the examples are placeholders from the demo; update them once the
> GIS-NL spec is finalized.

## Required credentials

Before any token can be requested, the wallets have to be provisioned. The table below explains who needs to acquire which credentials and how often.

| Actor                                      | Frequency                    | Action                                                                           | Requirement                                                      |
|--------------------------------------------|------------------------------|----------------------------------------------------------------------------------|------------------------------------------------------------------|
| Vendor                                     | Once                         | Acquire `ServiceProviderCredential`, load into vendor (SP) wallet                | Contract with the AORTA-afsprakenstelsel                         |
| Healthcare provider                        | Once per vendor              | Issue `HealthcareProviderCredential` to care-provider wallet at vendor           | UZI server certificate                                           |
| Healthcare provider                        | Once per vendor              | Issue `ServiceProviderDelegationCredential` to vendor wallet                     | —                                                                |
| Registered care professional               | Once per vendor              | Issue `HealthcareProfessionalDelegationCredential` to caregiver wallet at vendor | UZI smartcard (`zorgverlenerspas`)                               |
| Registered care professional *or* employee | Once per patient, per vendor | Issue `PatientEnrollmentCredential` to caregiver wallet at vendor                | UZI smartcard (`zorgverlenerspas` *or* `medewerkerspas op naam`) |

The steps to create the identities and acquire each credential are below. Once
provisioning is complete, request a token and query the LSP — see the
[data querying guide](data-querying-guide.md).

## Creating Identities

Every wallet belongs to a *subject* in the Nuts node. Before loading any
credential you create:

- one **SP subject** for the vendor itself, and
- one **HCP subject** per care-provider customer.

Creating a subject also generates its `did:web` DID, derived from the node's
public `.nl` URL.

### Create a subject

Provide the subject identifier you want to use (omit `subject` to have the node
generate a UUID). The identifier must match `[a-zA-Z0-9._-]+`:

Request:

```http request
POST <node>/internal/vdr/v2/subject
Content-Type: application/json

{
  "subject": "<subject>"
}
```

It'll respond with the created subject and its DID document(s):

```json
{
  "subject": "<subject>",
  "documents": [
    {
      "id": "did:web:example.nl:iam:some-identifier"
    }
  ]
}
```

The `did:web` DID under `documents[].id` is derived from the node's public `.nl`
URL. Store the `subject` value; you'll pass it to the credential and token
endpoints below.

Depending on the node's configuration the response may contain more than one
document — for example, an additional `did:nuts` DID. Some issuance requests need
the `did:web` DID itself (`<sp-did>` / `<hcp-did>`); take it from
`documents[].id` here, or look it up later for a known subject with:

```http request
GET <node>/internal/vdr/v2/subject/<subject>
```

```json
["did:web:example.nl:iam:some-identifier"]
```

As for the value of `<subject>`;

- For your service provider, it's recommended to use a static key (e.g. `serviceprovider`)
- For each of your healthcare providers, either:
  - Pass in an existing key which associates it with the healthcare provider (e.g. internal identifier), or
  - Don't pass `"subject"`, but store the returned `subject` field (a random GUID).

## Acquiring credentials

How to get each credential: who issues it, what prerequisites
are needed, and the steps to acquire and load it.

### `ServiceProviderCredential`

The `ServiceProviderCredential` is issued by AORTA/LSP to the Service Provider. It's requested by an authorized employee of the vendor (e.g. system administrator).

Use OpenID4VCI to issue the credential to the Nuts node:

```http request
POST <node>/internal/auth/v2/<sp-subject>/request-credential
Content-Type: application/json

{
  "wallet_did": "<sp-did>",
  "issuer": "TODO",
  "redirect_uri": "<app-redirect-uri>",
  "authorization_details": [
    {
      "type": "openid_credential",
      "credential_configuration_id": "ServiceProviderCredential"
    }
  ]
}
```

> 🚧 **TBD:** AORTA/LSP's ServiceProviderCredential issuer isn't available; blanks will be filled in due time.

### `HealthcareProviderCredential`

The `HealthcareProviderCredential` is the healthcare provider's organization
identity. Unlike the other credentials it is **self-issued from the HCP's UZI
server certificate**: the issuer is a `did:x509` derived from that certificate
(the URA is read from the certificate's SAN, not passed in), so it is generated
with the [`go-didx509-toolkit`](https://github.com/nuts-foundation/go-didx509-toolkit)
CLI rather than a node endpoint.

Generate the credential from the UZI server certificate:

```shell
didx509-toolkit vc \
  --type HealthcareProviderCredential \
  <cert-chain.pem> <signing-key.pem> "<ca-fingerprint-dn>" <hcp-did>
```

- `<cert-chain.pem>` — the full UZI **server** certificate chain (PEM).
- `<signing-key.pem>` — the unencrypted private key for that certificate (PEM;
  an Azure Key Vault URL is also accepted).
- `<ca-fingerprint-dn>` — Subject DN of the CA in the chain used for the
  `did:x509` ca-fingerprint (e.g. `"CN=UZI-register Private Server CA ..."`).
- `<hcp-did>` — the HCP subject's `did:web` DID (the credential subject).

The toolkit prints a JWT-encoded VC, signed RS256 (UZI keys are RSA, which is
what AORTA expects). Load it into the HCP wallet:

```http request
POST <node>/internal/vcr/v2/holder/<hcp-subject>/vc
Content-Type: application/json

"<the JWT VC printed by the toolkit>"
```

A `204 No Content` confirms the credential is in the HCP wallet.

### `ServiceProviderDelegationCredential`

The `ServiceProviderDelegationCredential` is issued by an authorized employee (e.g. system administrator) from the healthcare provider,
to the SP wallet. This is typically done through an administration UI of the healthcare provider/wallet.

In this pilot, the healthcare provider probably doesn't have a wallet separate from the vendor's,
meaning the vendor can self-issue the credential.

> 🚧 **Open question:** do we want a healthcare provider employee to issue this credential (even when the vendor could do it themselves), to show the process?

Issue the credential from the healthcare provider's DID:

```http request
POST <node>/internal/vcr/v2/issuer/vc
Content-Type: application/json

{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "http://gis-nl.example/context.jsonld"
  ],
  "type": ["VerifiableCredential", "ServiceProviderDelegationCredential"],
  "issuer": "<hcp-did>",
  "expirationDate": "2027-01-01T00:00:00Z",
  "credentialSubject": {
    "id": "<sp-did>",
    "@type": "ServiceProvider",
    "hasDelegation": {
      "@type": "Delegation",
      "delegatedBy": {
        "@type": "HealthcareProvider",
        "identifier": [
          {
            "@type": "Identifier",
            "system": "http://fhir.nl/fhir/NamingSystem/ura",
            "value": "<hcp-ura>"
          }
        ]
      },
      "scope": {
        "@type": "DelegationScope",
        "authorizationRule": "https://gis-nl.example/rules/medgeg-delegation",
        "authorizedActions": ["aorta.contextcode.MEDGEG"]
      }
    }
  },
  "format": "jwt_vc"
}
```

- `issuer` — the healthcare provider's DID (the delegating party). Self-issued
  in this pilot, so it must be a DID the node controls.
- `credentialSubject.id` — the SP's DID (the delegated party).
- `delegatedBy.identifier` — the healthcare provider's URA.
- `format` — `jwt_vc`, matching the pilot's JWT credentials.

The node responds with the issued credential, a JWT-encoded VC:

```json
"eyJhbGciOiJFUzI1Ni…<header>.<payload>.<signature>…"
```

Load it into the SP wallet:

```http request
POST <node>/internal/vcr/v2/holder/<sp-subject>/vc
Content-Type: application/json

<the verifiable credential returned above>
```

A `204 No Content` confirms the credential is in the SP wallet.

### `HealthcareProfessionalDelegationCredential`

The `HealthcareProfessionalDelegationCredential` is issued by the healthcare professional to the HCP wallet,
using their UZI smartcard ("zorgverlenerspas").

Use OpenID4VCI to issue the credential to the Nuts node:

```http request
POST <node>/internal/auth/v2/<hcp-subject>/request-credential
Content-Type: application/json

{
  "wallet_did": "<hcp-did>",
  "issuer": "<aet-sdk>/v1/openid",
  "redirect_uri": "<app-redirect-uri>",
  "profile": "aet",
  "authorization_details": [
    {
      "type": "openid_credential",
      "credential_configuration_id": "HealthcareProfessionalDelegationCredential"
    }
  ],
  "credential_request_params": {
    "credential_subject_data": {
      "@context": "http://gis-nl.example/context/v1",
      "@type": "HealthcareProvider",
      "hasEnrollment": {
        "issuedTo": {
          "@type": "HealthcareProvider",
          "identifier": {
            "@type": "Identifier",
            "system": "http://fhir.nl/fhir/NamingSystem/ura",
            "value": "<hcp-ura>"
          }
        }
      }
    }
  }
}
```

> **Note:** we make the `wallet_did` paramter optional, see [#4365](https://github.com/nuts-foundation/nuts-node/issues/4365). Otherwise, you'll have to query the subject's DIDs to find the right one to issue to.

The Nuts node will respond with the URL to which the user browser must be redirected:

```json
{
  "redirect_uri": "https://example.com/credential-issuer",
  "session_id": "some-session-id"
}
```

It's not required to store the session ID.

### `PatientEnrollmentCredential`

The `PatientEnrollmentCredential` is issued per patient to the HCP wallet.
It's either issued by:
- the healthcare professional using their UZI smartcard ("zorgverlenerspas"), or
- a healthcare provider employee using their UZI smartcard ("medewerkerspas op naam").

Use OpenID4VCI to issue the credential to the Nuts node:

```http request
POST <node>/internal/auth/v2/<hcp-subject>/request-credential
Content-Type: application/json

{
  "wallet_did": "<hcp-did>",
  "issuer": "<aet-sdk>/v1/openid",
  "redirect_uri": "<app-redirect-uri>",
  "profile": "aet",
  "authorization_details": [
    {
      "type": "openid_credential",
      "credential_configuration_id": "PatientEnrollmentCredential"
    }
  ],
  "credential_request_params": {
    "credential_subject_data": {
      "@context": "http://gis-nl.example/context/v1",
      "@type": "HealthcareProvider",
      "hasEnrollment": {
        "issuedTo": {
          "@type": "HealthcareProvider",
          "identifier": {
            "@type": "Identifier",
            "system": "http://fhir.nl/fhir/NamingSystem/ura",
            "value": "<hcp-ura>"
          }
        },
        "patient": {
          "@type": "Patient",
          "identifier": {
            "@type": "Identifier",
            "system": "http://fhir.nl/fhir/NamingSystem/bsn",
            "value": "<patient-bsn>"
          }
        }
      }
    }
  }
}
```