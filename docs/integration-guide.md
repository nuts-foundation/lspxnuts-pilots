# LSPxNuts Medicatie Overdracht Pilot 1 integration guide

> **OUTLINE — content to follow.** This guide gives the exact request/response
> shapes, headers, auth, errors, and ordering a committed vendor codes against.
> The [participation guide](LSPxNuts-participation-guide.md) scopes the work.

## Conventions used in this guide

- Base URLs and placeholders: `<node>` (internal API base), `<subject>`, `<hcp>`,
  `<did>`, `<vc-id>`.
- All examples captured from a real run of the demo stack, not hand-written.
- Pinned versions: `nuts-node` image tag, AET SDK version, PD bundle version —
  stated once here, every example is valid for these tags.
- Internal API security note: these endpoints sit behind the node's private
  interface; the vendor fronts them with their own operator auth (see
  participation guide A.2). Examples omit that layer.

## Required credentials

Before any token can be requested, the wallets have to be provisioned. The table below explains who needs to acquire which credentials and how often.

| Actor                                      | Frequency                    | Action                                                                           | Requirement                                                      |
|--------------------------------------------|------------------------------|----------------------------------------------------------------------------------|------------------------------------------------------------------|
| Vendor                                     | Once                         | Acquire `ServiceProviderCredential`, load into vendor (SP) wallet                | Contract with the AORTA-afsprakenstelsel                         |
| Healthcare provider                        | Once per vendor              | Issue `HealthcareProviderCredential` to care-provider wallet at vendor           | UZI server certificate                                           |
| Healthcare provider                        | Once per vendor              | Issue `ServiceProviderDelegationCredential` to vendor wallet                     | —                                                                |
| Registered care professional               | Once per vendor              | Issue `HealthcareProfessionalDelegationCredential` to caregiver wallet at vendor | UZI smartcard (`zorgverlenerspas`)                               |
| Registered care professional *or* employee | Once per patient, per vendor | Issue `PatientEnrollmentCredential` to caregiver wallet at vendor                | UZI smartcard (`zorgverlenerspas` *or* `medewerkerspas op naam`) |

Once provisioning is complete, the runtime flow (two-VP token request → MEDGEG
query) is the trace in §2; the per-endpoint detail behind each step above is in
§1.

## Acquiring credentials

How to obtain each credential listed above: who issues it, what prerequisites
are needed, and the steps to acquire and load it.

### `ServiceProviderCredential`

_TODO_

### `HealthcareProviderCredential`

_TODO_

### `ServiceProviderDelegationCredential`

_TODO_

### `HealthcareProfessionalDelegationCredential`

_TODO_

### `PatientEnrollmentCredential`

_TODO_

## 1. Endpoint reference

For every endpoint: method + path, path params, required headers + auth,
request body example, success response example, error responses (status, body,
when), preconditions/ordering.

### 1.1 Subject + DID management — `/internal/vdr/v2/...`
> Guide section A.3. Create SP subject, create HCP subject, issue/resolve DID.
- Create subject
- List / resolve subject DIDs
- (any add-service / update calls the pilot needs)
- Precondition for everything in §1.2+.

### 1.2 Load baseline VC into wallet — `POST /internal/vcr/v2/holder/<subject>/vc`
> Guide section B.2.
- Load `ServiceProviderCredential` (SP subject)
- Load `HealthcareProviderCredential` (HCP subject)
- Precondition: subject + DID must exist (§1.1).

### 1.3 Request AET-signed credential — `POST /internal/auth/v2/<subject>/request-credential`
> Guide sections B.3.b, B.3.c.
- `HealthCareProfessionalDelegationCredential` request
- `PatientEnrollmentCredential` request
- The `credential_request_params` field reference lives in §3.
- Workstation/UZI preconditions noted (cross-ref participation guide B.3).

### 1.4 Two-VP service access token — `POST /internal/auth/v2/<hcp>/request-service-access-token`
> Guide section B.4.
- Request body (SP subject id, scope, PD selection)
- Success response (access token, token type, expiry, scope)
- Errors: scope not configured, wallet missing required VC, AS unreachable.
- Precondition: both wallets populated (§1.2, §1.3).

### 1.5 Holder wallet list / delete
> Guide section B.6.
- List credentials in a holder wallet
- Delete a credential by id
- Why it matters: re-issuance leaves duplicates.

### 1.6 Healthcheck
> Guide section A.4.
- Endpoint(s), healthy vs unhealthy response shapes.

### 1.7 MEDGEG FHIR medication-request call
> Part C. The one external (RS) call, not an internal-API call.
- Method + path against the LSP FHIR API
- Attaching the access token from §1.4
- Success response (medication overview shape)
- Errors: token denied, no data, AS/RS unreachable.

## 2. End-to-end flow trace

Single narrative: wallet bootstrap → credential issuance → token request →
MEDGEG response. Numbered steps, each linking to its §1 endpoint, with the
actual request/response at each hop. This is the "sample full flow trace" the
participation guide promises.

## 3. `credential_request_params` field reference

Table of the AET-specific fields used in §1.3: field name, type, required?,
meaning, example. Separate the fields common to both AET credentials from the
ones specific to each (e.g. BSN for patient enrollment).

## 4. Sample wallet states per subject type

- SP subject wallet (after bootstrap + delegation)
- HCP subject wallet (after bootstrap + professional delegation + enrollment)
- What "correctly populated for a token request" looks like.

## 5. TLS / certificate details

> Filled in once #2 is decided. Nuts convention (nuts-node#4156): OAuth2
> endpoints → public cert, data endpoints → PKIoverheid Private cert. Exact
> pilot requirement TBD.

## Appendix: pinned versions

Single table of every pinned tag/version referenced above, kept in sync with the
demo stack.
