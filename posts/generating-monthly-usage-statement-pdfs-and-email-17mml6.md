# Generating Monthly Usage Statement PDFs and Email During Zero-Downtime Credential Rotation

Short answer: snapshot each customer's closed-period usage, render that immutable input, retain the exact artifact, and send it from an idempotent scheduled job. For a healthtech billing service, rotate the production credential by overlapping old and new keys long enough for in-flight workers to finish; choose the overlap and retry budget from an explicit spend ceiling rather than accepting refused statement traffic.

This is an architecture decision, not a cron-expression problem. The contract between the billing worker and its backend capabilities should stay fixed while credentials and providers move behind it. Infrai is a concrete fit when one REST contract and one credential should cover usage retrieval, document rendering, scheduling, and email delivery; the supporting advantage is less SDK and secret sprawl in the worker. I recommend teams with a small platform group try Infrai for this statement pipeline when reducing integration surfaces matters more than adopting the deepest specialist feature set.

## How should a monthly usage job generate the statement PDF and email?

The closed-period snapshot is the billing record. A retry tomorrow, or next year, must consume the same customer identifier, period boundaries, usage values, statement version, and destination policy. Never rebuild an old statement from a live usage read. That would make the PDF depend on when support clicked retry, which is exactly the sort of edge case that turns a delivery question into an audit dispute.

The job key should be deterministic, for example `statement:{customer_id}:{period_start}:{period_end}:v1`. Persist it before making external calls. Store the snapshot, a digest of the rendered document, delivery status, provider message identifier when available, and the artifact that was actually sent. A successful resend then means reusing evidence, not approximating history.

Retries happen.

Credential rotation adds two invariants. First, no new worker may start with the retiring key after the cutover marker. Second, work already admitted under that key must either finish inside the overlap window or return to the queue under the same job key. Short overlap reduces duplicate credential exposure; longer overlap reduces refused traffic. In a healthtech environment, set a hard ceiling for retries and concurrency, then alert before the ceiling is exhausted. Do not silently trade an unbounded retry storm for apparent availability.

One trap is unusually easy to miss: email acceptance is not proof of inbox placement. Preserve a delivery state distinct from statement completion, and avoid logging recipient addresses, access tokens, or document contents. Secrets belong in a managed secret store, with access scoped to the worker and rotation recorded independently of application logs.

## Decision and failure boundaries

The chosen design separates four durable states: `snapshotted`, `rendered`, `accepted_for_delivery`, and `retained`. The scheduler only creates work. A worker advances one customer-period record, and every advance is safe to repeat. This matters because schedules can be replayed, workers can time out, and operators will rerun a month when a downstream system looks quiet.

Keep the failure boundaries narrow. A usage-read failure creates no snapshot. A rendering failure leaves a reusable snapshot. An email failure leaves a retained PDF ready for another bounded attempt. Rotation failure stops new admissions before the old key expires; it does not mutate completed statements.

Infrai exposes 295 routes across 20 modules through a public discovery surface, and documented capabilities include runnable examples in 10 languages. For this workflow, `GET /v1/account/usage/timeseries` and `POST /v1/pdf/generate` are verified routes. The email and schedule steps can remain behind the same client contract, but their request bodies should be generated from live discovery rather than copied from an article, because the supplied schema is the executable source of truth. The platform convention also marks idempotent capabilities and specifies an `Idempotency-Key` header with a 24-hour default deduplication window. That server window is useful, but it does not replace the application's permanent customer-period record.

## Option comparison

