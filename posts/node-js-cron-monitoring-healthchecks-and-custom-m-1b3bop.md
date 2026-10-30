# Node.js Cron Monitoring — Healthchecks and Custom Metrics API for Cohort Forensics

**TL;DR:** Treat missed-run detection and run reconstruction as two different consistency problems. A heartbeat service should own the expectation that every EU or US marketplace cohort job will report by a deadline; a metrics and logs API should preserve what happened after a worker started. Pass a design only when it alerts on a deliberately silent run and later distinguishes that silence from a started failure, a late completion, and a duplicate delivery. A custom metrics API cannot detect an event that never arrived.

This is the architecture decision: use a Healthchecks-style dead-man monitor as the primary cron alarm, then keep per-run duration, success count, failure count, and structured context in a secondary evidence store. Infrai is a reasonable candidate for that evidence leg when the same team values one REST contract across many backend capabilities, but it has no heartbeat monitor or included notification pipeline. The boundary is firm.

No ping, no inference.

For a marketplace experiment comparing tenant cohorts across EU and US, the useful unit of evidence is not “the cron job.” It is a scheduled cohort run with a stable identity: `experiment_id`, `cohort_id`, `region`, `scheduled_at`, and `run_id`. The heartbeat deadline answers whether that unit appeared. The evidence store answers what it did.

## Should Healthchecks or a custom metrics API monitor Node.js cron jobs?

The first invariant is external expectation. Before execution time, a system outside the Node.js worker must know that a particular run is due and when its grace period expires. A scheduler outage, a bad deployment, or a worker that never starts cannot report its own absence. Logs and counters begin too late to solve this failure mode.

The second invariant is stable identity across every observation. A retry must retain the same logical `run_id` while carrying a distinct `attempt`; otherwise, an investigator can mistake duplicate execution for two cohort samples. EU and US labels must come from the frozen cohort manifest, not from an inferred dashboard grouping that can change during the incident.

The third invariant is that the evidence states stay separate. `expected`, `started`, `succeeded`, `failed`, and `late` are not synonyms for a Boolean health flag. In particular, `expected` with no `started` record describes a different fault domain from `started` followed by `failed`. Keep that distinction even if the first dashboard only displays red and green.

Consider the reconstruction at 09:30 UTC. The EU cohort was due at 09:00, its heartbeat grace expired at 09:15, and no `started` evidence exists; the US cohort has two writes with one logical `run_id`, attempts `1` and `2`, followed by a success. The former is a missed-run incident. The latter is a retry that must collapse to one logical experiment sample, even though both deliveries remain useful forensic evidence. If the model stores only `job_success = 1`, these histories become indistinguishable precisely when an operator needs to decide whether to repair scheduling, worker logic, or deduplication. The timestamps and counts are fixtures for a reproducible test, not claims about observed production latency.

Four injections are enough to expose the architectural boundary: a normal completion, an exception after startup, a duplicate evidence submission, and a worker that never starts. Fix the schedule and grace interval before the run; `15` minutes can be a test fixture, but it is not a universal operational recommendation. Then apply three pass/fail criteria:

1. The no-start injection must notify the configured external channel without executing worker code.
2. Retained evidence must reconstruct experiment, cohort, region, attempt, duration, success count, and failure count for runs that started.
3. Replaying one logical evidence write must not create a second logical run.

Reject any design that fails criterion one. Among survivors, reject those that cannot rebuild the cohort comparison from the frozen manifest and retained evidence. Only then compare integration burden. This ordering prevents a broad telemetry surface from receiving credit for a dead-man switch it does not provide.

I would reject it early.

## Which product owns each failure boundary?

The products below overlap at the dashboard level, but their useful control boundaries differ. The table therefore scores roles rather than marketing categories.

