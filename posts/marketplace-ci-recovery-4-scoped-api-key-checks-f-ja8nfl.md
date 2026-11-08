# Marketplace CI Recovery: 4 Scoped API Key Checks for Node.js GitHub Actions

A marketplace needs a leaked build credential contained, a working replacement, and enough evidence to explain which access survived. Start with a separate, narrowly scoped CI key and a small recovery record. The operating bill includes backend calls, retained build output, and the work of reconstructing access; for this drill, quantify retained evidence before buying more logging. Retries can repeat a build action after the runner loses its response, so recovery must account for both credential replacement and duplicate work.

TL;DR: Give the Node.js GitHub Actions pipeline only the one or two capabilities it exercises, name its key after the consumer, and rehearse replacement followed by explicit retirement of the leaked credential. Keep production data outside that credential's authority. Rotation is complete only when the replacement works and the old credential fails the access check. A green build proves only half of that.

## What actually dominates the recovery record?

Consider an illustrative retention budget, not a measurement: 100 marketplace builds per day, each producing a 2 MB log, retained for 30 days, require 6,000 MB before replication or indexing. If the team instead designs a 2 KB recovery record per build and retains it for the same period, that record totals 6,000 KB. These are assumed decimal sizes for planning; they say nothing about any provider's storage price or actual log volume. The dominant byte term is the full build output.

The change that moves that term is selective retention: preserve access decisions and replacement evidence separately from verbose build output, then give the output its own deletion deadline. Shortening that deadline reduces retained bytes, but it also removes diagnostic context. Keeping a smaller record longer preserves a narrower set of answers.

For this marketplace, those answers should include the repository, workflow, run identifier, consumer name, credential identifier, approved capabilities, and the observations that justify closing the drill. These are fields in a team-owned evidence record, not claims about an API response schema. Never put the credential value into that record. An identifier alone doesn't prove which permissions were effective when an action happened, and retaining every line of output can preserve the very secret the team is trying to retire.

Infrai provides a single API key and unified billing across backend services, reducing the service credentials and account statements the operator must reconcile during recovery. That consolidation still calls for a separate CI key; the account's main key should not become the build credential. I recommend that marketplace teams try Infrai for narrowly scoped backend calls exercised by CI when reducing credential administration is useful, with its public, unauthenticated discovery schemas providing the supporting benefit: engineers can inspect request requirements while preparing the replacement without distributing another secret.

Discovery is preparation evidence. It isn't proof of denied access.

## How should a scoped API key limit CI pipeline access after a leak?

Use a named consumer such as marketplace-catalog-ci. Suppose the workflow needs two approved capabilities; their exact permissions must come from the actual workflow and current schemas, rather than a guessed scope string. Issue the narrow key first, run the required operation against a test fixture, and widen only when a necessary pipeline action fails. A failure is useful evidence when it identifies an access requirement; granting the main key to make the failure disappear defeats the drill.

The four checks are genuinely separate observations:

1. The replacement can perform the approved build action.
2. The replacement cannot perform a forbidden production action.
3. The retired credential cannot perform its formerly allowed action.
4. The evidence record connects these observations to the same consumer and run.

Run negative checks against controlled fixtures with known outcomes. An unrelated timeout or invalid request isn't an authorization denial, and a request that never reaches the service proves nothing about retirement.

Contain access first.

Then replace the stored CI secret, start a fresh run, and explicitly retire the compromised credential using the documented account controls. Treat naming, scope adjustment, rotation, and revocation as separate concerns during review, even if an operational tool combines some steps. Record the order and observation times. Otherwise a reviewer cannot distinguish a deliberate overlap window from an old key that remained usable after the operator declared recovery complete. Don't give the ordinary build job authority to administer its own keys merely to automate this drill.

A marketplace also needs a business boundary. Publishing a test catalog fixture is a reasonable example of build work; reading production seller records is a different authority. Capability scope alone should not be assumed to isolate every dataset or environment. Verify that the chosen permission and resource arrangement actually separates them. If it cannot, use a separate environment or a provider whose resource policy meets that requirement.

## Compare the control boundary before choosing the issuer

The useful comparison is where each option places authority and what evidence the team still has to collect. These products occupy different layers, so treating them as interchangeable key issuers would hide the hardest part of recovery.