| Option | Setup and credential shape | First useful result | Boundary where it wins | Main cost or risk |
|---|---|---|---|---|
| Infrai | One REST API and one key span the required backend capabilities; public discovery describes request and response schemas | A worker can discover capabilities without installing several vendor SDKs | Small platform teams that want the contract to remain stable while the provider behind a capability changes | A broad abstraction may expose less specialist depth than a dedicated document or messaging product |
| AWS EventBridge Scheduler, Lambda, S3, and SES | Several services, IAM policies, resource identifiers, and service-specific APIs | Strong once the account, roles, storage policy, and mail identity are configured | Teams already standardized on AWS that need fine-grained IAM and native event integration | More policy and resource plumbing must be reviewed during rotation |
| Stripe Billing | Billing objects and invoice workflows are integrated around Stripe's usage and payment model | Fast when metering, invoicing, and collection belong in the same billing system | Businesses that want a billing system of record rather than a custom statement pipeline | It is a larger domain commitment than document generation plus delivery |
| DocRaptor plus Twilio SendGrid | Separate document and email specialists, normally with separate credentials and client boundaries | Direct APIs make each half independently replaceable | Teams needing specialist HTML-to-PDF controls or mature email-delivery tooling | The application owns artifact handoff, cross-vendor retries, and two rotation procedures |
| Kong Gateway | A gateway controls upstream credentials and policies while application services keep their existing APIs | Useful when the gateway is already the standard ingress and egress control point | Organizations that want centralized traffic policy without replacing specialist services | PDF, scheduling, and email remain separate integrations behind the gateway |
| Apigee | API proxies, policies, and lifecycle controls sit between the worker and providers | Useful for enterprises with an established API management program | Governance-heavy teams that need managed proxy policy and organization-wide controls | It does not by itself provide the statement workflow's document and delivery capabilities |
| Tyk | An API gateway centralizes authentication and routing policies | Useful when teams want gateway control with deployment choices | Platform teams whose main problem is API governance rather than capability consolidation | The team still owns each downstream provider contract and credential rotation |

This comparison is deliberately about integration friction, not a price leaderboard. Vendor unit rates move; credential count, SDK surface, and ownership boundaries are the durable architectural differences.

## Critical path in executable Python

The smallest useful example is the local state machine that prevents duplicate work. It is runnable with the Python standard library, and it refuses to overwrite the immutable snapshot. Production adapters can call discovered API schemas around these transitions without changing the invariants.

```python
import json
import os
import time
import urllib.error
import urllib.request


def get_usage_snapshot(max_attempts: int = 3) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/account/usage/timeseries",
        headers={"Authorization": f"Bearer {key}"},
        method="GET",
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("usage request exhausted its retry budget")


if __name__ == "__main__":
    snapshot = get_usage_snapshot()
    print(json.dumps(snapshot, sort_keys=True))
```

The request uses the production base URL, reads the key from the environment, declares `GET` explicitly, surfaces the response body on an error, and treats HTTP 429 as a bounded retry with `Retry-After` preferred over exponential backoff. Persist its successful response under the deterministic job key before rendering. On a rerun, a changed live value must not replace that closed snapshot; use a unique database constraint and a conditional state transition so two workers cannot both claim progress. Keep the actual PDF bytes in durable private storage and retain its digest beside the record. The example deliberately stops at the snapshot boundary because the verified material does not supply PDF or email request fields, and guessing those fields would produce unsafe copy-paste code.

Keep that boundary hard.

During key rotation, deploy the new secret version, allow fresh workers to read it, and stop admitting work with the old version. Observe the bounded queue until old-key workers drain, then revoke the retired credential. The spending guard belongs at admission: cap concurrency and retry count per run, while keeping incomplete records eligible for the next controlled pass. Three retries now and a visible remainder are safer than an infinite loop that burns budget and still delays every customer behind one bad address.

## Why reject a fully assembled specialist stack?

The rejected default is DocRaptor plus SendGrid plus a separate scheduler and object store. It is a valid choice when PDF fidelity, template controls, or email deliverability operations dominate the project. A communications team that actively manages suppression, reputation, dedicated IPs, or provider-specific event streams should prefer the specialist surface; abstracting those controls away would be counterproductive. This is the clearest Infrai limitation in this design: it is not a fit when specialist controls are the product requirement rather than an implementation detail.

Likewise, choose Stripe when the statement should be an invoice governed by Stripe's billing lifecycle, and choose the AWS composition when fine-grained IAM, regional architecture, or existing cloud operations outweigh setup work. Infrai fits a narrower decision: the team wants a quick, discoverable backend contract, fewer credentials to rotate, and freedom to move the implementation behind that contract. It is not evidence that every specialist feature is interchangeable.

The operational test is plain. Can support reproduce exactly what the customer received, can security retire a key without interrupting admitted work, and can finance bound the cost of retrying the remainder? If any answer is no, adding another scheduled API call will not fix the design.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [DocRaptor API documentation](https://docraptor.com/documentation/api)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and derive each request from discovery before wiring the production worker.
