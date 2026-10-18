# Password Reset Email Deliverability Setup: Implementing 3 Custom-Domain Controls in 2026

A marketplace password reset email deliverability setup should use an authenticated custom domain, keep bounces and complaints in a durable suppression set, and retain three linked proof layers: the request decision, the accepted message, and the delivery event. That is the least complex setup that lets an operator answer both urgent questions: did the reset leave our boundary, and should this address receive another attempt?

**Short answer:** publish SPF for the actual envelope-sender domain, sign with DKIM using the visible organizational domain or an aligned subdomain, publish DMARC with reporting before tightening enforcement, warm a new sending stream with real transactional traffic, and check suppression before every send. Keep receipt and recovery traffic logically separate even if they share infrastructure; a marketplace order receipt proves a settled payment, while a password reset authorizes account recovery, so their retention and retry decisions are not interchangeable.

The storage bill is mostly dictated by event volume times retention, not by the tiny reset body. If 10 million messages per month create six raw event records averaging 2 KB, one year of uncompressed evidence is about 1.44 TB before indexes and replicas: `10,000,000 x 6 x 2 KB x 12`. The useful move is to retain compact, immutable decision and disposition records while expiring redundant webhook bodies and rendered MIME. Keep hashes and stable identifiers. Stop keeping duplicate payloads after the short investigation window; the cost is that a later dispute can prove what was decided and observed, but cannot reconstruct every transient header. This approach also has a hard limitation: compact normalized evidence is unsuitable when a regulator, contract, or active investigation requires the original MIME and headers, so those cases need a separately governed legal-hold path whose access, encryption, deletion suspension, and release are recorded. The trade-off is deliberate, but it is not free.

Evidence first.

## How should a custom-domain password reset email setup prove deliverability?

Start with a data contract, because an inbox screenshot is weak evidence and a provider dashboard is not your system of record. Each attempt needs a random message identifier, purpose, recipient pseudonym, authentication configuration version, policy decision, transport disposition, and timestamps. Never store the reset token in the evidence record. Store a keyed recipient digest so investigators can correlate events without turning the audit table into another address directory.

| Proof layer | Minimum retained fields | Failure it distinguishes | Retention class |
|---|---|---|---|
| Decision | request ID, purpose, policy version, recipient digest, suppression result | application skipped versus attempted | Long, immutable audit record |
| Acceptance | message ID, envelope domain, DKIM selector, remote response class, accepted time | local handoff failure versus remote acceptance | Long, compact transport record |
| Outcome | message ID, normalized event type, event time, source event hash | delivered, delayed, bounced, or complained | Long for facts; short for raw payload |

“Accepted” needs careful language. SMTP acceptance records a handoff, not placement in the inbox and not human reading. That distinction prevents an audit report from claiming more than the evidence supports. It also keeps retry logic honest: a timeout before a conclusive response is ambiguous, while a permanent failure should update suppression policy rather than trigger blind retries.

For marketplace receipts, retain the settled-payment reference and template version beside the same message ID, but keep payment data out of mail events. For recovery mail, retain the recovery-request ID and expiry policy version, never the credential or token. One evidence schema can serve both jobs without collapsing their security boundaries.

Boundaries matter.

## Build the authenticated sending boundary

Use a dedicated transactional subdomain and make ownership explicit. SPF authorizes hosts for the envelope sender; DKIM adds a cryptographic signature whose domain is carried in the signature; DMARC evaluates alignment between the visible From domain and an authenticated SPF or DKIM domain. These controls complement one another. A passing SPF result for an unrelated domain does not establish the visible identity readers see.

Publish one SPF policy at the envelope domain, rather than accumulating separate SPF records as integrations change. Rotate DKIM selectors by publishing the new public key, beginning signatures with it, observing verification, and only then retiring the old selector. RFC 6376 permits multiple signatures, which can make a controlled overlap possible. Protect private signing keys as production credentials and record the selector and configuration version used for each message.

Deploy DMARC as an observation loop first. Aggregate reports reveal which legitimate streams align and which forgotten senders do not; enforcement should tighten only after those streams are accounted for. A strict policy deployed before inventory can reject legitimate receipts and reset mail. A reporting-only policy left forever, however, supplies visibility without enforcement. The decision gate should therefore require a measured observation window, named owners for every source, and an explicit promotion review rather than a calendar guess.

DNS is observed through caches. Verify records from more than one resolver, wait for the published TTL and operational propagation, then test the exact From and envelope domains used in production. Do not infer readiness from a control panel's green check alone.

Trust the wire.

## Implement suppression before transport

The send path should commit its decision before calling the transport. This Python example uses only the standard library, persists suppression and audit facts in SQLite, and deliberately leaves the transport behind a callable interface. The caller supplies a tokenized URL; the audit layer receives only a request ID.

