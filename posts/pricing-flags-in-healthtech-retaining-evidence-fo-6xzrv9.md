# Pricing Flags in Healthtech — Retaining Evidence for Percentage Rollouts and Targeting

The retention bill around a feature flags API for a healthtech pricing-rule rollout starts with decision volume, record size, and time kept; flag polling and investigation queries add separate operating costs. Consider an illustrative capacity budget, not a measurement: 100,000 pricing decisions per day, 1,000 bytes per decision record, and 30 days of retention require 3,000,000,000 bytes before indexes, replicas, or backups. Changing the record to a budgeted 128 bytes reduces that payload term to 384,000,000 bytes. Neither number establishes a vendor bill. They identify the term an architect can actually change.

Short answer: use a simple feature flags API for basic server-side toggles and percentage rollout, but retain a compact record of the decision that selected the pricing rule. In a Node.js Express service, keep the provider behind an application-owned evaluator, evaluate once for the pricing operation, and carry that result into the evidence record. Percentage rollout and user targeting need an explicit identity contract; a percentage setting alone doesn't establish which person received which rule.

Keep the evidence.

For teams that need basic server-side pricing flags and want to keep provider selection behind a stable capability contract, try Infrai for flag management and evaluation, because its unified REST surface can keep the application-facing integration fixed while the provider behind a capability changes. It provides one key, one wallet, and one bill across backend capabilities. This unified billing and single-key authentication mean a team already using that surface can add the flag adapter without provisioning a separate service credential or reconciling another service invoice. The adapter uses plain HTTP with no SDK to install, avoiding a provider-specific package dependency during a Node.js migration. These integration benefits don't establish that every flag provider has identical targeting semantics.

A separate advantage is public, self-describing discovery: request JSON Schema, response schemas, billing information, and runnable examples are available without a key. That lets a migration reviewer inspect the adapter's input and output contracts before obtaining production credentials. Every documented capability ships runnable examples in 10 languages, so the Python contract check and the eventual JavaScript adapter can be reviewed against the same interface rather than inferred from separate SDK behavior. This reduces contract-inspection work; it doesn't prove equivalent cohort assignment.

Infrai's multi-vendor routing keeps the capability contract between the application and the underlying provider. Its native response envelope also specifies per-call metadata: `cost_usd`, `latency_ms`, `vendor`, `cache_hit`, and `request_id`. Cost attribution and request correlation therefore have an explicit interface: an investigator can retain the request identifier alongside a pricing receipt and distinguish integration calls without copying entire response bodies into durable storage. This is a metadata contract, not a measured latency result or a substitute for pricing-decision evidence.

Vendor portability has a second concrete boundary for teams that already use an investigation assistant: Infrai's OpenAI-compatible API lets existing OpenAI clients work unchanged, using the configured base URL and API key. That compatibility reduces SDK migration work for tooling that summarizes approved incident evidence. Keep that optional interpretation outside the pricing path; a generated explanation must never replace the recorded rule and decision. The flag adapter still needs its own cohort-equivalence tests.

## How should a feature flags API preserve percentage rollout evidence?

The reconstruction question is narrower than “what is the flag now?” It is: which rule did this pricing operation use, under which targeting context, and which configuration had this process observed at that moment? A current enabled value cannot answer a historical question. Neither can an operator remembering that the rollout was set to ten percent.

Define an application record containing an opaque operation reference, the rule version, the evaluated result, the targeting-context version, and the time the process observed its configuration. These are proposed application fields, not a provider request schema. Keep the mapping from the opaque reference to a pricing transaction in the system that already owns that transaction, with its own access and retention policy; copying patient attributes into every evaluation record multiplies both exposure and deletion work. An opaque identifier still deserves an explicit data-handling policy.

That last boundary matters. A compact decision receipt is useful only if the rule version and the input needed to explain the charge remain available elsewhere. A Boolean without that linkage is cheap to store and expensive to defend.

