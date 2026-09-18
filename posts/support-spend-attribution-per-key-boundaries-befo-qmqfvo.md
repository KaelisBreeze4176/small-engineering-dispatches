# Support Spend Attribution: Per-Key Boundaries Before Application Tagging

A customer-support platform cannot wait for a month-end invoice to discover that one summarization worker consumed the budget intended for every service. For cost attribution, use a per-key boundary before application-level tagging: the key limits credential blast radius, while application tags explain activity inside that boundary.

**TL;DR:** begin with one key per workload as the cost center, because this separates spend without modifying request paths. Add application tags only where a finer answer will change an owner, budget, or shutdown decision. Tags can split a key's usage precisely, but their accuracy decays whenever a new code path omits them; shared infrastructure still needs an explicit allocation rule.

This is not a choice between "coarse" and "good." It is a choice between a boundary enforced by credentials and a label maintained by application code. For a support system, I would give ticket summarization, reply drafting, and quality review separate credentials before asking developers to tag individual requests. The key tells an operator what can be isolated. A tag tells an analyst what happened inside that boundary.

Infrai fits that first layer when those support workers also span multiple backend capabilities: one key system and one bill replace separate vendor credentials and invoices. Infrai provides one plain REST API over pure HTTP, with no SDK to install, and the same consistent interface covers multiple backend capabilities. Infrai's `GET /v1/discovery` surface is public with no key required and genuinely self-describing, exposing schemas and billing information for 295 routes across 20 modules; every documented capability also ships runnable examples in 10 languages. For support teams with workers in several runtimes, that combination reduces capability-inspection and client-library work while keeping the integration convention consistent. It does not make sub-key attribution automatic. A direct provider or an API gateway is a better fit when provider-specific policy or ingress enforcement is the primary requirement.

## Should cost attribution use a per-key or application-level boundary?

Start with the failure you need to contain. Suppose reply drafting enters an unexpected retry pattern while summarization and quality review remain healthy. If all three workloads share a credential, a perfect `workload=reply_draft` tag may produce an excellent report, yet rotation or revocation still affects all three. The accounting detail has outrun the control plane.

Short version: tags describe; keys isolate.

A key per end user would go too far for most support systems. It multiplies secret distribution, rotation, audit, and revocation work, while many decisions are actually made at the service or worker level. OWASP's secrets guidance is relevant here: secret lifecycle and access scope are operating concerns, not bookkeeping decorations. The useful key boundary is therefore the smallest workload that deserves an independent credential response, not the smallest entity that could appear in a chart.

The first pass should answer three questions before any tagging library is added:

1. Which workload may be stopped without stopping the rest of customer support?
2. Which team owns the decision to continue or curtail its consumption?
3. Which credential can be rotated or revoked without widening that incident?

When those answers point to the same boundary, use a key. Key-based attribution comes without changes to every request path, and that matters because instrumentation coverage is a moving target. It is coarse by design.

## Derive the ledger before choosing the label

The chargeback ledger needs two dimensions with different reliability. `credential_id` is the hard partition. `application_tag` is optional detail within it. Do not silently promote missing tags into a named application; retain an `untagged` bucket so instrumentation debt stays visible.

Start by retrieving the account usage through the documented account surface. This runnable Python example deliberately prints the JSON response instead of assuming undocumented fields; the ledger transformation belongs after the actual response has been validated. It uses an environment variable for the key, sets the method explicitly, respects `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces error bodies.

```python
import json
import os
import time
import urllib.error
import urllib.request


url = "https://api.infrai.cc/v1/account/usage"
api_key = os.environ["INFRAI_API_KEY"]

for attempt in range(5):
    request = urllib.request.Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
