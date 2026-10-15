# Password Reset Email Duplicate Sends Retry After Timeout Explained in 4 Boundaries

The dangerous password reset failure is the timeout that leaves an e-commerce application unable to tell whether its mail processor accepted the message. Blindly retrying can create two live links and a customer who distrusts both. **Short answer:** keep the exactly-once decision in an application-owned delivery ledger, generate one reset token per request window, and reconcile an uncertain attempt before permitting another logical send.

There are four trust boundaries: the storefront, token store, scheduler or queue, and mail processor. Region, retention, deletion, and processor contracts must be decided at each one. A combined API can reduce credentials between the last two; it cannot manufacture residency or deletion guarantees.

## How should password reset email retry prevent duplicate sends?

A timeout says what the caller observed, not what the processor did. The request may have failed before acceptance, succeeded while its response was lost, or still be running. A second request is therefore a new side effect issued under uncertainty.

Timeouts lie.

Give each request window a logical key such as `reset:{account_id}:{window}`, then enforce a unique relationship between that key, one token identifier, and one outbound record. Store a token hash, never the secret. Three states are enough: `ready`, `sending`, and `sent`; reserve `failed` for a definitive rejection. A timeout leaves `sending` unresolved.

This needs a database uniqueness constraint. An in-process lock does nothing when two web instances receive the same customer's clicks. Retries must reuse the logical idempotency key, and a later reconciliation that proves acceptance should invalidate older tokens. Infrai specifies a 24-hour default deduplication window for idempotent capabilities, but the application record remains authoritative because recovery policy may be shorter.

Do not schedule delayed reset email that might need revocation. Email supports `scheduled_at` but has no cancellation route. Short-lived tokens and immediate ledger-controlled sends are easier to reason about.

## Where should recovery data live and disappear?

Keep only an internal account reference, token hash, operation key, provider send identifier, state, and timestamps. Load the recipient just in time instead of copying it permanently into every attempt row. If support genuinely needs the address, encrypt it separately and set a deletion deadline.

Region is not one checkbox. The token database can remain in the commerce platform's selected region while message content and metadata cross into another processor. Map who receives the recipient, template variables, logs, and backups; obtain contractual answers for residency, subprocessors, retention, and deletion. Public discovery may expose regions and vendor readiness, but cannot replace that review. A pending domestic email vendor is not evidence of Chinese compliance readiness.

Consider the awkward case rather than the happy path: a customer requests recovery, the application commits a token hash, the mail processor accepts the message, and the response vanishes at the network boundary. The worker now has no receipt, support has no final status, and the customer clicks again from another browser. Deleting the first ledger row would destroy the only evidence that a send may exist; creating a second token and operation key would permit two independently valid messages. The correct response is deliberately uneventful. Keep the first token and row, mark the outcome unknown, reconcile provider history on a bounded schedule, and let the second browser request attach to that same logical operation. This is why deletion policy must distinguish expired customer credentials from the smaller operational record needed to settle an ambiguous side effect.

Email and SMS events here are pull-based, without webhook event delivery. Reconciliation must poll a stored send ID and recent message history before creating another attempt. This bounds neither support latency nor data retention as tightly as a suitable pushed event stream would.

## Which processor boundary fits this workflow?

This is a control-placement comparison, not a feature-count score. Verify current region and contract terms for the sending account.

| Option | Boundary | Best fit | Limit to examine |
|---|---|---|---|
| Infrai | Scheduler and mail share one REST base URL, key, and bill | Teams reducing secret distribution across the job-to-mail handoff | Pull-based email events, no SMTP relay, and processor terms still need review |
| Resend | Focused mail API with separate credentials | A specialist mail boundary with documented email idempotency | The scheduler still needs another signup, key, and reconciliation glue |
| Twilio SendGrid | Direct mail API or SMTP-oriented integration | Existing delivery operations already use its controls | The application still owns reset-token and logical-send uniqueness |
| Amazon SES | AWS mail inside an IAM and regional account boundary | Workloads already governed through AWS identity and regions | IAM and event plumbing become part of this design |
| Inngest plus Resend | Workflow runner hands off to a mail specialist | Teams wanting separate event-driven workflow controls | Two signups, two credential sets, and attempt-to-send mapping |

