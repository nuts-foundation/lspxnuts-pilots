# LSPxNuts Medicatie Overdracht Pilot 1 data querying guide

This guide explains how to request an access token from your Nuts node and use
it to query the LSP FHIR endpoint. It assumes the wallets are already
provisioned (see the [provisioning guide](provisioning-guide.md)).

## Conventions used in this guide

- `<node>`: Nuts node internal API base URL (e.g. `http://nutsnode:8081`), as reachable by the EHR.
- `<sp-subject>`: Service Provider subject as registered in the Nuts node.
- `<hcp-subject>`: Healthcare Provider subject as registered in the Nuts node.
- `<auth-server>`: the LSP authorization server (e.g. `https://ontmedmij-issuer.vzvz.nl/nuts`).
- `<access-token>`: the access token returned by the token request below.
- `<patient-bsn>`: BSN of the patient whose data is queried (e.g. `999911234`).

## 1. Request an access token

The Service Provider requests a service access token on behalf of one of its
healthcare providers. Including `service_provider_subject_id` triggers the
two-VP flow: the node builds one presentation for the HCP subject (in the path)
and one for the SP subject.

```http request
POST <node>/internal/auth/v2/<hcp-subject>/request-service-access-token
Content-Type: application/json

{
  "authorization_server": "<auth-server>",
  "scope": "patient/MedicationRequest.s?category=http://snomed.info/sct|33633005",
  "token_type": "Bearer",
  "service_provider_subject_id": "<sp-subject>",
  "credential_selection": {
    "patient_bsn": "<patient-bsn>"
  }
}
```

`credential_selection.patient_bsn` selects the `PatientEnrollmentCredential` for the patient being queried.

The node responds with the access token:

```json
{
  "access_token": "<access-token>",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "patient/MedicationRequest.s?category=http://snomed.info/sct|33633005"
}
```

> **Note:** the access token is scoped to this patient (via `patient_bsn`) and
> this query (via `scope`). It can't be used for other patients or other
> queries; request a new token for each.

## 2. Query the LSP FHIR endpoint

Call the LSP FHIR endpoint directly, passing the access token as a bearer token.
The token is opaque to your software; just attach it.

```http request
GET https://ontmedmij-inlog.vzvz.nl/aortagtk/fhir/stu3/v1/MedicationRequest?category=http://snomed.info/sct%7C16076005
Authorization: Bearer <access-token>
Accept: application/fhir+json, application/json
```

The LSP responds with a FHIR `Bundle` of `MedicationRequest` resources.

> 🚧 **TODO:** the `MedicationRequest` query above is one example. MEDGEG supports
> a limited, fixed set of queries; document the full set (with matching scopes)
> for this pilot here.