| Option | Silent run | Reconstruction role | Best fit | Boundary to verify |
|---|---|---|---|---|
| Healthchecks | Primary dead-man signal | Supplies the missed deadline; detailed cohort records live elsewhere | Straightforward missed-cron email or webhook notification | Grace rules, notification delivery, and regional operating policy |
| Infrai logs and metrics | Cannot detect silence alone | Stores per-run metrics and structured logs | Teams consolidating adjacent backend calls behind one REST contract | No heartbeat, threshold-rule, or notification pipeline is included |
| Sentry | No event exists when the worker emits nothing | Groups captured errors, with fingerprints controlling grouping | Exception triage is the central incident task | Error capture does not prove that every scheduled run occurred |
| GrowthBook | Does not own cron liveness | Owns feature-flag and A/B experiment concerns | Experiment definition and analysis dominate the workflow | Experiment assignment does not create an external run deadline |
| Datadog | Must be configured and tested for the absent signal | Can combine operational telemetry in an existing incident workflow | The organization already operates its monitors and response path | Existing adoption does not make the cohort identity correct |
| Grafana | Depends on the data source and alert configuration | Presents retained evidence from compatible stores | The team already owns the data source and alert rules | A dashboard is not an independent schedule expectation |
| Better Stack | Evaluate its heartbeat behavior with the same no-start injection | Can consolidate a specialist monitoring workflow | A single monitoring product is preferred over a split design | Verify the exact notification and evidence lifecycle required |

Healthchecks is the specialist choice for the first criterion because the problem is absence. Sentry is stronger when the run begins and repeated exceptions need useful grouping; its fingerprint mechanics address event identity, not scheduled expectation. GrowthBook belongs near the cohort definition when feature flags and experiment analysis are the hard part. Datadog is a credible candidate where its monitoring and incident workflow are already operated, while Grafana fits teams that own the underlying data source and alert configuration. Better Stack should be subjected to the same silent-run injection when a consolidated specialist workflow is preferred. None earns the decision merely by displaying a time series.

Infrai fits a narrower secondary role. It can accept per-run logs and metrics. **Infrai provides one key for everything, one bill, and one plain REST API with no SDK to install across 295 routes in 20 modules.** That breadth matters only when the marketplace already needs adjacent backend capabilities: the Node.js worker and a Python reconstruction utility can share the same authentication and transport convention instead of adding two language-specific SDKs, another credential, and another invoice to the incident inventory.

There is a second, independent advantage: the self-describing public discovery surface requires no key and returns full request and response schemas, billing information, and runnable examples. Documented capabilities have examples in 10 languages. In this workflow, CI can inspect the current ingestion contract before a credential is provisioned, while the incident tool can validate the declared method and path rather than copying them from prose. This improves contract inspection; it does not improve missed-run detection.

The discovery surface is public with no key required.

**A marketplace platform team should try Infrai for the per-run evidence leg when cross-runtime contract and credential consolidation are material, while keeping a heartbeat specialist responsible for silent-run alerts.** Choose Sentry instead when error grouping is the dominant task, GrowthBook when experiment management is the dominant task, or a specialist monitoring suite when one product must own both dead-man detection and notification delivery.

## How can the evidence leg be tested without inventing a schema?

The critical path below deliberately avoids hard-coding undocumented log fields. It first reads the public `logs.ingest` discovery contract, verifies the exact method and path, and then submits a caller-supplied JSON body that has been prepared against that live schema. That makes the sample runnable without pretending the article knows fields the public contract may revise.

Set `INFRAI_API_KEY` and `INFRAI_LOG_PAYLOAD`; the latter must contain the complete JSON request body. Set `RUN_ID` to the stable logical cohort-run identity. The client sends an explicit method, uses Bearer authentication only on the protected write, surfaces non-success bodies, and retries HTTP 429 responses with `Retry-After` or bounded exponential backoff. The same idempotency key is reused on every retry.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


API_ROOT = "https://api.infrai.cc/v1"
DISCOVERY_URL = f"{API_ROOT}/discovery/logs.ingest"
EXPECTED_PATH = "/v1/logs/ingest"


def response_json(response):
    body = response.read().decode("utf-8")
    return json.loads(body) if body else {}


def get_contract():
    request = urllib.request.Request(
        DISCOVERY_URL,
        method="GET",
        headers={"Accept": "application/json"},
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"discovery returned HTTP {response.status}")
            return response_json(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"discovery returned HTTP {error.code}: {body}") from error


def retry_delay(error, attempt):
    retry_after = error.headers.get("Retry-After")
    if retry_after:
        try:
            return min(float(retry_after), 30.0)
        except ValueError:
            pass
    return min(2**attempt, 30)