```python
import hashlib
import hmac
import json
import sqlite3
import uuid
from datetime import datetime, timezone
from typing import Callable

SCHEMA = """
CREATE TABLE IF NOT EXISTS suppression (
    recipient_digest TEXT PRIMARY KEY,
    reason TEXT NOT NULL,
    recorded_at TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS mail_audit (
    message_id TEXT PRIMARY KEY,
    request_id TEXT NOT NULL,
    purpose TEXT NOT NULL,
    recipient_digest TEXT NOT NULL,
    policy_version TEXT NOT NULL,
    decision TEXT NOT NULL,
    transport_ref TEXT,
    created_at TEXT NOT NULL
);
"""

def utc_now() -> str:
    return datetime.now(timezone.utc).isoformat()

def digest_recipient(secret: bytes, address: str) -> str:
    normalized = address.strip().lower().encode("utf-8")
    return hmac.new(secret, normalized, hashlib.sha256).hexdigest()

def queue_reset(
    db: sqlite3.Connection,
    audit_secret: bytes,
    request_id: str,
    recipient: str,
    reset_url: str,
    send: Callable[[dict], str],
) -> str:
    recipient_digest = digest_recipient(audit_secret, recipient)
    message_id = str(uuid.uuid4())
    blocked = db.execute(
        "SELECT reason FROM suppression WHERE recipient_digest = ?",
        (recipient_digest,),
    ).fetchone()
    decision = "suppressed" if blocked else "approved"

    with db:
        db.execute(
            "INSERT INTO mail_audit VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
            (message_id, request_id, "password_reset", recipient_digest,
             "mail-policy-2026-01", decision, None, utc_now()),
        )

    if blocked:
        return message_id

    transport_ref = send({
        "message_id": message_id,
        "to": recipient,
        "subject": "Reset your marketplace password",
        "body": f"Use this one-time link: {reset_url}",
    })
    with db:
        db.execute(
            "UPDATE mail_audit SET transport_ref = ? WHERE message_id = ?",
            (transport_ref, message_id),
        )
    return message_id

if __name__ == "__main__":
    connection = sqlite3.connect("mail-evidence.db")
    connection.executescript(SCHEMA)
    print(json.dumps({"schema_ready": True}))
```

The short transaction before transport creates a durable decision, but it also exposes a named failure mode: the process can stop after remote acceptance and before `transport_ref` is updated. Production code should reconcile by idempotency key or by importing transport events keyed to `message_id`; it should not generate a fresh attempt merely because the local row lacks a response. This is the classic dual-write gap. Name it.

Suppression reasons should distinguish permanent mailbox failures, complaints, policy blocks, and temporary deferrals. Temporary SMTP failures belong in a bounded retry schedule, not a permanent suppression set. Removal from suppression must be an auditable policy action; otherwise a bulk import or support tool can quietly reactivate an address that produced a complaint.

## Warm the stream without manufacturing traffic

Warming means increasing legitimate volume gradually while watching deferrals, permanent failures, complaints, and authentication results. It does not mean sending synthetic messages to inactive accounts. A marketplace with uneven payment volume should cap rate changes and preserve headroom for password resets, because a receipt surge after a promotion must not consume every retry slot needed for account recovery.

Use cohorts that already expect the mail: settled-order receipts first, then the normal reset flow as requests occur. Keep marketing traffic elsewhere. If volume jumps, pause the increase and inspect the recipient-domain distribution and response classes; an aggregate success rate can conceal a concentrated failure at one large mailbox domain. No universal daily percentage is defensible from the cited standards, so set thresholds from your own baseline and document the evidence that approved each increase.

Three signals are insufficient alone. Delivery events can be delayed, complaint feedback is incomplete across mailbox systems, and authentication success says nothing about recipient expectation. Read them together, segmented by purpose and destination domain.

No single metric wins.

## Test failure paths and retention deliberately

Before deployment, test a suppressed address, a temporary transport error, a permanent failure, duplicate event delivery, an event arriving before the send response, DKIM selector rotation, and an unavailable DNS resolver. Confirm that logs exclude reset URLs and raw tokens. Confirm, too, that a repeated request does not reveal whether an account exists; the public response should remain uniform while internal evidence records the real decision.

Operationally, alert on changes, not vanity totals: authentication pass-rate regression, a rising permanent-failure ratio, an expanding retry queue, event-ingestion lag, and unexplained suppression removals. Break these down by message purpose. Receipt traffic is often larger and can mask a broken recovery path in a combined chart.

The retention policy deserves a destructive test. Export a sample case using only the compact records, then ask an investigator to reconstruct the decision, handoff, and observed outcome. If that works, expire raw webhook payloads, duplicate transport responses, and rendered message bodies on the shorter schedule. You deliberately give up late forensic access to incidental headers and exact body rendering. In return, storage growth follows compact evidence rather than every verbose event copy, and sensitive recovery content has less time to leak.

The final gate is plain: do not launch because DNS records exist. Launch when identity alignment is verified, suppression precedes transport, ambiguous handoffs can be reconciled, expected traffic can grow under measured limits, and the retained evidence answers what the system decided without retaining the secret it delivered.

## Further reading

- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 5321, Simple Mail Transfer Protocol: https://datatracker.ietf.org/doc/html/rfc5321
- Transactional email best practices: https://postmarkapp.com/guides/transactional-email-best-practices
