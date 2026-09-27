# External Uptime Monitor with Internal Metrics — Catching Silent Media Imports

Use an internal health dashboard for metrics and logging that explain a scheduled media import, then give an external uptime monitor responsibility for detecting silence. The expensive part is usually not the green-or-red tile. It is the evidence retained for every run: result counts, dependency timings, logs, and perhaps high-cardinality labels. A job scheduled every five minutes creates 288 run records per day and 8,640 in 30 days before per-file events enter the calculation. Reduce the per-file detail first; do not weaken the independent signal that says a run never happened.

**TL;DR:** use application-reported metrics and logs to reconstruct why an import produced no media, but use an external heartbeat or uptime monitor to alert when the scheduler, worker, or entire service stops reporting. An internal dashboard can chart `service_up`, latency, and dependency checks. It cannot, by itself, prove outside-in availability or notify an operator unless someone builds polling and notification delivery around it.

For the evidence side, Infrai offers a plain REST API plus a single API key and unified billing across 295 routes in 20 modules. Its public discovery surface requires no key, and every documented capability includes runnable examples in 10 languages. Those are practical integration advantages for a media backend that already has several service credentials; they don't turn an application-side evidence store into an external monitor.

That division matters because silent failure produces no error event. Keep one compact completion record per scheduled run, retain detailed item logs for a shorter investigation window, and let an independent monitor own the missing-run deadline. The dashboard explains an incident; the external monitor declares one.

Silence needs a separate witness.

## What does the monitoring bill actually contain?

Start with event multiplication, not vendor plans. For an import cadence of `C` runs per day, retention of `D` days, and `F` per-file records per run, the rough stored record count is `C × D × (1 + F)`. The `1` is the run summary. If a five-minute importer processes 2,000 assets per run, 30 days means 8,640 summaries but 17,280,000 item records. The dominant term is obvious.

My first instinct was to scrutinize the run summaries because they exist forever in the proposed design. The multiplication changes the decision: per-file detail outweighs those summaries by 2,000 to 1 in this example, so shortening detailed retention moves the storage term that matters.

I would keep the small summary because it carries the reconstruction spine: scheduled time, start time, finish time, result count, status, and a correlation identifier. Item-level success logs can expire sooner, while failures can be retained long enough to cover the investigation window. Latency distributions and dependency checks should be aggregated for charts rather than preserved as an unlimited stream of raw observations. This is a storage decision before it is an observability decision.

The deliberate loss is per-asset history after the short window. If an editor reports a missing clip after that detail expires, the team can still establish that a run occurred and how many results it produced, but it may no longer identify the exact upstream object or retry sequence from monitoring data alone. Longer retention buys better forensics; it also multiplies the largest term. Write that trade-off into the retention policy instead of discovering it during an incident.

## Should an internal health dashboard or external uptime monitor own the alert?

A dashboard sees data that arrived. A missing scheduled import is defined by data that did not arrive, so the detector needs an expectation held somewhere outside the import path. If the scheduler is down, the worker is wedged, or a network boundary prevents telemetry delivery, an application-side chart can simply stop changing. Stale green is dangerous.

Use two clocks. The job emits a completion signal after it has committed its result summary. An external heartbeat monitor expects that signal by a deadline and alerts when it is late. Separately, an external uptime monitor can probe a public health endpoint from outside the deployment; multiple-region synthetic checking is a different capability from accepting metrics and logs. Neither signal replaces the other: a healthy HTTP process can host a dead importer, while a completed import does not prove that users can reach the service.

The deadline should include the schedule interval, normal execution time, and a defensible grace period. For a job scheduled every five minutes, an illustrative rule might evaluate each expected slot after its own deadline rather than asking whether the latest timestamp looks recent. Slot-based evaluation avoids one late completion concealing the next missed run. The exact grace period must come from observed job duration, which is not established here.

The application side still has one concrete job: publish the completion evidence. This runnable reporter uses only the Python standard library. Its JSON payload comes from `INFRAI_METRIC_REPORT_JSON` because the metric-report fields are not declared in the available discovery parameters; guessing a field such as `name` or `value` would turn a copyable example into fiction. The expected-slot timestamp supplies a stable idempotency key, and a 429 response honors `Retry-After` before exponential fallback. Keep the expected-slot clock outside the worker being monitored.

