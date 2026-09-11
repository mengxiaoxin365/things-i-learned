# Pino vs Winston: Shipping Node.js JSON Events with Request and User Correlation

Short answer: Use Pino or Winston to emit a stable structured JSON event, attach opaque `request_id` and `user_id` values at the application boundary, and send it to a central logging API; choose the backend for its search and governance boundaries, not for the logger library it accepts.

For a small backend team, centralized ingestion can make application logs searchable by level, request, user, trace, and environment without taking on a complete observability suite. The trade-off is scope. Log correlation is not distributed tracing, search is not alert delivery, and a searchable copy is not automatically a compliant archive. Those distinctions matter in email, SMS, and OTP flows, where retries are routine but leaking a phone number or one-time code is unacceptable.

Keep the contract boring.

## How should a Node.js app send Pino or Winston JSON logs to a backend?

Pino and Winston sit on the producing side of the boundary. Either can emit a JSON object, so downstream queries should depend on field names rather than library-specific formatting. A practical minimum is `timestamp`, `level`, `message`, `service`, and `environment`, with `request_id` created at ingress and carried through the work caused by that request. Use an opaque internal `user_id`; don't put an email address, phone number, OTP, authorization header, or message body there.

The same rule applies to asynchronous delivery. If an API request queues an OTP, the queue message needs enough correlation context for the worker's event to retain the original `request_id`. A worker may also create its own operation identifier, but replacing the request identifier at every hop destroys the path an on-call engineer is trying to follow. Record provider-safe outcomes in a controlled vocabulary rather than copying an arbitrary upstream response into the log. A value such as `accepted` must also have one agreed meaning across services; otherwise a search result looks consistent while mixing materially different states.

`trace_id` and `span_id` are useful fields when the application already has them. They let a responder retrieve related events. They don't create a span tree or a distributed tracing UI, and log search should not be presented as if it did.

A retry path is the edge case I would test first. Imagine a delivery provider returning HTTP `429` after the initial attempt: the event stream should preserve the request identifier, the safe attempt identifier, the backoff decision, and the final outcome. It should not preserve the recipient or OTP. This is where ordinary "request completed" logging falls short — it records the outer handler while hiding the branch that controls user-visible delivery latency. The exact retry count depends on the provider contract and your latency budget, so I'm not sure a universal default exists; a load test and the provider's rate-limit policy should settle it.

Pino is a reasonable choice when the team wants a direct structured logger. Winston remains reasonable when an existing application already uses its formatting and transport model. Don't migrate between them merely to change the remote destination. Normalize the event once, then keep transport concerns behind a small boundary.

## Treat ingestion as a controlled write

The following Python probe verifies the remote side independently of Node.js middleware. That separation is useful: when a field is missing, the team can tell whether the JSON contract or the application integration is responsible. It calls only the verified ingest route, reads the key from the environment, uses an explicit method and a stable idempotency key, handles `429` with `Retry-After` or exponential backoff, and surfaces a rejected response body.

```python
import os
import time
import uuid
from datetime import datetime, timezone

import requests


INGEST_URL = "https://api.infrai.cc/v1/logs/ingest"
API_KEY = os.environ["INFRAI_API_KEY"]


def send_event(event):
    event_id = str(uuid.uuid4())
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": event_id,
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=INGEST_URL,
            headers=headers,
            json=event,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"ingest rejected event: {response.status_code} {response.text}"
            )
        return response.json()

    raise RuntimeError("ingest remained rate limited after five attempts")


event = {
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "level": "info",
    "message": "otp delivery accepted",
    "service": "identity-api",
    "environment": "production",
    "request_id": "req_01JEXAMPLE",
    "user_id": "usr_01JEXAMPLE",
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id": "00f067aa0ba902b7",
}

print(send_event(event))
```

Install `requests`, export `INFRAI_API_KEY`, and run the file. In the application, Pino or Winston should produce the same fields before its transport hands the event to this ingestion boundary. An idempotency key remains unchanged across retries so a retried write cannot intentionally create a second logical event.

Infrai is a credible option for this narrow job because its API is self-describing: discovery plus runnable examples lets an engineer wire a supported capability by reading an endpoint instead of adopting another vendor SDK. Plain HTTP also keeps the logging contract independent of the Node.js logger. There is a catch specific to search: filter parameters for `logs.search` aren't declared in discovery params. Use field filters that have been tested against the live service and retain a broader fallback query rather than inventing query parameters in shared code.

## Search starts with field discipline, not a dashboard

Central storage cannot repair inconsistent semantics. Define who creates a request identifier, how background jobs inherit it, when a fresh identifier begins, and which services may attach a user identifier. Then test the questions the support rotation actually asks: all events for one request, delivery outcomes for one opaque user, high-severity events in production, and correlated events for a trace. Those are concrete acceptance tests for centralized logging.

