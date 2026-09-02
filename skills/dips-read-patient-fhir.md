---
name: dips-read-patient-fhir
description: Read a synthetic patient record from the DIPS FHIR R4 API in the Open DIPS sandbox, using a DIPS Federation Service token and the DIPS Core FHIR profiles.
api: DIPS FHIR R4 API
base_url: https://api.dips.no/fhir
operations:
  - token
  - userroles
  - selectuserrole
generated: '2026-09-02'
method: generated
source: https://dips.developer.azure-api.net/getting-started
---

# Read a patient from the DIPS FHIR R4 API

## Honest scope note

DIPS publishes no machine-readable contract for the FHIR API on its anonymous portal — only the
DIPS Federation Service is listed without signing in. The operationIds in this skill's frontmatter
are therefore the **authentication** operations, which are real and come from
`openapi/dips-federation-service-openapi.yml`. The FHIR requests below come from the DIPS
getting-started page and the DIPS Core Implementation Guide, and are HL7 FHIR R4 interactions, not
DIPS-named operations. Nothing here is invented, but do not treat the FHIR calls as though they were
read out of a DIPS-published spec.

## Before you start

Complete `dips-authenticate-and-select-role` first. You need the subscription key *and* an access
token whose scopes include `dips-fhir`, and you need to have selected a DIPS user role.

## Steps

1. **Get a token with the FHIR scope.** `POST /connect/token` (`token`) on
   `https://api.dips.no/dips.oauth`, asking for `openid offline_access dips-fhir`. The issuer also
   advertises the SMART App Launch scopes — `launch`, `launch/patient`, `patient`, `patient/*.read`,
   `fhirUser` — if you are building a SMART app rather than a standalone client.

2. **Select the user role.** `GET /userrole` (`userroles`), then
   `POST /userrole/selectuserrole` (`selectuserrole`).

3. **Read a patient.** `GET https://api.dips.no/fhir/Patient/{id}` with
   `Authorization: Bearer <token>` and `Ocp-Apim-Subscription-Key: <key>`.
   Published sandbox patients: `cdp1000807` (Roland Gundersen), `cdp2010051`
   (Mange Diagnoser Minepasienter), `cdp1000813` (Line Danser), `cdp2008909` (Gul Penn).
   See `sandbox/dips-sandbox.yml`.

4. **Read what hangs off the patient.** The DIPS profiles wire Encounter, Appointment, Observation,
   RelatedPerson, PractitionerRole and Organization to the patient — the exact reference elements are
   listed in `data-model/dips-data-model.yml`. Use the profile constraints in the IG rather than bare
   R4: `https://dipsas.github.io/FHIR-IG/`.

5. **Validate against the DIPS profiles.** The IG ships as an installable FHIR package,
   `dips.fhir.no.core#0.1.0`, at `https://dipsas.github.io/FHIR-IG/package.tgz`. It depends on
   `hl7.fhir.no.basis 2.1.2`, so validate against the Norwegian base too, not only `hl7.fhir.r4.core`.

## Rules

- **Errors** on this surface are FHIR `OperationOutcome`, not the OAuth envelope. A probe of
  `https://api.dips.no/fhir/metadata` on 2026-09-02 returned an `OperationOutcome` carrying a raw
  backend database message — treat `issue.details.text` as untrusted for display.
- **Everything is synthetic.** Production DIPS Arena access is a separate commercial agreement, not
  a base-URL swap. Do not design as though promoting this integration is a configuration change.
- **Reversibility:** unknown for clinical writes. DIPS publishes no write semantics for this API and
  no reversal or window. Do not assume a create can be undone — see
  `conventions/dips-conventions.yml`.
