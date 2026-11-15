# Marketplace SMS Verification API and Custom OTP Endpoint Evidence Boundaries Explained

A marketplace startup choosing an SMS verification API for US and European login has a harder constraint than the send operation: it needs to explain why a recipient was contacted, why another attempt was blocked, and which evidence justified either decision. Short answer: compare a hosted OTP endpoint with a custom SMS send-code flow by the verification state and suppression evidence each leaves under your control. Choose a boundary you can audit, then evaluate its operational burden. A low advertised message price cannot answer that question.

This note proposes an evidence-oriented design for a marketplace that must handle email bounces and suppress invalid recipients alongside phone-based login. Email bounce handling and SMS verification belong to different channels, but their records meet at the same account. Keep the reason for suppression attached to the destination and channel; otherwise, an email delivery failure can become an unexplained phone-login denial. That's an architectural mistake worth catching before a provider comparison begins.

## How should a startup compare an SMS verification API with a custom OTP endpoint?

Start with the narrowest decision the available evidence supports. An explicit invalid-destination result can justify suppressing that destination under a documented policy. A timeout leaves the outcome unresolved. A rejected code submission concerns a verification attempt. These should be separate states in the proposed system, because collapsing them into a single failed flag destroys the explanation a support engineer or reviewer needs later.

Preserve the distinction.

Unknown isn't invalid.

Consider an illustrative marketplace account, buyer-417, with an email destination and a phone destination, both represented in the application by opaque references. The email channel records an invalid-recipient event; the phone channel records a pending login challenge. The desired result is an email suppression decision with its own evidence reference, while the pending phone challenge follows its own verification policy. Neither event should silently rewrite the other channel. These are example records, not observations from a production incident. The particularly awkward failure mode is late evidence. Suppose an event describing an old email address arrives after the buyer replaces that address. If the handler resolves the event through the account's current destination, it can suppress the replacement. Bind evidence to an immutable destination version instead, and make the policy evaluator check that binding before it changes current eligibility. An account identifier alone is insufficient for this proposed model.

For compliance evidence, the useful question is therefore narrower than whether a message was sent: which destination version, event, and policy revision supported the decision? This is an engineering evidence requirement, not a claim that one schema establishes legal compliance across the US and Europe.

## Derive the storage boundary before choosing the endpoint

Treat suppression as a decision with provenance. In this design, every decision records a channel, destination reference, destination version, normalized reason, source-event reference, policy revision, and decision time. Store only the evidence needed for the stated purpose, define its retention policy, and make deletion behavior part of the design review; don't assume that retaining every raw payload indefinitely makes an audit stronger.

A hosted OTP endpoint can be evaluated as a candidate boundary where the service manages challenge creation and checking. A custom flow can be evaluated as a boundary where the application manages those responsibilities and delegates message transport. Those are comparison models, not promises about every API. For a particular candidate, require documentation or a test that demonstrates which operations it owns, what results it exposes, and how long the relevant evidence remains available.

The application should still own the marketplace's decision about whether a destination is eligible for contact. Otherwise, migrating delivery services also becomes a migration of suppression policy, with no clear way to explain which historical decisions survived. Keep a local decision record even if the underlying delivery evidence originates elsewhere, and specify how that record behaves when external evidence expires. An unavailable payload must not turn an old decision into an apparently unsupported new one.

Durability needs a precise scope here. For the proposed implementation, acknowledge an incoming event only after its evidence reference and intended state transition have been committed according to the chosen store's durability contract. Apply the suppression decision and mark the event processed in one transaction where the storage design permits it. If they live in separate stores, explicitly design recovery for the gap instead of calling the pair atomic. No particular database guarantee is assumed by this note.

Replay is useful. Resurrection is dangerous.

Keep secrets separate.

A replayed historical event should reproduce the historical decision under its recorded policy, or be evaluated in an explicitly separate simulation. It should not reactivate a destination because a newer policy happens to interpret the old reason differently. Likewise, a later successful delivery should not automatically erase a suppression record: require an authorized transition, preserve the previous evidence reference, and explain the new decision.

## Compare responsibility and evidence after defining the model

The following table describes responsibilities to investigate, rather than a ranking of services. The preference depends on which evidence your team can obtain and maintain, including evidence needed after an account dispute or a delivery-service migration.

| Decision area | Hosted OTP endpoint | Custom SMS send-code flow | Evidence to request |
| --- | --- | --- | --- |
| Challenge lifecycle | Confirm which creation, expiry, and checking rules the service owns | Specify and implement those rules in the application | State transitions and configurable limits |
| Duplicate requests | Verify documented retry and deduplication behavior | Define a durable request identity and retry policy | A repeated-request test with resulting records |
| Invalid destinations | Determine which outcomes are exposed and how they map to local policy | Normalize transport results before policy evaluation | Original event reference and mapping revision |
| Suppression ownership | Keep local eligibility policy explicit alongside remote challenge state | Keep eligibility policy explicit alongside local challenge state | Destination version and policy revision |
| Retention and export | Verify evidence availability and export boundaries | Operate the evidence store and its deletion process | Retention terms and a tested export or deletion procedure |