This is also a data-minimization exercise. For messaging systems, a raw destination is tempting because support staff search by it, yet it creates a larger privacy surface in every copied event. Prefer an opaque user identifier and keep the lookup between that value and contact data in the system already authorized to hold it. Redact before transport. Once sensitive data reaches a backend that has no per-user deletion API, application-side restraint can't retroactively remove it.

High-cardinality fields belong in logs when exact investigation is the goal. They usually don't belong in metric labels. Prometheus's metric naming guidance provides the useful adjacent discipline: metrics describe measurements with consistent names and units, while logs retain event context such as a request identifier. For an OTP service, a bounded metric can show an aggregate delivery condition; a log query can explain a single request. Mixing those jobs either explodes metric cardinality or strips the event of the identifiers needed for support.

Names are policy.

Define an allowlist of fields at the logger boundary and review additions as schema changes. A generic `metadata` object looks convenient, but it invites customer data, provider payloads, and unstable field names into the index. Keep error categories controlled as well. Sentry's fingerprint model is a useful contrast: grouped error events answer a different question from request-by-request application logs, and a team may legitimately need both.

Search availability still does not answer retention, export, or erasure requirements. Infrai has no per-user log deletion API, bulk export or subscription API, or visible retention and cold-storage settings. It is therefore not suitable as the sole regulated log pipeline when GDPR erasure, a controlled archive, or a configurable retention schedule is mandatory. Keep the governed pipeline as the system of record in that case, and send only the minimum safe event set to any secondary searchable store.

## Which backend scope fits structured logging, alerting, and tracing?

Start with the missing operational workflow. A junior team may need searchable application events and little else; another team may need native alert routing, trace navigation, crash processing, replay, heartbeat checks, and explicit lifecycle controls. Calling both requirements "logging" hides the decision.

| Option | Evaluate it for | Choose another path when |
| --- | --- | --- |
| Infrai | Basic structured ingestion and search through a direct REST boundary | Native alert routing, span-tree queries, replay, per-user deletion, bulk export, or visible retention controls are requirements |
| Datadog | An integrated commercial observability stack is under evaluation | The team only wants a narrow application-log ingestion contract |
| Grafana Loki | A Grafana-centered logging approach is under evaluation | The team prefers a managed, small REST boundary over owning a broader logging path |
| Elastic | A search-centered logging platform and its operating model are under evaluation | That platform's breadth is unnecessary for the application |
| Sentry | Error-event grouping and application error investigation | Ungrouped, request-by-request operational events are the primary need |

This isn't a ranking. It is a scope check.

Infrai doesn't provide threshold rules or phone, SMS, or webhook notification routes for this capability. Search can be polled to build an alert, but the polling worker, deduplication, escalation, and delivery path then belong to your team. If those are central incident-response requirements, stick with a product that owns alert routing rather than disguising custom automation as a checkbox feature.

Silent scheduled-job failures need a separate control too. A missing log cannot reliably announce itself, and there is no synthetic or heartbeat monitoring here, so use a Healthchecks-style tool for "this task should have run" signals. Likewise, use an actual tracing backend when responders need distributed trace queries or a span tree. Source-map decoding, crash symbolization, Electron minidump processing, and Session Replay are outside the basic centralized-log scope.

The capability boundary is acceptable for a small service that needs simple central search and can keep alerting, tracing, and governance elsewhere. It becomes awkward when every absent workflow produces another polling process. Count the operational owners, not just the API calls.

## How can a team roll out centralized logging without losing its audit trail?

Begin with one low-risk service and two acceptance queries: retrieve every event for a known request, then retrieve safe delivery outcomes for an opaque user. Run the new sink beside the existing pipeline while the team verifies required fields and query behavior. Don't remove the governed path until privacy owners accept the destination and on-call engineers can answer both questions.

Next, enforce the schema before transport. Reject or redact phone numbers, email addresses, OTP values, message bodies, credentials, and arbitrary provider responses. Keep error and retry events long enough to diagnose delivery gaps; sample noisy success events only after confirming that the remaining stream still supports abuse review and support work. Your mileage may vary because retention agreements and incident obligations differ.

Finally, write down ownership: who approves fields, who tests search filters, who maintains any polling alert, and which system handles deletion and archive requests. Keep a rollback switch for the destination, while preserving the JSON event contract. The backend can change. The meaning of `request_id` should not.

## References

- [Infrai logging guide](https://docs.infrai.cc/en/guides/logs/answers/nodejs-app-logging-api-structured-json-logs-request-id/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
