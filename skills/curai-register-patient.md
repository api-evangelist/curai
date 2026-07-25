---
name: Register a Curai patient and launch the embedded experience
description: >-
  Onboard a patient into Curai Health via the Partner API and use the returned
  access token to launch the embedded Curai patient experience (Web SDK or
  WebView), then end the service when the engagement is over.
api: openapi/curai-partner-openapi.json
operations:
- Register_Patient_register_patient_post
- End_Service_end_service_post
---

# Register a Curai patient and launch the embedded experience

Use the Curai Partner API to onboard a patient and hand them off to the embedded
Curai telemedicine experience.

## Auth
- Every call sends your partner key in the `X-Api-Key` request header.
- Use the staging gateway `https://gateway.staging.curaihealth.com/partner` with a
  staging key while testing; use `https://gateway.curaihealth.com/partner` in
  production. (See `sandbox/curai-sandbox.yml`.)

## Steps

1. **Register the patient** — `POST /register-patient`
   (`Register_Patient_register_patient_post`). Body: a `patient` object
   (`unique_id`, `first_name`, `last_name`, `date_of_birth`, `sex`,
   `permanent_address` with a valid US `state` enum + `zip_code`; `email`/`phone`
   optional but recommended), plus `primary_account_holder_unique_id` (the same as
   `unique_id` for adults, or the sponsor/guardian id for dependents). Optionally
   pass `metadata` for your own fields (e.g. `conversation_id`, `employer_name`).
   A `201` returns `access_token`, `refresh_token`, and `auth_url`.

2. **Launch the experience** — initialize the Web SDK with
   `Curai.startChatSession({ clientId, accessToken })`, or open the WebView URL
   `https://customer.curaihealth.com?clientId=...&accessToken=...`. See
   `components/curai-components.yml`.

3. **End the service** — when the engagement concludes,
   `POST /end-service` (`End_Service_end_service_post`) with the patient's
   `unique_id` and an `end_date` (RFC 3339 datetime with timezone). A `204`
   confirms.

## Errors
- A `422` returns a FastAPI validation envelope
  (`detail[]` of `loc`/`msg`/`type`) — inspect `loc` to find the bad field.
  Common causes: unsupported `state` value, malformed `email`/`date_of_birth`,
  zip shorter than 5 chars. See `errors/curai-problem-types.yml`.
- A missing/invalid `X-Api-Key` fails authentication (not modeled in the spec).

## Notes
- No idempotency key is supported; do not blindly retry a `register-patient`
  call — reconcile by your `unique_id` first.
- No pagination or list endpoints exist; this is a two-operation onboarding
  surface.