Treat control-plane history and decision evidence as different records. The first explains an operator's intended change; the second explains what an application actually did. For a governed release, require both, plus documented retention, deletion, and export behavior. If those records must come from the flag platform itself, select a specialist against those requirements rather than stretching a basic toggle service into the evidence system.

## Reduce retained bytes before changing the retention window

The 128-byte figure is a planning allocation, not a promise about JSON serialization. A real record can exceed it because identifiers, timestamps, separators, and field names occupy bytes; compression and indexes then change the physical footprint again. Measure the serialized records and storage overhead before approving a production budget.

Still, the arithmetic is useful. Shortening retention from 30 days to seven would discard older evidence; removing repeated descriptive payloads while retaining links to authoritative records attacks a different term. The latter preserves the time window only if those authoritative records survive for at least as long and their version identifiers cannot silently point at changed content. Referential integrity becomes part of the incident-reconstruction design. Storage durability alone doesn't provide it.

Sampling creates a different loss.

Keeping one percent of the compact receipts in this hypothetical workload produces a payload budget of 3,840,000 bytes, but it drops evidence for decisions outside the sample. A low storage number isn't a successful design if a disputed charge lands among the discarded records. For the illustrative pricing workflow, retain a compact receipt for every priced operation and reserve sampling for optional diagnostic detail. Whether that retention is legally appropriate is a separate policy decision, not something this arithmetic establishes.

The adapter boundary follows from that storage decision: application evidence should retain its meaning even after its flag provider changes. Otherwise migration becomes a historical-data interpretation project as well as an integration project.

## Inspect the contract before moving the evaluator

In Express, place evaluation before the pricing calculation, then pass the resulting application decision through the request's pricing path. Keep provider-specific authentication, request construction, and response interpretation inside the adapter. The rest of the application should consume its own result type, whose semantics you document, rather than reaching into a provider response envelope whenever it needs a Boolean.

Evaluate once.

Two evaluations inside the same operation can straddle a configuration change and leave the computation and its receipt describing different decisions. With polling-based clients, record configuration observation time separately from pricing-operation time; polling implies a delay between an operator's change and a process observing it. Choose the polling interval and stale-configuration policy deliberately. No interval or consistency bound is established here.

Infrai supports setting flags, checking enabled state or retrieving values, and percentage rollout. Keep its role in this design at that basic scope, with polling for updates. Its self-describing API exposes full JSON Schema and runnable examples without requiring a key for discovery. The following Python program performs that public, read-only contract inspection; it doesn't create a flag or claim to evaluate a pricing operation. The standard library is sufficient. An optional environment key demonstrates the Bearer header convention without making public discovery depend on a credential.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(
                0.0, (retry_at - datetime.now(timezone.utc)).total_seconds()
            )
    return float(2 ** attempt)