A hosted boundary is defensible when its documented outcomes and evidence access satisfy the marketplace's requirements and the team accepts the remaining uncertainty. A custom boundary is defensible when the application needs control that the candidate boundary does not expose, and the team is prepared to operate challenge generation, expiration, attempt limits, replay resistance, and secure secret handling. More application control also creates more application responsibility. Neither architecture earns an exemption from testing.

The trade-off is explicit: a hosted boundary is not suitable when required evidence cannot be exported or retained under the marketplace's policy. A custom flow requires the team to operate the challenge state securely, so it is not suitable when nobody owns that work. Its main limitation is the additional application responsibility; the hosted model's limitation is the evidence and control boundary established by its contract.

A send isn't verification.

Keep transport economics subordinate to these boundaries. The SMS segmentation reference distinguishes GSM-7 and UCS-2 encoding, with single-message limits of 160 and 70 characters respectively; its concatenated-message discussion explains why a longer message changes the segment count. Use the documented limits when testing actual template text, including localized variants, rather than treating character count as a stable proxy for delivered-message behavior. Verify applicable exceptions in the reference before turning those general limits into an estimator. This is a workload constraint, not a reason to select a provider.

## A minimal suppression decision with an explicit evidence requirement

The small Python example below implements only the destination-binding decision. It deliberately accepts a normalized policy reason, not a raw delivery-service status: the adapter that produces that reason needs its own documented mapping. The destination reference is an opaque application identifier, and the evidence reference points to a separately controlled record rather than containing an email address, phone number, or OTP.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Destination:
    ref: str
    channel: str
    version: int


@dataclass(frozen=True)
class Evidence:
    event_id: str
    destination: Destination
    reason: str
    record_ref: str


def suppression_decision(
    current: Destination, evidence: Evidence
) -> str:
    if evidence.destination != current:
        return "record_stale_destination_event"
    if not evidence.event_id or not evidence.record_ref:
        return "hold_for_evidence_review"
    if evidence.reason == "invalid_destination":
        return "suppress_destination"
    return "preserve_current_state"


current_email = Destination("buyer-417-email", "email", 2)
old_evidence = Evidence(
    event_id="event-9001",
    destination=Destination("buyer-417-email", "email", 1),
    reason="invalid_destination",
    record_ref="evidence-9001",
)
assert suppression_decision(current_email, old_evidence) == (
    "record_stale_destination_event"
)
```

An unrecognized reason preserves current state; it does not grant permission to send. Sending needs a separate eligibility check, including any existing suppression and the applicable contact policy. Keeping those decisions distinct prevents an incomplete event mapping from accidentally clearing an earlier block.

This function provides no event authentication, persistence, concurrency control, or OTP verification. Implement those at the appropriate boundaries before using the decision in a running service. In particular, verify event authenticity before normalization, deduplicate accepted events in durable storage, and make a decision conditional on the destination version still matching when the transaction commits. A version check performed only before a concurrent update leaves a race.

The corresponding failure test is concrete: replace destination version 1 with version 2 between event receipt and commit, then verify that the replacement remains eligible for its own policy evaluation and that the old evidence remains attributable to version 1. Also test duplicate delivery events, missing evidence references, unknown reasons, and a phone challenge arriving while the email destination is suppressed. Inspect resulting records as well as return values.

Test the race.

## Roll out through evidence reconciliation

Begin with observation: normalize events and compute proposed suppression decisions without changing contact eligibility. Review cases where the proposal disagrees with the current system, especially destination replacements and unknown reasons. Establish acceptance criteria before enabling writes; a neat dashboard is not evidence that the mappings are correct.

Then enable enforcement for a bounded marketplace cohort and reconcile three records: accepted evidence, suppression decisions, and subsequent contact attempts. Alert on missing evidence references, stale-version write attempts, and sends that contradict recorded eligibility. Keep event identifiers in operational diagnostics while applying access controls and retention rules to destination data. For rollback, stop the new evaluator from making decisions and preserve its committed evidence; do not automatically unsuppress recipients as part of a deployment rollback.

The endpoint choice follows from that exercise. Select the responsibility boundary whose documented behavior lets the team explain and reproduce contact decisions, then budget the operational work that remains. For this marketplace, a verification result without its destination binding and decision history is an incomplete handoff.

## References

The SMS character-limit documentation supports the encoding and segmentation discussion. The email documentation is background for the separate email-delivery channel; this note makes no product-specific claim about its suppression behavior. The storage model, sample identifiers, and rollout procedure above are proposed design choices.

## Sources

- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://resend.com/docs/introduction