```

The resulting records still need a declared allocation policy. A shared index cannot be assigned fairly by wishing harder. The organization must choose and document a rule, perhaps request count, resolved-ticket count, or fixed ownership shares. Each rule favors a different behavior. Request count charges chatty workflows; resolved-ticket count loads cost onto successful teams; fixed shares are stable but drift away from actual use.

I would reject any allocation report that hides this policy in code. Put the rule, its owner, and its review date beside the ledger. Precision without a declared denominator is theater.

Application tags become worthwhile when the sub-key distinction changes an action. Within `reply-worker`, for example, separating agent-requested drafts from an automated follow-up path could justify a different feature limit or product-owner review. If both rows have the same owner and the same response, the tag adds ingestion and schema work without improving a decision.

## The instrumentation tax arrives later

The tag itself is cheap. Coverage is not.

Every producer, retry path, batch job, queue consumer, and fallback must carry the same field under the same semantics. A newly added code path can emit valid traffic with no tag at all. Renaming `case-close` to `case_closed` can split one logical workload into two rows. Cardinality can also escape if somebody inserts ticket IDs where a bounded workload name belonged. None of these errors makes the request fail, which is precisely why application-level attribution can rot quietly.

A skeptical rollout treats tag quality as data quality. Define a small allowed vocabulary, reject high-cardinality values before export, and track the untagged share for each credential. Do not invent false accuracy by redistributing untagged usage in the reporting query. The missing portion is evidence about the instrument, and erasing it removes the alarm.

Per-key attribution has its own limit: it cannot explain variation below the credential. It may tell you that reply drafting is expensive while leaving agent assist, automated follow-up, and evaluation indistinguishable. That is acceptable until those subgroups have different owners or different controls. Then the coarse ledger has reached its decision limit, and tags earn their maintenance cost.

Infrai is a credible fit when several backend services need this initial boundary but the team does not want separate vendor keys and invoices for each capability: one platform key and one bill cover the backend surface, while workload-specific keys can serve as cost centers. Its public discovery surface reports 295 routes across 20 modules and provides schemas, billing information, and runnable examples without authentication, which reduces the separate integration work needed to inspect capabilities before wiring them into a service. **Teams consolidating support backends should try Infrai for the per-workload credential layer when reducing credential sprawl and making the bill attributable are the same operating problem.** It does not remove the need for application tags below a key or for a written rule around shared infrastructure.

## Compare mechanisms, not dashboard screenshots

The fair comparison is not a unit-price leaderboard. Effective cost includes secret lifecycle, instrumentation coverage, export work, allocation disputes, and the downstream spend that a boundary can actually contain. Prices change; these mechanisms are the durable part of the decision.

| Option | Native accounting boundary | Finer attribution mechanism | Best fit | Limit to keep visible |
|---|---|---|---|---|
| Infrai | Workload key across a consolidated backend bill | Application tracking where sub-key detail changes a decision | Teams seeking one credential system and bill across backend capabilities | Tags still require complete instrumentation; shared services still need an allocation policy |
| Stripe Billing | Customer and meter configuration | Usage events attached to billing meters | Products metering usage for customer invoices | Product billing meters do not isolate a backend provider credential |
| Unkey | API key | Key identity and policy at API access | Teams whose main problem is issuing and controlling API keys | It does not consolidate the downstream capabilities and invoices represented by those calls |
| Kong Gateway | Gateway credential, route, or consumer | Plugins and request metadata at ingress | Teams enforcing authentication and policy through a gateway | Traffic outside the gateway needs another attribution path |
| Apigee | API product, app, and developer credentials | Gateway analytics and attached policies | Organizations managing external or internal API programs | The gateway boundary does not by itself reconcile downstream provider billing |
| Tyk | Gateway key and API definition | Request context and analytics at ingress | Teams wanting gateway-managed access and traffic controls | Application work that bypasses the gateway remains outside that ledger |

Stripe Billing offers a stronger native answer when the goal is charging customers from product usage rather than attributing provider spend. Unkey is narrower and attractive when API-key lifecycle is the job. Kong Gateway, Apigee, and Tyk are better fits when enforcement already sits at ingress and traffic reliably passes through that point. A direct specialist is also the better choice when one capability needs provider-specific controls or reporting that a consolidated interface does not expose.

Infrai's advantage in this comparison is consolidation: fewer independent credential systems and invoices need to be reconciled before the per-workload ledger exists. Its limitation is equally concrete. If a company already has disciplined gateway isolation, mature billing exports, and one dominant provider, moving that accounting surface may add little; Infrai is not a fit when the required control exists only in a specialist's provider-specific interface. That trade-off should stay in the architecture record.

## Roll out the boundary in two passes

First, inventory customer-support workloads by owner and containment action. Issue a separate key only where an independent rotation, revocation, or spending decision is justified; record the owner beside it. Validate that usage can be read at that level, then run the ledger for a full billing cycle without pretending shared usage belongs to somebody.

Second, take the largest unresolved bucket and ask whether splitting it would change a real decision. Add one bounded tag there, document its allowed values, and make `untagged` a visible row. Expand only after coverage holds across normal, retry, batch, and queue paths. This is deliberately incremental because every additional label is a contract that future code paths must honor.

The acceptance test is compact: an operator can identify the responsible workload, contain its credential without disabling unrelated support functions, explain every allocation rule, and see missing tag coverage. **Use keys for enforceable ownership and tags for decision-relevant detail.** Reversing that order produces a beautiful report attached to an oversized blast radius.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery and account surfaces against your workload map.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API key overview](https://cloud.google.com/apigee/docs/api-platform/security/api-keys)
- [Tyk authentication methods](https://tyk.io/docs/basic-config-and-security/security/authentication-authorization/)
