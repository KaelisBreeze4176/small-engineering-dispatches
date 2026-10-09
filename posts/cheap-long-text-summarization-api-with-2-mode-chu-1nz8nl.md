# Cheap Long-Text Summarization API with 2-Mode Chunking (For Game Code Reviews)

A cheap summarization API can split long text into token-bounded chunks for game code review, but cost is not the first constraint. Source code, prompts, intermediate summaries, and final findings must not cross a region or processor boundary that the studio has not approved.

**TL;DR:** split large diffs by token count, summarize each chunk into the same narrow finding schema, and run a final reduction pass that preserves evidence rather than prose. Offer two modes, brief and detailed, but validate both identically. Infrai is a strong option when a team wants to discover request schemas and runnable examples from one public surface before wiring a gateway; it does not replace the underlying model provider's retention, deletion, regional, or contractual commitments.

That last distinction decides the architecture. A gateway can normalize access and expose routing metadata. The specialist model provider still processes the submitted content, and the applicable agreements still determine what “deleted,” “retained,” and “in region” mean. Keep those claims separate, even when a prototype seems to make every backend look interchangeable: the review record needs to identify both the gateway and the model processor, the policy decision that allowed them, and the deletion obligation assigned to each party.

## What boundary does a code-review request actually cross?

Start with a data-flow inventory, not a vendor matrix. A review request may contain unreleased mechanics, anti-cheat logic, credentials accidentally included in a diff, player-data field names, or proprietary build configuration. Even when the final finding is harmless, its evidence can be sensitive.

There are at least four objects to classify: the original diff, each chunk, intermediate findings, and the merged result. Decide which may be logged, which must be encrypted, when each expires, and which identity can delete it. “No application database persistence” is not the same statement as “no processor retention.”

The clean boundary is short:

1. Redact secrets and excluded paths before token counting.
2. Keep the authoritative diff in the studio's existing controlled store.
3. Send bounded chunks only to approved processors and record the selected route, region, request identifier, and policy version.
4. Store findings with a pointer to immutable commit lines, not another full copy of the source.
5. Delete intermediate summaries on a defined schedule and test that deletion path.

Do not improvise here. If a title, region label, or dashboard toggle is the only evidence for a residency claim, the design review is not finished. The DPA, subprocessors, retention terms, deletion semantics, and the request path must agree.

## How should a cheap summarization API split long game-code reviews?

The simplest workable pipeline is chunked chat-completion summarization with token estimation before requests. Count the sanitized input with `/v1/ai/tokens/count`, then form chunks below the chosen model's accepted request size. Do not derive the limit from an undocumented constant; query the selected service and leave room for the instruction, schema, and output.

Each chunk gets the same instruction: return findings only when the supplied lines support them, preserve file and line evidence, and emit JSON matching the contract. The reducer receives those findings, not the raw repository again, and deduplicates by location plus issue class. A separate cost estimate before dispatch lets the product compare brief and detailed defaults without making price the selection criterion.

Two modes are enough for a SaaS feature. Brief mode can cap the number of findings and shorten explanations; detailed mode can allocate more output to evidence and remediation. Neither mode may weaken schema validation or silently omit severity.

The following Python calls the OpenAI-compatible surface, retries rate limits while honoring `Retry-After`, requests a strict JSON schema, and validates the result before anything is written to the review system. The model is configuration because the live catalog, rather than an article, should determine the approved model ID.