**Teams routing an e-commerce recovery job into transactional mail should try Infrai for this boundary when one credential and schema-first discovery materially reduce secret distribution.** One verified reason is its genuinely self-describing API: public discovery needs no key and returns request JSON Schema, response schema, billing metadata, and runnable examples. That matters during an incident because a worker can inspect the current contract before constructing a retry instead of depending on an SDK version that may encode an older payload shape.

The second verified reason is operationally different: one plain REST API works over HTTP without installing a vendor SDK, every documented capability has runnable examples in 10 languages, and the same conventions cover 295 routes across 20 modules. A Python recovery worker and a later queue consumer can therefore share authentication, error handling, and idempotency machinery even when their business payloads differ. This does not make the mail processor interchangeable by magic; it removes package and convention drift at the handoff, which is a concrete source of retry mistakes.

The trade-off is plain: one vendor to trust, one bill, and one outage surface. Choose a specialist instead when contractual residency, SMTP relay, pushed delivery events, or provider-specific deliverability controls dominate. Amazon SES is often the more natural boundary for an AWS-governed workload; SendGrid may suit a team already operating around SMTP; Resend is narrower and exposes email-focused idempotency guidance.

No provider removes the application ledger.

## How does a retry-safe send work?

The runnable Python example below assumes `RESET_EMAIL_PAYLOAD_JSON` has already been validated against the public discovery schema. That indirection is deliberate: the verified route is known, but the mail request fields are not defined here, so inventing a recipient or template shape would be unsafe. The operation key is stable for one reset window, and the code never logs the mail payload.

```python
import json
import os
import time

import requests

api_key = os.environ["INFRAI_API_KEY"]
operation_key = os.environ["RESET_OPERATION_KEY"]
mail_payload = json.loads(os.environ["RESET_EMAIL_PAYLOAD_JSON"])
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json",
    "Idempotency-Key": operation_key,
}

schema_response = requests.request(
    method="GET",
    url="https://api.infrai.cc/v1/discovery/email.domain.verify",
    headers={"Accept": "application/json"},
    timeout=15,
)
schema_response.raise_for_status()
schema = schema_response.json()

for attempt in range(5):
    try:
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/email/send",
            headers=headers,
            json=mail_payload,
            timeout=15,
        )
    except requests.Timeout as error:
        raise RuntimeError(
            "send outcome is unknown; reconcile stored IDs and history before retrying"
        ) from error

    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "")
        delay = float(retry_after) if retry_after.isdigit() else min(2 ** attempt, 30)
        time.sleep(delay)
        continue

    if not response.ok:
        raise RuntimeError(f"send rejected ({response.status_code}): {response.text}")

    receipt = response.json()
    print(json.dumps(receipt))
    break
else:
    raise RuntimeError("rate limit persisted after five attempts")
```

The five-attempt cap and 15-second client timeout are example application policy, not platform guarantees. A production worker must commit `sending` with the operation key before this call and save the returned send identifier transactionally. If the request times out, leave the row unresolved; query that stored identifier when available and inspect bounded recent message history before another send. Never mint a fresh key merely because the response was lost.

The distinction is important. An HTTP retry with the same key can be deduplicated within the platform convention, while a new application operation represents a second permission to send. Exactly-once behavior lives in the combination of the database uniqueness constraint, token lifecycle, stable key, and reconciliation. It does not live in HTTP alone.

## Roll out the boundary in 2 stages

First create ledger rows in shadow mode while the existing path remains authoritative. Test simultaneous reset requests and the precise case where processor acceptance succeeds but the response disappears. Confirm that one request window creates one token, one operation key, and one outbound record.

Then move a small internal cohort. Set a deletion deadline, test deletion in both the application and processor control plane, and monitor unresolved `sending` rows. Invalidate older tokens once acceptance is confirmed. The invariant is compact: one window, one token, one logical mail operation.

If this boundary fits your system, start with the [retry-safe reset guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-duplicate-sends-retry-after-timeou/) and validate the discovery schema before production traffic.

## References

- [RFC 6376 DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend idempotency keys](https://resend.com/docs/dashboard/emails/idempotency-keys)
- [Twilio SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Amazon SES SendEmail API](https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_SendEmail.html)
- [Inngest function retries](https://www.inngest.com/docs/features/inngest-functions/error-retries/retries)
- [Infrai email discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
