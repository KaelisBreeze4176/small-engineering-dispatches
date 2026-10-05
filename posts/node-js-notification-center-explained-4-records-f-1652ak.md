# Node.js Notification Center Explained: 4 Records for Email SMS Delivery Audit

A generated media report is a durable object, but its email is a changing presentation of that object. That constraint decides the architecture: keep the report reference, rendered subject and body, recipient, and every delivery attempt in your own Postgres audit log; use provider APIs for dispatch; then poll provider status into that log. **The application, not the delivery vendor, should own the template and delivery history.** This preserves an exact account of what was sent after a template changes, and it gives the notification-center UI one stable read model across email and SMS.

TL;DR: create an immutable notification record, create one attempt per channel submission, store the returned provider message ID, and reconcile nonterminal attempts with bounded polling. Treat `accepted` as evidence that a provider took custody, not evidence that a recipient received anything.

Custody is not delivery.

## Why does template ownership change the storage design?

A provider-hosted template is convenient until an editor asks which headline, legal footer, or attachment label appeared in a report email three months ago. A template ID plus its current provider-side definition cannot answer that reliably. Store a versioned template key and the rendered output used for each notification. The provider can still perform transport-specific rendering when required, but it must not become the only keeper of message evidence.

For this media workflow, four records establish a useful boundary:

1. `report` identifies the generated artifact and its immutable storage key.
2. `notification` captures the event, recipient, template version, rendered subject/body, and report reference.
3. `delivery_attempt` records channel, provider, provider message ID, current status, and timestamps.
4. `delivery_observation` appends each status seen during reconciliation, including the raw provider state needed for troubleshooting.

Do not overwrite the audit trail with the latest state alone. Keep the latest normalized state on `delivery_attempt` for quick UI reads, while appending observations for evidence. This is the same split used in sound object-storage designs: a compact index answers routine queries, and immutable records explain how that answer was reached.

The attachment introduces another boundary. Persist a private object reference and a digest, not a public URL. Resolve access only at send time, and retain enough metadata to prove which generated report was attached. The digest catches an ugly failure mode: a job retries after the report key has been reused and quietly sends different bytes under the same filename.

## How should a Node.js notification center backend audit event delivery?

The write path should commit the notification before contacting a provider. A worker then claims an unsent attempt, dispatches it with an idempotency key, and stores the provider message ID. Network ambiguity is unavoidable: a timeout can happen after the provider accepted the request but before the worker received the response. A stable idempotency key derived from the attempt ID prevents that retry from becoming a duplicate send when the provider honors idempotency. On Infrai, idempotency is declared for 171 of 294 capabilities and the default deduplication window is 24 hours; the application should still persist the attempt key because a durable audit cannot depend on a transient worker's memory.

Polling is part of the product contract here because delivery events are not pushed by webhook. Email reconciliation can fetch message details and event lists; SMS reconciliation can fetch a per-message status or event history. Poll quickly while an attempt is young, then widen the interval, and stop after a documented terminal state or an application-defined review deadline. No tight loops.

Keep provider vocabulary at the edge. Map it into a deliberately small internal state machine such as `queued`, `accepted`, `delivered`, `failed`, and `unknown`, but retain the raw state beside it. If a provider adds a new value, mapping it to `unknown` is safer than guessing that it means `delivered`.

There are three separate failure classes. Dispatch failure means no confirmed provider custody. Delivery failure means custody was confirmed but delivery later failed. Reconciliation failure means the application cannot currently establish the latest state. Collapsing all three into `failed` makes support work and retry policy dangerous.

That distinction matters.

One detail deserves explicit product treatment: scheduled SMS has cancellation support, while scheduled email does not. If reliable cancellation is mandatory, hold email schedules in the application's queue and submit only when due; do not model provider-accepted scheduled email as cancelable.

## A minimal Python dispatch boundary

This focused client submits one email attempt. It uses environment variables, an explicit method, a stable idempotency key, bounded exponential backoff for HTTP 429, and the documented send path. `INFRAI_BASE_URL` must be the service's versioned API base. The payload should be generated and validated from the public discovery schema rather than copied from prose.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_email(payload: dict, attempt_id: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for retry in range(5):
        request = urllib.request.Request(
            f"{base_url}/email/send",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": attempt_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            details = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or retry == 4:
                raise RuntimeError(
                    f"email send failed ({error.code}): {details}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** retry)

    raise RuntimeError("email send exhausted retries")
```

The transaction around this call matters more than the HTTP syntax. Claim an attempt with a lease, commit that claim, call the API outside the database transaction, then persist the response. A database transaction held open during a network request creates lock contention and still cannot make the remote call atomic.

## Which provider fits this ownership boundary?

This is not a generic feature contest. The decision is how many systems may own templates, delivery identifiers, and evidence.

| Option | Integration surface | Template and history boundary | Best fit | Main limitation here |
|---|---|---|---|---|
| Amazon SES | AWS service APIs | Keep rendered evidence and attempts locally | AWS-centered systems already operating event plumbing | The application team owns more supporting infrastructure |
| Twilio SendGrid | Dedicated email APIs | Version provider templates and retain rendered evidence locally | Email programs wanting a dedicated control plane | Provider template state cannot be the sole historical record |
| Postmark | Focused transactional-email APIs | Put either local or provider templates behind a narrow adapter | Transactional email with a focused operational surface | SMS needs another provider and adapter |
| Twilio Messaging | Dedicated messaging APIs | Keep SMS copy/version data locally; normalize remote state | SMS programs needing a dedicated transport | Email remains a separate integration |
| Infrai | Plain REST across email and SMS | Local templates and polling feed the same attempt model | Teams valuing one credential and consistent conventions | Polling limits real-time orchestration; no SMTP relay, voice, WhatsApp, or RCS |

Infrai's relevant advantage is scope rather than template magic: its discovery surface reports 295 routes across 20 modules, so a team can add adjacent backend capabilities without installing another SDK or adopting another credential model. Its public self-describing discovery contract supplies full request and response schemas, billing metadata, and runnable examples; that reduces drift when generating the small adapter around this report workflow. Still, the absence of webhook delivery events makes it a poor fit for strict real-time multichannel orchestration or advanced analytics. It also lacks a tag-aggregated cost-report API, and geographic anti-abuse controls or country-price circuit breakers for SMS remain application responsibilities.

Domestic email compliance must be evaluated independently; a pending Tencent email vendor is not evidence for it. Security requirements are similarly local. OTP abuse controls, recovery policy, and recipient enumeration defenses belong in the application threat model, while the email side has no managed OTP interface.

## Roll out without losing the evidence

Start with one report type and email only. Backfill a `notification` row for every new report send, dual-write delivery attempts beside the existing send path, and compare provider IDs before exposing history in the UI. Then enable the poller for nonterminal attempts and alert on stale reconciliation, not merely on send errors.

Add SMS only after the normalized state machine has survived real provider vocabulary. During migration, never reinterpret old status values in place; introduce a mapping version so an audit query can reproduce the meaning used at the time. Once the database record is authoritative, move template changes behind versioned application releases and require the notification row to capture the exact version.

Keep the rollout small. The design is approachable for a normal SaaS notification center, but teams needing subsecond cross-channel decisions, complex journey orchestration, or analytics across campaign dimensions should choose a purpose-built orchestration system rather than stretching a polling loop into one.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