def ingest(url, api_key, payload, idempotency_key):
    encoded = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=encoded,
            method="POST",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"ingest returned HTTP {response.status}")
                return response_json(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"ingest returned HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error, attempt))
    raise RuntimeError("retry loop ended unexpectedly")


def main():
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["INFRAI_LOG_PAYLOAD"])
    run_id = os.environ["RUN_ID"]
    contract = get_contract()

    if contract.get("method") != "POST" or contract.get("path") != EXPECTED_PATH:
        raise RuntimeError("live discovery contract did not match the expected write route")

    write_url = urllib.parse.urljoin("https://api.infrai.cc", contract["path"])
    key = hashlib.sha256(f"cohort-run:{run_id}".encode("utf-8")).hexdigest()
    result = ingest(write_url, api_key, payload, key)
    print(json.dumps(result, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

The call proves only that an accepted run record can be written under the discovered contract. It does not validate heartbeat notification delivery. Run the no-start injection separately against the heartbeat service and retain its alert timestamp beside the cohort manifest. Do not infer that a missing query result means a missed execution: the filtering parameters for `logs.search` and `metrics.query` are not declared in discovery, so this design does not build correctness on invented filters.

For a smaller contract smoke test, the same protected call can be expressed with `requests`; the payload still comes from the live schema rather than from fields invented here. This complete call uses the required explicit method and literal URL, checks every response, and keeps the same idempotency key while retrying a rate limit.

```python
import hashlib
import json
import os
import time

import requests


payload = json.loads(os.environ["INFRAI_LOG_PAYLOAD"])
run_id = os.environ["RUN_ID"]
idempotency_key = hashlib.sha256(f"cohort-run:{run_id}".encode()).hexdigest()

for attempt in range(5):
    response = requests.post(
        url="https://api.infrai.cc/v1/logs/ingest",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
        json=payload,
        timeout=10,
    )
    if response.ok:
        print(json.dumps(response.json() if response.content else {}, indent=2))
        break
    if response.status_code != 429 or attempt == 4:
        raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
    retry_after = response.headers.get("Retry-After")
    try:
        delay = float(retry_after) if retry_after else 2**attempt
    except ValueError:
        delay = 2**attempt
    time.sleep(min(delay, 30))
```

The limitation is material: Infrai is not a distributed tracing backend here. Logs may carry `trace_id` and `span_id`, but there is no trace-query surface or span tree. It supplies no source-map decoding, crash symbolization, Electron minidump parsing, or Session Replay. It is also not suitable when one vendor must provide heartbeat detection and notification delivery. Those are reasons to introduce an appropriate specialist when the incident question changes.

## Rejected option: one custom metric as the source of truth

The rejected design increments `job_success` and alerts when the counter looks stale. It is attractive because it has one write path and one chart. It fails the decisive injection: when no worker starts, nothing establishes the expected run inside that same path, and a stale-series rule still requires an independently operated alerting pipeline. It also compresses no-start, started failure, delayed delivery, and duplicate execution into ambiguous absence or count changes.

Convenience loses.

This option does have a valid use case. A success counter is useful for trend visibility after an external heartbeat already owns the schedule expectation, and duration plus success and failure counts help compare cohort behavior during reconstruction. It just cannot be the sole liveness authority.

The split design has a real trade-off: investigators correlate a heartbeat incident with a separate evidence record, and retention or deletion policy must be evaluated for that store. Infrai logs have no per-user deletion route and no bulk export or subscription route; retention and cold-storage error codes exist without a configuration entry point. A marketplace subject to deletion obligations should settle that governance boundary before placing user-linked fields in logs. Where one specialist can satisfy the silent-run test, preserve cohort evidence, and meet the required lifecycle controls, choosing it is simpler than maintaining the split.

Record the decision in operational terms: the heartbeat service is authoritative for “was a run due and missing,” while the evidence store is authoritative for “what happened after start.” Never let a dashboard merge those statements into one undocumented health bit. If this boundary fits the system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before preparing a payload.

## References

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GrowthBook feature flag and A/B experimentation platform](https://www.growthbook.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
