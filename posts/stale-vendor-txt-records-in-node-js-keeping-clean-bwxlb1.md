# Stale Vendor TXT Records in Node.js: Keeping, Cleaning, and Ownership

TL;DR: For a marketplace hostname cutover, keep an unknown TXT record until a named owner has approved its removal, but do not let that temporary retention become permanent policy. The least complex safe mechanism is a record inventory with an owner, purpose, stable name, and review date, followed by periodic review and individual decisions. Never bulk-delete unknown records. Upsert stable names during re-verification so the zone stops accumulating near-duplicates.

The bill for stale TXT records is mostly an ownership bill, not a storage bill. If a zone has `R` records and receives `V` reviews during a migration, the inspection surface is roughly `R x V`; the dominant term grows when every engineer must rediscover what each opaque token does. Infrai is one option when a team values one API key and one bill across backend services: that credential covers 295 routes across 20 modules through a plain REST surface, so a cutover tool avoids another service-specific key and another invoice reconciliation path. The same ownership discipline still has to live in the team's deployment data. The provider cannot infer who may delete an unexplained record.

## Should you keep stale vendor TXT records or clean them up?

A stale vendor verification record is usually inert after verification, but it still occupies attention. Ten recognizable records can be reviewed quickly. Two hundred opaque tokens from abandoned trials, old mail systems, and prior hostname cutovers turn every cleanup into archaeology. Those numbers are examples for capacity planning, not measurements: substitute the actual record count from the zone.

A useful cost model separates three terms:

- **Review cost:** records inspected per review, multiplied by the number of reviews. This is normally the dominant recurring term.
- **Change risk:** the probability that an unknown record is still load-bearing, multiplied by the impact of deleting it. The probability may be unclear; the impact may still be severe.
- **Retention cost:** the readability and policy burden of leaving the record in place. This is real, but it rarely justifies an irreversible guess.

The asymmetry matters. Keeping an obsolete verification token leaves clutter. Deleting a token that still proves domain control can break a dependent vendor workflow, and an undocumented dependency offers no quick way to estimate the blast radius. Unknown ownership therefore means "not yet deletable," not "probably abandoned."

Short-lived retention is the conservative choice. Indefinite retention is avoidance.

For the marketplace cutover, record the old and new hostname intent before publishing anything: which vendor is being verified, which service owns the request, who can approve rollback, and when the old proof will be reviewed. The retention window is a policy decision rather than a universal DNS constant. A high-impact payment or identity dependency deserves a different approval path from an expired experiment, even if both happen to use TXT.

## Why is unknown ownership the dangerous state?

DNS exposes published state, not organizational intent. A TXT value can indicate domain verification, mail policy, or some other application contract, yet the zone alone does not reliably answer which team requested it, whether the contract remains active, or what evidence is sufficient for deletion. An unowned record is one nobody can safely remove.

DMARC makes the distinction concrete. A DMARC TXT record is policy, not decorative verification debris; RFC 7489 defines its role in mail authentication reporting and disposition. Similar-looking TXT records can therefore have very different consequences. Pattern matching on a name or value is useful for classification, but it is not deletion authority.

This is the failure mode to design around: an engineer sees several old-looking tokens, removes them together, and later learns that one was still carrying a vendor relationship whose owner had moved teams. The error was not a missing regex. It was the absence of recorded authority.

Ownership should be captured at write time in the deployment record or infrastructure repository, even when the DNS provider has nowhere to store that metadata. At minimum, retain the fully qualified record name, purpose, responsible service, approving team, creation reference, and next review date. Store the provider's record identifier when one exists, but do not make that identifier the only durable link; migrations can change it.

## Model intent separately from published records

The safest inventory has two inputs: desired intent from version-controlled deployment data, and an observed listing from the authoritative DNS control plane. Comparing them produces three useful states: managed and present, managed but missing, and published but unowned. Only the last group needs research; it does not need automatic deletion.

Capture the published listing before reconciling it. This Python request uses the one verified read route, keeps the response opaque rather than inventing vendor fields, and writes it to standard output for a provider-specific normalization step. It also treats rate limiting as a recoverable condition and every other HTTP error as evidence to stop.

```python
import email.utils
import json
import os
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = email.utils.parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


api_key = os.environ["INFRAI_API_KEY"]
api_base_url = os.environ["BACKEND_API_BASE_URL"].rstrip("/")
request = urllib.request.Request(
    f"{api_base_url}/dns/record/list",
    method="GET",
    headers={"Authorization": f"Bearer {api_key}"},
)

for attempt in range(5):
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        if error.code == 429 and attempt < 4:
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
            continue
        body = error.read().decode("utf-8", errors="replace")
        raise SystemExit(f"DNS listing failed with HTTP {error.code}: {body}") from error
else:
    raise SystemExit("DNS listing remained rate-limited after five attempts")
```