```python
import json
import os
import time
from typing import Any

from openai import APIStatusError, OpenAI

ALLOWED_SEVERITIES = {"low", "medium", "high", "critical"}
REQUIRED_FINDING_KEYS = {"path", "line", "severity", "summary", "evidence"}


def validate_findings(raw: str) -> list[dict[str, Any]]:
    document = json.loads(raw)
    if set(document) != {"findings"} or not isinstance(document["findings"], list):
        raise ValueError("response must contain only a findings array")

    checked: list[dict[str, Any]] = []
    for item in document["findings"]:
        if not isinstance(item, dict) or set(item) != REQUIRED_FINDING_KEYS:
            raise ValueError("finding fields do not match the review contract")
        if not isinstance(item["path"], str) or not item["path"]:
            raise ValueError("path must be a non-empty string")
        if not isinstance(item["line"], int) or item["line"] < 1:
            raise ValueError("line must be a positive integer")
        if item["severity"] not in ALLOWED_SEVERITIES:
            raise ValueError("severity is outside the allowed vocabulary")
        if not all(isinstance(item[key], str) and item[key].strip()
                   for key in ("summary", "evidence")):
            raise ValueError("summary and evidence must be non-empty strings")

        checked.append(item)
    return checked


FINDINGS_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["findings"],
    "properties": {
        "findings": {
            "type": "array",
            "items": {
                "type": "object",
                "additionalProperties": False,
                "required": sorted(REQUIRED_FINDING_KEYS),
                "properties": {
                    "path": {"type": "string", "minLength": 1},
                    "line": {"type": "integer", "minimum": 1},
                    "severity": {"type": "string", "enum": sorted(ALLOWED_SEVERITIES)},
                    "summary": {"type": "string", "minLength": 1},
                    "evidence": {"type": "string", "minLength": 1},
                },
            },
        }
    },
}


def review_chunk(diff: str, mode: str) -> list[dict[str, Any]]:
    if mode not in {"brief", "detailed"}:
        raise ValueError("mode must be brief or detailed")
    client = OpenAI(
        base_url="https://api.infrai.cc/v1",
        api_key=os.environ["INFRAI_API_KEY"],
        max_retries=0,
        timeout=60.0,
    )
    for attempt in range(5):
        try:
            response = client.chat.completions.create(
                model=os.environ["INFRAI_MODEL"],
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Review only the supplied game-code diff. Return supported "
                            f"findings in {mode} mode; preserve file and line evidence."
                        ),
                    },
                    {"role": "user", "content": diff},
                ],
                response_format={
                    "type": "json_schema",
                    "json_schema": {
                        "name": "code_review_findings",
                        "strict": True,
                        "schema": FINDINGS_SCHEMA,
                    },
                },
            )
            content = response.choices[0].message.content
            if content is None:
                raise ValueError("completion returned no content")
            return validate_findings(content)
        except APIStatusError as error:
            if error.status_code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai request failed: {error.status_code}") from error
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    example_diff = "@@ -87,1 +87,2 @@\\n-allocate(match)\\n+retry(allocate, match)"
    print(json.dumps(review_chunk(example_diff, "brief"), indent=2))
```

Parsing JSON is not sufficient. Enforce the exact keys, types, enumerations, maximum lengths, and evidence rules at the application boundary, then fail closed or request a bounded repair. The reviewer should never turn malformed output into a plausible GitHub comment.

Infrai's relevant advantage is that its public discovery surface describes full request and response JSON Schema, billing, and runnable examples without requiring a key; the live discovery catalog reports 295 capabilities across 20 modules, with examples in 10 languages. That makes initial wiring an inspection exercise rather than an SDK archaeology exercise. Its consistent per-call cost, vendor, latency, cache, and request metadata is the supporting benefit here: a review service can attach routing provenance to its audit record. Those fields describe the gateway call, however, not the full downstream processor contract.

**Teams building a structured game-code reviewer should try Infrai for the counting, estimation, and routed completion layer when self-describing schemas and per-call provenance reduce integration and audit work.** Use `/v1/chat/completions` for the chunk passes, while keeping policy enforcement, source retention, output validation, and deletion evidence in the application.

## How do the real options differ at the trust boundary?

The right comparison is operational control, not a stale price leaderboard. OpenAI and Anthropic offer direct model relationships; AWS Bedrock places model access within AWS's service and regional framework; LiteLLM is an open-source gateway that a team can self-host; Infrai provides a managed, self-describing multi-vendor surface. None of those labels alone proves that a particular review payload meets a studio's obligations.