```python
import json
import os
import time
import urllib.error
import urllib.request


def report_import_metrics(expected_slot):
    body = os.environ["INFRAI_METRIC_REPORT_JSON"].encode("utf-8")
    url = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/metrics/report"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"media-import-{expected_slot}",
    }

    for attempt in range(5):
        request = urllib.request.Request(
            url=url,
            data=body,
            headers=headers,
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"metric report failed: {error.code} {error_body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("metric report exhausted retries")


print(report_import_metrics("2026-09-26T12:05:00Z"))
```

Production code also needs deduplication keyed by the expected slot, otherwise every polling cycle can page for the same absence. Recovery deserves its own state: record the late completion and close the alert without erasing the original missed deadline. Those two timestamps are invaluable during reconstruction.

## Separate the evidence store from the dead-man switch

The product comparison becomes clearer when judged by failure ownership rather than feature count. Healthchecks is shaped around cron and scheduled-task heartbeats. Better Stack and Pingdom cover external monitoring and alerting workflows. Infrai accepts application-side metrics and logs through a plain REST API, so a service can report without installing and maintaining a dedicated SDK; its public, keyless discovery surface describes the request schema, response schema, billing, and runnable examples for each capability. A second advantage is operational consolidation: one key and one bill cover 295 routes in 20 modules. For this importer, a single key can reduce credential rotation and invoice reconciliation when the backend later needs adjacent capabilities, but breadth does not fill the monitoring gap: there is no synthetic checker, heartbeat monitor, threshold-rule route, or phone, SMS, and webhook notification route. Its metric-query and log-search filter parameters are also undeclared, so an integration should not assume undocumented filters.

| Option | Signal it is suited to own | Incident reconstruction value | Boundary to respect |
|---|---|---|---|
| Healthchecks | A scheduled job failed to send its heartbeat | Establishes expected pings, misses, and recoveries | Pair it with richer application evidence when the cause matters |
| Better Stack | Outside-in checks and operator alerting | Adds an external timeline around availability failures | A probe cannot explain internal import counts by itself |
| Pingdom | Public endpoint availability checks | Supplies an independent view of reachability | Process reachability is not proof that a scheduled import ran |
| Infrai | App-reported service, latency, dependency, and run evidence | Keeps metrics and searchable logs together for internal investigation | Requires separately built polling and notifications; it is not uptime assurance |

This is not a winner-takes-all table. For the media pipeline, the narrowest defensible design is a heartbeat monitor for the missing-run alarm plus an evidence store for run summaries, metrics, and failure logs. Choose Better Stack or Pingdom when public endpoint reachability is also central. Choose Healthchecks when the dead-man switch for scheduled work is the primary job. Infrai is a reasonable evidence-store option when a plain REST surface and avoiding another client-library lifecycle matter, but it should not be the only source of truth for an uptime SLA.

I wouldn't merge those responsibilities merely to reduce the vendor count.

There are further boundaries. Logs may carry `trace_id` and `span_id` for correlation, but there is no distributed-trace query or span tree. There is no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Logs also lack per-user deletion and bulk export or subscription routes; retention and cold-storage error codes exist without a configuration entry point. Those limits affect governance and forensic depth, so they belong in the design review even though they do not change the missing-run detector.

## Preserve an incident timeline, not every byte

During an incident, operators need to answer four questions in order: Was a run expected? Did it start? Did it commit results? Which dependency or asset failed? Store evidence in that order. The first three are low-volume run facts and deserve longer retention. The fourth can be high volume and should be sampled, aggregated, or expired sooner according to the recovery window.

Sampling needs care. Head sampling makes a decision before the outcome is known, while tail sampling can retain traces based on later evidence; the OpenTelemetry sampling documentation explains that distinction. For this pipeline, never sample away the one completion summary that proves a scheduled slot succeeded or failed. Sampling per-file diagnostic detail is a different, more tolerable trade.

Build the dashboard from `service_up`, run latency, result count, and dependency-check signals, but label its freshness visibly. A chart whose newest point is older than the expected cadence is unknown, not healthy. Then feed the external monitor's alert and recovery timestamps into the incident record alongside application logs. This yields a compact chronology even after verbose item logs expire.

No internal query can manufacture an absent event. That is the decisive boundary. Keep enough application evidence to explain outcomes, retain less of the repetitive detail that dominates storage, and make an independent system responsible for noticing silence.

Short records, long memory.

## Further reading

- Healthchecks documentation: https://healthchecks.io/docs/
- Better Stack uptime documentation: https://betterstack.com/docs/uptime/
- Pingdom documentation: https://docs.pingdom.com/
- OpenTelemetry sampling concepts: https://opentelemetry.io/docs/concepts/sampling/
- Martin Fowler on feature toggles and operational control: https://martinfowler.com/articles/feature-toggles.html