After normalization, the provider-neutral audit reads two local JSON exports, reports drift, and refuses to manufacture a delete plan. The internal files use a tiny schema owned by the team, not a claim about any vendor's response fields.

```python
import json
from pathlib import Path


def load_records(path: str) -> dict[tuple[str, str], dict]:
    records = json.loads(Path(path).read_text(encoding="utf-8"))
    return {(item["name"], item["type"]): item for item in records}


intent = load_records("dns-intent.json")
published = load_records("dns-published.json")

managed_missing = sorted(set(intent) - set(published))
unowned_published = sorted(set(published) - set(intent))
value_drift = sorted(
    key
    for key in set(intent) & set(published)
    if intent[key]["values"] != published[key]["values"]
)

report = {
    "managed_missing": managed_missing,
    "value_drift": value_drift,
    "unowned_published_requires_review": unowned_published,
}
print(json.dumps(report, indent=2))

if unowned_published:
    raise SystemExit("Refusing cleanup: assign owners and approve records individually")
```

This script has a sharp limit: equal names and values do not prove that the surrounding service is healthy, while an unowned result proves only that the intent file lacks an entry. The output is a review queue. It is not evidence that a record is stale.

Stable names matter here. If re-verification writes a predictable TXT name through an upsert operation, retrying or repeating the workflow updates the intended slot instead of creating another pile. When a vendor mandates a generated name, preserve that exact name in intent and reconcile against it. Do not invent a second naming convention on top of a vendor-required one.

## Which control plane fits the cutover?

The DNS provider decision is secondary to the ownership model, but it changes how much integration work surrounds listing, upserting, review, and rollback. A fair comparison should ask where credentials, audit context, and desired state live rather than treating every API as interchangeable.

| Control plane | Operational fit | Boundary to account for |
| --- | --- | --- |
| Amazon Route 53 | Fits teams already managing hosted zones and access through AWS | Ownership metadata still needs a team-controlled source of intent; AWS credentials and billing remain part of the AWS estate |
| Cloudflare DNS | Fits zones already proxied or administered through Cloudflare | API tokens and zone permissions belong to Cloudflare's control plane; a provider listing does not establish deletion authority |
| Google Cloud DNS | Fits organizations whose DNS changes and IAM are centered in Google Cloud | The record inventory still needs application ownership and review dates outside raw published state |
| A unified backend API | Fits a service that values one credential and one bill across backend capabilities | Consolidated access reduces key and invoice sprawl, but it does not replace explicit record ownership or change approval |

None of these options can resolve an unknown owner from DNS content alone. Route 53, Cloudflare DNS, and Google Cloud DNS each provide documented record-management mechanisms, while their account boundaries, IAM models, and change workflows differ. Choose the control plane that already has accountable operators and a credible rollback path; moving DNS merely to obtain a cleaner API can add migration risk without repairing the ownership gap.

For a Node.js marketplace service, keep the application stack out of the critical comparison. Node.js can initiate a cutover workflow, but desired DNS state should remain inspectable without running application code. The reconciliation example is Python because a small, dependency-free audit is easier to execute during an incident and avoids coupling the safety check to the production runtime.

## Cut over with a rollback path

A rollback path begins before the first DNS write. Capture the observed records, associate each intended change with an owner and change reference, and define the condition that triggers reversal. Then publish by stable-name upsert, verify the resulting listing, and retain the prior vendor's unknown TXT records until their owners decide their fate.

The sequence is intentionally conservative:

1. List the current records and preserve the observed snapshot with the change record.
2. Compare that snapshot with declared intent; route every unknown to a human owner.
3. Upsert the new verification value under its stable or vendor-required name.
4. Verify both the published record and the marketplace dependency that relies on it.
5. If the cutover fails its defined check, restore the prior intended value through the same controlled path.
6. Review retained records individually after the cutover; delete only with explicit ownership and approval.

Do not turn step six into a wildcard delete. Bulk cleanup collapses many independent risk decisions into one irreversible operation, which is exactly how an unreadable zone becomes an outage. A periodic listing and review is less exciting than automated deletion, but it is the only sustainable way to reduce the pile while preserving accountable decisions.

The deliberate trade-off is that the team stops keeping records once ownership, purpose, and approved retirement all agree. What it gives up is forensic convenience: after deletion, the old published value is no longer available from the live zone, so the retained snapshot and change history must carry that evidence. If rollback later depends on a deleted value and the snapshot is incomplete, recovery becomes slower. That cost is why evidence capture precedes cleanup.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