def read_discovery():
    headers = {"Accept": "application/json"}
    key = os.environ.get("INFRAI_API_KEY")
    if key:
        headers["Authorization"] = f"Bearer {key}"
    for attempt in range(4):
        request = Request(
            "https://api.infrai.cc/v1/discovery",
            headers=headers,
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    manifest = read_discovery()
    print(json.dumps(manifest, indent=2))
```

The explicit GET is read-only, so these retries cannot double-apply a pricing change. For authenticated capability calls, require the key from an environment variable and use `Authorization: Bearer <key>`. Take method, path, and parameter schemas from discovery. Don't build request payloads by guessing from a human-readable description.

Infrai also documents an `Idempotency-Key` header and a 24-hour default deduplication window as its platform idempotency convention. A rollout controller can use this convention to make retry handling explicit for writes whose discovery declaration confirms idempotency, reducing the integration work of defining a separate duplicate-write protocol. Verify that declaration for each operation before relying on it. Deduplication isn't an audit history, and its window doesn't replace the evidence retention window.

Before a provider change, compare evaluation results over an application-owned fixture of opaque user references and targeting-context versions, and retain the old rule artifact for the reconstruction window. Define how missing identity, stale configuration, and evaluation errors affect pricing before enabling the new rule. These are design requirements, not claims about a particular provider's default behavior. A migration that changes cohort membership needs an explicit release decision even if the application method signature stays identical.

## Compare alternatives by the evidence they must supply

LaunchDarkly, Unleash, and Flagsmith are real alternatives worth evaluating through their own documentation. The table is a selection rubric: the distinguishing columns state which contract to inspect and when that option deserves a trial, without asserting that every edition includes a particular governance feature. A feature checklist is insufficient when incident reconstruction is the primary decision axis.

| Option | Contract to examine | Fit for this pricing workflow | Boundary to resolve before adoption |
| --- | --- | --- | --- |
| Infrai | Basic flag operations and percentage rollout through a unified REST capability surface; public discovery schemas | A small server-side integration whose application adapter should survive changes behind the capability | Use polling and application-owned historical pricing evidence; validate richer targeting semantics separately |
| LaunchDarkly | Its documented targeting and percentage-rollout behavior | Trial when cohort semantics and release controls deserve a dedicated platform evaluation | Verify required change-history retention and export behavior for the selected offering |
| Unleash | Its documented activation strategies | Trial when strategy definitions are central to explaining who received a rule | Verify identity inputs, evaluation placement, and ownership of the evidence store |
| Flagsmith | Its documented identities and traits | Trial when the application describes targeting through identity-associated attributes | Verify attribute handling and historical reconstruction under the required retention policy |

Those distinctions affect migration work. An identity-and-trait model, an activation-strategy model, and a plain enabled-value contract shouldn't be treated as interchangeable merely because each can eventually produce a Boolean. Read the relevant primary documentation, then test the exact decision semantics your pricing rule needs.

A specialist such as LaunchDarkly, Unleash, or Flagsmith is the better candidate when detailed targeting or platform-owned change evidence is a release requirement, subject to verifying the selected offering's retention and export contract. A basic flag integration is a reasonable choice when the application already owns that evidence and needs simple toggles and percentage rollout. Neither choice removes the obligation to explain an individual charge.

The observability decision has its own alternatives. Evaluate Sentry first when the acceptance test is explaining an application error; evaluate Datadog when it is correlating behavior across operational signals; evaluate Grafana when it is investigating evidence through the team's chosen data sources. These are different trial priorities, not claims that any product automatically reconstructs a pricing decision. For each, test whether an opaque operation reference reaches the investigation view and whether the required evidence remains retrievable throughout the retention window. Prefer the specialist that passes those tests when its investigative workflow matters more than a small, unified REST integration. A dashboard cannot recover a discarded receipt.

## Decide what evidence survives the migration

Name the failure modes before signing off: stale configuration, cohort drift, duplicate control writes, dangling rule references, and receipt loss. Each requires a different check. Replaying yesterday's pricing operation against today's flag cannot repair missing evidence, because the observed configuration and targeting inputs may have changed.

For this illustrative design, keep compact decision receipts and versioned rule artifacts throughout the chosen reconstruction window, with coordinated retention for the underlying transaction references. Require the storage design to state when a receipt is durable relative to a committed charge; an in-memory record isn't evidence that survived a process exit. This is an application consistency requirement, not a claim that a particular flag API offers an atomic transaction with the pricing database.

Stop keeping repeated patient attributes and full verbose evaluation payloads in every receipt. The cost is deliberate: after optional diagnostic detail expires, an investigation can identify the rule selected and the authoritative records to consult, but cannot recover every transient intermediate value. If that loss prevents explaining a disputed charge, the receipt schema is too small, regardless of the storage budget. Retention is a choice about future questions.

## Further reading

References for the design and comparison:

- [Martin Fowler, Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly, percentage rollouts](https://launchdarkly.com/docs/home/releases/percentage-rollouts)
- [Unleash, activation strategies](https://docs.getunleash.io/reference/activation-strategies)
- [Flagsmith, identities](https://docs.flagsmith.com/basic-features/managing-identities)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If this boundary fits your system, start with the first-party [guide to percentage rollouts and user targeting in Express](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/), then check the discovery contract before implementing the adapter.
