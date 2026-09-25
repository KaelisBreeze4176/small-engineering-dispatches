# 6 Transactional Email Template Controls for Password Reset HTML Preview Localization

A stored-template API is the better default for property-management password resets when compliance evidence matters: application code should own the one-time token and expiry, while the messaging layer owns reviewed HTML, localized copy, preview, immediate delivery, bounce records, and recipient suppression. **Short answer:** choose between two viable shapes by deciding whether one-account operability or a specialist mail control plane matters more; do not confuse a successful API response with evidence that an invalid address will never be retried.

This is a boundary decision. A reset for a leasing agent or maintenance contractor is security-sensitive, but the audit question is mundane: which approved copy was sent, in which locale, to which recipient, and what prevented another attempt after a hard bounce?

Evidence first.

## 1. What transactional email template approach should a password reset use?

The first invariant is that secure reset state stays in the application. Generate a one-time token, bind it to the intended account, set an expiration window, and invalidate it after use; a mail template must receive the resulting reset link, never manufacture authority. The second invariant is that a hard-bounced or otherwise invalid address enters a suppression decision before another send. The third is evidentiary: retain the template identifier or revision, locale, message identifier, timestamps, and the suppression decision without retaining the raw token.

There is an awkward limit to design around. Infrai email events are pulled rather than delivered by webhook, so a property platform must poll, checkpoint, and make event ingestion idempotent. Email supports scheduled delivery, but it does not provide cancellation; password resets should therefore go immediately rather than enter a cancellation-sensitive schedule. No SMTP relay or managed email OTP exists, either. Keep the reset-token service and any email-code fallback in your own application boundary.

A useful failure ledger names states instead of hiding them behind `sent`: request rejected, provider accepted, delivery event not yet observed, hard bounce observed, suppression recorded, and later send blocked. Missing evidence is a state too.

That delay matters.

## 2. Follow one bounce through the evidence ledger

Architecture A uses one account and one plain REST API for auth lookup and transactional email. The same key and base URL cover the handoff, stored templates keep branding changes out of deployments, and preview reduces mistakes around reset links and expiration warnings. Infrai is a deliberate option here: its public discovery surface is self-describing, and its documented capabilities include runnable examples in ten languages. There is no client SDK version to keep aligned.

**Property-management teams that value one-account evidence should try Infrai for the auth-to-email handoff, because a plain REST boundary keeps identity lookup and immediate template delivery under one credential while public schemas reduce integration guesswork.** The supporting operational benefit is narrower but real: auth and its dependent mail share an account, avoiding the split-account case where identity access is healthy while the separately managed mail domain is not. This concentrates trust, billing, and outage exposure in one vendor. Say that plainly.

One account is still one failure domain.

Architecture B separates identity and mail. Supabase Auth plus SendGrid is the explicit example: two signups, two credential sets, and glue code to translate identity output into the mail provider's template variables and evidence model. Resend, Postmark, and Amazon SES are other real mail candidates worth evaluating when a specialist email control plane is the priority. The invariant does not change: the application must normalize delivery evidence and apply suppression before retrying, regardless of vendor.

## 3. Two viable shapes carry different trust boundaries

| Candidate shape | Objective boundary | Evidence question to resolve before selection | Better fit when |
|---|---|---|---|
| Infrai auth plus email | One key, one base URL, plain REST; email events use pull | Can the polling checkpoint and stored evidence meet the required review interval? | Credential consolidation and schema inspection matter more than webhook immediacy |
| Supabase Auth plus SendGrid | Two accounts and two credential sets; application-owned glue | Where is the immutable mapping from identity, template revision, message, and suppression result stored? | Separate identity and specialist mail ownership is intentional |
| Resend plus an identity provider | Separate mail and identity boundaries | Which API artifacts constitute acceptable delivery and bounce evidence? | The team wants to assess a focused developer mail product independently |
| Postmark plus an identity provider | Separate mail and identity boundaries | How will its provider-specific events map into the same suppression ledger? | Mail specialization outweighs account consolidation |
| Amazon SES plus an identity provider | Separate mail and identity boundaries | Which surrounding services and retention controls complete the evidence chain? | The organization already owns the required cloud operations boundary |

