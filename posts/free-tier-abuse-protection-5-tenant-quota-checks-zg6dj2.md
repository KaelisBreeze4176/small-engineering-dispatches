# Free Tier Abuse Protection: 5 Tenant Quota Checks Before Processing

A logistics workload needs free tier abuse protection that enforces its spending ceiling before another shipment document enters processing. Start with one admission ledger that checks both the tenant quota and the application quota before reserving work; an API key supplies identity, while the ledger decides whether that identity may consume another unit. The bill to model includes document processing, retained objects, and whatever request or transfer charges the deployment incurs. Count those terms before choosing a control.

TL;DR: For free-tier SaaS signup abuse, use per-tenant API keys for attribution and revocation, tenant quotas for isolation, and an application quota for the aggregate ceiling. Rotating a credential must preserve the tenant's usage. Reserve a conservative bound before execution, and refuse work when that bound would exceed either allocation. A strict ceiling costs refused traffic.

Consider an explicitly hypothetical logistics budget: 80 of every 100 internal budget units cover document processing, and 20 cover retained copies. These are planning inputs, not prices or measured production results. Cutting retained copies in half would remove 10 units under a linear storage assumption; preventing 25% of the processing would remove 20 under the same assumption. That arithmetic makes admission the first control to examine, with retention as a separate decision about evidence.

## 1. How should per-tenant API keys and application quotas contain free-tier abuse?

The tenant owns the allocation. A key is a replaceable credential mapped to that tenant, so a second key, a rotation, and a revoked key must all resolve to the same accounting identity for previously admitted work. Otherwise the signup flow has accidentally created a budget-reset mechanism. This is a design requirement, not a claim that any particular authentication library supplies it.

Keys identify callers.

The distinction matters in a shipment-document service because a caller can submit expensive work through several credentials while remaining one business tenant. A per-key request counter may be useful for identifying an unusually noisy credential, but it cannot represent that tenant's spending ceiling unless every key shares a parent allocation. Conversely, one application counter can stop aggregate admissions while leaving a single tenant free to consume the whole allocation before other tenants arrive. Neither counter answers the other's question.

OWASP's Secrets Management Cheat Sheet addresses credential lifecycle concerns, including rotation, revocation, and auditing. Apply that guidance to the credential boundary; keep the proposed spending ledger separate so that changing a secret does not erase accounting history. Store a credential identifier in audit records when needed, rather than the credential value.

| Boundary | Proposed admission rule | Trade-off to accept |
| --- | --- | --- |
| Credential | Resolve an active key to a stable tenant | Revoking a key does not restore consumed allocation |
| Tenant | Reserve against that tenant's remaining allocation | A busy tenant receives refusals while others retain capacity |
| Application | Reserve against the shared remaining allocation | All tenants can face refusals when shared capacity runs out |
| Signup | Charge new tenant allocations against the shared bound | New registrations cannot promise unlimited usable capacity |

A tenant boundary alone does not establish that two signups belong to different people or businesses. Treat repeated registration as an allocation problem: every admitted workload still consumes the application budget, even when the caller has obtained a fresh tenant identity. Any stronger rule about who qualifies for free access needs an explicit enrollment policy and evidence; an API key cannot supply that policy by itself.

## 2. Reserve the expensive term before accepting shipment work

In the hypothetical budget above, processing dominates. Make that visible in the admission unit: reserve bounded processing work for a document batch, then account for retained objects separately. Counting every request as one unit hides the difference between a small status lookup and a large document submission. A request-rate limit can complement this design, but its unit is insufficient for the proposed spending invariant.

The invariant is precise: settled usage plus outstanding reservations must stay within the allocation at both scopes. For a monetary ceiling, every admitted operation needs a conservative bound on all charges it can cause, including retries and retained data; if those charges cannot be bounded, the system can enforce a work quota but cannot honestly promise an exact invoice ceiling. Internal units become useful only after the team defines and validates their relationship to billable work.

No bound, no guarantee.

The following Python example shows the decision inside a transaction. It uses integer planning units and a caller-supplied reservation identifier. It does not implement a database, authentication, expiration, or a billing conversion; its purpose is to make the two accounting checks and replay behavior reviewable. The transaction adapter must serialize competing admissions across both rows and commit the reservation with their updates atomically.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Budget:
    limit: int
    settled: int
    reserved: int