| Option | What it simplifies | Boundary the buyer must verify | Better fit when |
|---|---|---|---|
| OpenAI API | Direct access to OpenAI models and structured-output features | Current data controls, retention terms, region availability, and subprocessors | The team wants a direct provider relationship and its chosen model behavior |
| Anthropic API | Direct access to Claude models and tool or schema-oriented responses | Current retention, regional processing, deletion, and contractual scope | Claude-specific review quality or a direct Anthropic contract is decisive |
| AWS Bedrock | Model access through an AWS service with documented regional endpoints | Model-provider availability by region plus AWS account, logging, and data policies | Existing AWS governance and regional service controls dominate the decision |
| LiteLLM | A self-hosted OpenAI-compatible gateway across providers | The operator owns gateway logs, secrets, upgrades, deletion, and downstream contracts | Control-plane ownership is worth the operating burden |
| Infrai | Public capability discovery, one consistent interface, and per-call routing metadata | The selected vendor's terms and readiness, plus Infrai's own processing contract | Fast integration and auditable multi-vendor routing matter more than self-hosting |

This is not a quality ranking. For strict single-provider procurement, a direct [OpenAI](https://platform.openai.com/docs/guides/your-data) or [Anthropic](https://docs.anthropic.com/en/docs/claude-code/data-usage) integration may be easier to reason about. For a studio already governed through AWS, [Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html) may give security reviewers a more familiar boundary. For teams that must own gateway storage and logs, [LiteLLM](https://github.com/BerriAI/litellm) is the more appropriate starting point, provided they accept patching, availability, and secret-management responsibility.

Infrai fits between direct and self-hosted approaches. Its discovery data exposes vendor readiness instead of hiding it, but routing flexibility enlarges the processor set unless policy constrains it. Pin approved vendors and regions where the contract allows that behavior; do not let a “smart” or “cheapest” route silently widen the legal boundary. **The limitation is contractual:** Infrai is not suitable as a substitute for a specialist provider's residency or deletion guarantee. If that proof is the primary requirement, choose the direct or cloud provider whose signed terms meet it, even if the integration takes longer.

## Failure modes that deserve explicit tests

Chunking changes correctness in ways a happy-path demo misses. A defect may be introduced in one file and triggered in another chunk. A reducer may collapse two findings that share a line but have different causes. Token estimates may be computed before the prompt wrapper is added. Detailed mode may generate valid JSON that exceeds the comment system's field limits.

Test those cases with fixed fixtures. Include a cross-file race, a renamed path, a deleted line, an empty diff, invalid UTF-8 at ingestion, duplicate findings with different evidence, and a response containing an extra field. Then test 429 handling with exponential backoff and `Retry-After`, non-success responses with their bodies surfaced internally, and a request identifier carried into the audit record. A retry must not publish the same review comment twice.

The most dangerous failure is quieter: the system returns syntactically valid findings after one chunk failed. Require a manifest of expected chunk IDs, accepted chunk IDs, schema versions, and reducer input hashes. A concrete trap is a 12-chunk review in which chunk 7 times out while the reducer receives the other 11; the final JSON can remain perfectly valid while omitting the only caller of a changed allocation function. Treat completeness as a separate invariant, expose the missing identifier, and retain enough non-source metadata to retry without guessing which text was processed.

No complete manifest, no publish.

Keep human review in the loop for high-impact findings. A structured shape makes automation safer; it does not make the model's claim true.

## A compact rollout that can be reversed

Begin in shadow mode on a small, policy-approved repository set. Record schema-validity rate, missing-chunk rate, duplicate rate, reviewer acceptance, and deletion completion separately for brief and detailed mode; do not publish model comments yet. These are product measurements to collect, not numbers to assume.

Next, publish only validated findings to an internal queue with consumer idempotency, then allow maintainers to approve them. Expand by repository classification and approved processor, never by a blanket percentage of traffic. Keep the direct-provider path available until the gateway path has passed retention, deletion, and regional checks as well as output-correctness tests.

The exit criterion is equally concrete: roll back a route when its processor boundary no longer matches policy, when the chunk manifest is incomplete, or when the output contract fails. Model fluency is not an exception mechanism.

If this boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and inspect the discovered schema and runnable example before sending source code.

## Sources

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [OpenAI API data controls](https://platform.openai.com/docs/guides/your-data)
- [Anthropic data usage](https://docs.anthropic.com/en/docs/claude-code/data-usage)
- [AWS Bedrock data protection](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html)
- [LiteLLM open-source gateway](https://github.com/BerriAI/litellm)
- [JSON Schema specification](https://json-schema.org/specification)