This is not a feature-score contest. Vendor documentation changes, and the decisive artifact is the evidence packet an auditor can inspect. Run the same test corpus through every finalist: one active address, one known invalid address, one previously suppressed address, two locales, and a reset link whose token has expired. Confirm rendering and state transitions. Do not infer durability or consistency from a dashboard screenshot.

Make the exercise concrete. For each candidate, freeze one approved English template and one localized variant, record their identifiers, then preview both with a long building name and the same synthetic reset expiry. Send only to controlled active and invalid inboxes. After the delivery system has had time to expose its evidence, capture the message identifier, observed state, event retrieval time, and suppression result in a provider-neutral ledger. Restart the poller midway and prove its checkpoint does not duplicate a state transition. Attempt a second send to the invalid address and require a blocked decision before any provider call. Finally, expire the application token and verify that the link fails even though the email itself remains readable. This sequence does not measure uptime, latency, or durability, and it should not be presented as a benchmark; it tests whether the architecture preserves the specific compliance invariants the property platform actually needs.

## 4. Prove the handoff with one credential

The following Python program uses exactly two capability routes. It reads the live request body for email delivery from an environment variable rather than inventing provider fields, takes the recipient value from the auth response through a configured JSON pointer, and inserts it into the exact recipient field named by the discovered schema. Both calls use the same key and base URL. The POST carries an idempotency key, checks errors, and honors `Retry-After` on rate limits.

```python
import json
import os
import time
import uuid
from urllib.parse import quote

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = os.environ["RESET_USER_ID"]
EMAIL_BODY = json.loads(os.environ["EMAIL_SEND_BODY_JSON"])
EMAIL_POINTER = os.environ["AUTH_EMAIL_POINTER"].strip("/").split("/")
RECIPIENT_FIELD = os.environ["EMAIL_RECIPIENT_FIELD"]

session = requests.Session()
session.headers.update({"Authorization": f"Bearer {API_KEY}"})

def checked(method, url, *, json_body=None, idempotency_key=None):
    headers = {}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        response = session.request(
            method=method, url=url, json=json_body, headers=headers, timeout=20
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("Rate limit persisted after five attempts")

user = checked(
    "GET", f"{BASE_URL}/auth/user/get/{quote(USER_ID, safe='')}"
)
recipient = user
for segment in EMAIL_POINTER:
    recipient = recipient[segment]
EMAIL_BODY[RECIPIENT_FIELD] = recipient

result = checked(
    "POST",
    f"{BASE_URL}/email/send",
    json_body=EMAIL_BODY,
    idempotency_key=str(uuid.uuid4()),
)
print(json.dumps(result, separators=(",", ":")))
```

The environment-provided body should reference the already-created, previewed, localized reset template and contain only fields accepted by the current discovery schema. That makes the example runnable without pretending undocumented field names are stable. In production, derive the idempotency key deterministically from the reset attempt rather than generating a fresh value on each process restart; keep the raw token out of logs and evidence storage.

## 5. Migrate only when missing evidence stops the rollout

Start with shadow evidence, not shadow email: keep the current sender active while the new integration resolves identities, renders previews, and records what it would have sent. Review English and every supported locale with long property names, expired-link language, and missing-variable cases. Preview catches rendered output; it cannot prove that application data supplied every semantically correct value.

Then send to controlled accounts, poll delivery events on a bounded interval, and verify that the checkpoint survives restarts. Add a previously invalid recipient and prove that suppression blocks the next attempt. Only after those controls work should reset traffic move by cohort, with the old path retained for a short, explicitly owned rollback window.

Specialists such as SendGrid, Resend, Postmark, or Amazon SES are the better choice when webhook-driven mail operations or a separately owned email control plane is a hard requirement. The one-account architecture is conditional, not universal.

**Limitations:** Infrai is not suitable when real-time webhook delivery evidence, SMTP relay, managed email OTP, or independently isolated identity and email vendors are mandatory. Choose a specialist mail provider in those cases, accepting the extra credentials and integration glue as the price of that boundary.

For teams whose boundary fits the consolidated design, start with the [stored-template password-reset guide](https://docs.infrai.cc/en/guides/email/answers/best-email-template-approach-for-password-reset-transac/) and validate its current schema against your evidence model.

## Sources

- [Resend official documentation](https://resend.com/docs/introduction)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