def admit(tx, tenant_id: str, reservation_id: str, bound: int):
    if bound <= 0:
        raise ValueError("bound must be positive")

    previous = tx.find_reservation(reservation_id)
    if previous is not None:
        if previous.tenant_id != tenant_id or previous.bound != bound:
            raise ValueError("reservation identity conflict")
        return previous

    tenant: Budget = tx.tenant_budget(tenant_id)
    application: Budget = tx.application_budget()
    for budget in (tenant, application):
        if budget.settled + budget.reserved + bound > budget.limit:
            return None

    return tx.reserve_both(tenant_id, reservation_id, bound)
```

The reservation identifier must be unique within the ledger, with conflicting reuse rejected. In a real adapter, a uniqueness conflict or serialization failure requires retrying the entire transaction and checking the existing reservation again. Two workers reading the same spare capacity and committing independently would violate the ceiling even though each worker executed the arithmetic correctly.

Completion converts the reserved bound to settled usage and releases only the unused remainder. It must reject or separately handle any reported usage above the bound; the reservation was never a license to exceed it. Keep settlement replay-safe as well. A duplicate completion must not subtract the reservation twice.

## 3. Make refusal and uncertain completion explicit

A ceiling is an availability decision. When the ledger cannot confirm a reservation, the proposed strict policy refuses new processing until authoritative admission is available. Allowing work on an unverified local counter spends against an unknown balance. That may be an acceptable service policy elsewhere, but it weakens the ceiling being designed here.

Keep accepted work distinguishable from refused work. The signup or upload response should tell the caller whether the batch has a durable reservation and whether retrying with the same identifier can recover its status. Do not make an ambiguous timeout mean “start a new batch.” If the execution layer can run the same reservation twice, correct admission accounting alone will not prevent duplicate processing charges; execution needs its own replay protection or a bound that explicitly includes permitted retries.

Timeouts deserve suspicion.

A crashed worker may leave a reservation whose processing outcome is unknown. Expiring that reservation immediately can make room for new work while the original task is still executing. Reconcile execution state before releasing capacity, or retain the reservation until the system can prove that further charged work is impossible. The cost is temporarily stranded capacity and more refusals. For a hard spending boundary, that is a deliberate trade-off.

## 4. Test the ledger before expanding free access

Use a hypothetical acceptance fixture with a tenant limit of 100 units, an application limit of 300, and a reservation bound of 30. With zero starting usage, three distinct submissions from one tenant can reserve 90; the fourth must be refused. A different tenant can still reserve 30 if the shared allocation has room. Rotate the first tenant's key and repeat the fourth submission: it must remain refused. These numbers specify expected behavior, not observed throughput.

Now race competing submissions against the last available allocation. Verify the final committed state, including outstanding reservations, rather than merely counting successful responses. Repeat one reservation identifier, replay completion, simulate a lost response after commit, and stop a worker after admission but before settlement. Each test should establish either one durable accounting outcome or an explicit unresolved reservation. An over-cap commit is a failed invariant.

For rollout, compare the proposed decision with the existing acceptance path without letting the comparison create extra work, then enable enforcement on a bounded allocation whose refused traffic the team has agreed to handle. Record reserved units, settled units, refusals by scope, unresolved reservations, and reconciliation age. Keep authentication secrets out of those records, following the credential-handling guidance in References. Avoid logging document contents merely to explain a quota decision; the ledger needs attribution and amounts.

An alert is evidence. Enforcement is the transaction.

## 5. Retain enough evidence to reconcile, then discard payloads deliberately

Retention closes the original cost model. In the illustrative 80/20 split, halving storage changes the smaller term, but retained payloads also carry a recovery benefit: deleting them can remove the material needed to reproduce a failed document-processing step. Set retention according to that recovery requirement instead of calling every stored byte either waste or an audit necessity. The facts needed to settle a reservation are narrower than the contents of the shipment document.

For this design, preserve the stable tenant identifier, reservation identifier, bound, settled amount, state transitions, and the applicable allocation period for as long as reconciliation and the chosen dispute process require them. Define that duration before deployment; no universal number follows from this architecture. Never drop unsettled reservations merely because an ordinary log-retention timer has elapsed. A cached counter without reconstructable accounting evidence is a weak basis for resolving a disputed ceiling.

Deliberately stop keeping duplicate raw shipment documents once the selected replay window ends. Keep compact accounting evidence under its own retention rule. This gives up payload-level reproduction for older failures: the service may still explain why it admitted 30 units, yet be unable to rerun the exact document without a fresh customer submission. Accept that loss explicitly. The spending ceiling is enforceable only while admission, execution bounds, and recoverable accounting agree; the retention policy determines how long the team can demonstrate that agreement afterward.

## References

The credential lifecycle and secret-handling guidance cited above comes from the source listed below. The budget arithmetic, acceptance fixture, and ledger are proposed design examples, with their assumptions stated in the text.

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