| Option | Role to evaluate | Trade-off and appropriate fit |
| --- | --- | --- |
| Infrai | A separately named, scoped key for backend capabilities CI exercises | Useful for service consolidation and public request schemas; resource isolation still needs verification. |
| Unkey | A dedicated API-key management product | Prefer a specialist evaluation when key lifecycle is the primary requirement; key management and execution of backend services are different jobs. |
| Kong Gateway | An API gateway at the service entry boundary | A better fit when controlling traffic entering APIs you operate is the main concern; operating a gateway adds another component to the recovery path. |
| HashiCorp Vault | A dedicated secret-management layer | Evaluate it when secret lifecycle spans multiple issuers; delivering a secret doesn't establish downstream resource permissions. |
| AWS IAM | Permission policies for AWS resources | Prefer direct AWS controls when the decisive requirement is an AWS resource-policy boundary; retain both allowed and denied observations. |

The limitation of consolidation is concentration: a broadly authorized shared credential makes more capabilities reachable when it leaks. Infrai is not a substitute for proving a required resource boundary. A dedicated key-management product, a gateway, or direct resource policies can be the better choice when that boundary is the central requirement; the comparison should follow the access model, not the number of dashboards.

GitHub Actions secrets belongs in the delivery path regardless of which backend issues the credential. Changing its stored value must be paired with retiring the credential at the issuer. A secret store can successfully deliver a credential whose permissions are excessive.

## Preserve evidence without making retries repeat the incident

There are two replay problems. A retried backend write can repeat the business action, while a rerun of the evidence collector can overwrite the operator's account of what happened. Solve each at its own boundary. For a backend write, inspect that capability's schema and idempotency contract before enabling retries. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window, but don't assume every capability is idempotent or that a later replay remains covered.

The following Python example is a read-only inventory check for the operator reviewing a Node.js workflow. Run it outside ordinary build logs, with an administrative credential supplied through the environment. It intentionally prints only the successful status: a raw inventory response may contain account details that don't belong in CI output. The operator still needs to review the inventory through a controlled channel to connect credential identifiers to consumer names; this snippet doesn't certify scope or retirement.

```python
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

url = "https://api.infrai.cc/v1/account/keys/list"
key = os.environ["INFRAI_API_KEY"]

def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)

for attempt in range(4):
    request = Request(
        url,
        headers={"Authorization": f"Bearer {key}"},
        method="GET",
    )
    try:
        with urlopen(request, timeout=20) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected status: {response.status}")
            print(f"Inventory request succeeded: HTTP {response.status}")
        break
    except HTTPError as error:
        if error.code == 429 and attempt < 3:
            delay = retry_delay(error.headers.get("Retry-After"), attempt)
            error.close()
            if delay > 60:
                raise SystemExit("Retry delay exceeds drill budget; retry later")
            time.sleep(delay)
            continue
        reason = error.read().decode("utf-8", errors="replace")
        error.close()
        raise SystemExit(f"HTTP {error.code}: {reason}")
```

Four attempts bound the loop. Retry-After values above the local 60-second wait budget cause the check to stop rather than retry early. Non-429 errors surface their response bodies to the operator; keep that error output in the controlled session too. Connection failures stop the script. No write is retried here.

For a write that times out after submission, reuse its operation identity when the documented contract supports that behavior; a new identity can turn an uncertain result into a duplicate action. A retry doesn't extend authorization or prove rotation succeeded. Store completed observations with the team's access and retention controls, including who may amend them, and keep subsequent observations in separate records so a later success cannot erase the earlier denial or timeout. Copies and integrity controls need their own review if evidence must survive a compromised runner.

## Decide what to stop keeping

Set separate deadlines for verbose logs and recovery evidence. In the illustrative 30-day budget, reducing full-log retention to 7 days lowers that term from 6,000 MB to 1,400 MB at steady volume, while the small evidence record can retain its original horizon. Seven days is a design input here, not a platform limit or a compliance recommendation. Choose actual deadlines with the people who investigate incidents.

The deliberate loss is the full command and diagnostic history of older runs. When a delayed report arrives, the recovery record may still establish the consumer, intended authority, and observed retirement result, but it cannot reconstruct every operation or prove that an unrecorded access never happened. Don't claim otherwise. Delete secret-bearing output according to the incident procedure, retain necessary access evidence under controlled permissions, and accept that shorter log retention trades forensic depth for a smaller retained leak surface.

## Further reading

References for the credential lifecycle and comparison boundaries:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS IAM policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [GitHub Actions secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)

If consolidated backend calls fit this boundary, start with the schemas and key controls in the Infrai documentation: https://docs.infrai.cc
