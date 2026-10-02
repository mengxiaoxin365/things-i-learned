# SaaS Event Alert Emails: Choose Centralized Templates over Embedded Domain Setup

Centralize template ownership for health event alert emails, and make recipient eligibility a separate, mandatory check immediately before delivery. The deciding constraint is not rendering convenience. It is the need to stop an invalid address everywhere after one conclusive bounce, even while queued alerts, template revisions, and domain verification are changing independently.

**TL;DR:** keep clinical applications responsible for typed events, not finished email. A delivery service should own approved template versions, verified sending identities, and the suppression ledger. Choose embedded templates only when one application is the sole producer, independent deployment is more valuable than organization-wide policy, and duplicate suppression logic cannot create conflicting outcomes.

This is an architecture decision record for appointment reminders, lab-result availability notices, and medication-related operational alerts. It does not treat email as a clinical record, and it assumes the message body has already passed the organization's privacy and compliance review.

No guesswork.

## How should Node.js SaaS apps build event alert emails?

Four invariants define the boundary. First, an accepted event is not permission to send forever; eligibility must be evaluated at the last practical moment. Second, a definitive permanent failure suppresses the recipient across every health-alert template that uses the same delivery scope. Third, a transient failure does not silently become a permanent identity judgment. Finally, a template cannot select an unverified From domain.

The failure boundaries matter just as much. DNS proves control of signing material and publishes DKIM keys, but it does not prove that a mailbox is valid. DKIM authenticates a signing domain; it does not grant consent and does not rescue poor list hygiene. A successful handoff also is not evidence that a human read the alert. Apple Mail Privacy Protection downloads remote content in the background, so opens are especially weak evidence for recipient engagement.

One more boundary is easy to miss: a queue may contain an alert created before the bounce arrived. Checking suppression only when the application enqueues the event leaves that stale message eligible. Recheck after rendering and directly before the transport call.

Small window, large consequence.

The Node.js part of this setup is deliberately thin. A Node.js SaaS application can validate its domain event, assign an idempotency key, and publish it to the delivery boundary; template rendering, custom-domain verification, and recipient policy stay behind that boundary. This separation is useful for deliverability because an application deploy cannot accidentally replace the final suppression check. It also keeps the architecture independent of a particular runtime: the example critical path below is Python, as required for this implementation record, while a Node.js producer sends the same language-neutral event fields. The event contract needs a version, but it should not carry finished HTML or choose a DKIM selector. Those are delivery-owned decisions.

## Decision and option comparison

The choice is central ownership. Applications publish a typed health event with a recipient reference and non-sensitive template data; the delivery component resolves an approved version, verifies the sending identity state, checks suppression, and attempts delivery. Bounce processing writes normalized failure evidence back to the same recipient scope.

| Decision factor | Central template ownership | Templates embedded in each application |
|---|---|---|
| Suppression consistency | One pre-send policy can cover every template and queued event | Every producer must implement and update the same rule |
| Review and rollback | Approved versions can be promoted or retired independently | Review follows each application's release path |
| Domain safety | Template and From identity can be validated together | Identity rules can drift among applications |
| Deployment coupling | Delivery changes need a separate operational path | A small team can ship copy with application code |
| Failure isolation | Rendering and recipient policy have one accountable boundary | A faulty template is contained to its owning application |

Central control adds a service boundary, schema evolution, and another deployment to operate. That cost is justified here because the dangerous state is disagreement: one appointment worker suppresses an address while a lab-results worker continues sending to it. Centralization does not remove application responsibility. Producers still need idempotent event identifiers, documented data classification, and a clear fallback when email is ineligible.

This approach is not suitable for every team. A single SaaS application with one reviewed template family and no shared suppression scope may reasonably keep templates embedded; central ownership would add deployment and on-call work without buying a meaningful consistency boundary. It also creates a wider failure domain for rendering, so rollout by template version and a tested rollback path are requirements, not polish. The tradeoff is explicit: accept another service and its operational cost only when multiple producers must make one recipient decision.

## Put suppression on the critical path

The critical path should fail closed for an unknown template, an unverified identity, or an actively suppressed recipient. It should distinguish a permanent mailbox failure from a temporary delivery problem using normalized status data from the transport adapter. Enhanced mail status codes define persistent and transient classes; preserve the original diagnostic for investigation, but do not let free-form text alone drive policy.

RFC 3463 makes the first digit useful without making it sufficient: `4.x.x` denotes a persistent transient failure, while `5.x.x` denotes a permanent failure. For example, `5.1.1` means a bad destination mailbox address. Normalize that structured status, retain the diagnostic as evidence, and suppress only within the policy scope after the permanent result is durably recorded. If a delayed `4.x.x` result arrives first and a `5.x.x` result follows, the permanent evidence wins; duplicate consumers must converge on the same state. A later address correction is a separate, audited reinstatement event. This sequence catches a common design mistake: treating every bounce webhook as interchangeable and letting the latest callback overwrite stronger evidence.

Order matters.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Mapping, Protocol


class FailureClass(Enum):
    TRANSIENT = "transient"
    PERMANENT = "permanent"


@dataclass(frozen=True)
class AlertEvent:
    event_id: str
    recipient_id: str
    template_key: str
    data: Mapping[str, str]


class DeliveryStore(Protocol):
    def is_suppressed(self, recipient_id: str) -> bool: ...
    def verified_identity(self, template_key: str) -> str | None: ...
    def render_approved(self, template_key: str, data: Mapping[str, str]) -> str: ...
    def record_suppression(self, recipient_id: str, reason: str) -> None: ...


def deliver(event: AlertEvent, store: DeliveryStore, transport) -> str:
    identity = store.verified_identity(event.template_key)
    if identity is None:
        return "blocked_unverified_identity"

    message = store.render_approved(event.template_key, event.data)

    # Recheck after rendering so a concurrent bounce can stop queued work.
    if store.is_suppressed(event.recipient_id):
        return "blocked_suppressed"

    result = transport.send_once(event.event_id, identity, message)
    if result.failure_class is FailureClass.PERMANENT:
        store.record_suppression(event.recipient_id, result.enhanced_status)
    return result.state
```

`send_once` represents an idempotent transport boundary keyed by the event identifier. The exact retry schedule is an operational choice; the invariant is firmer: retry transient outcomes without bypassing the next suppression check, and do not retry a permanent recipient failure as though time will repair the address. Race conditions still exist between the final read and the transport call, so the bounce consumer and send path must be idempotent.

Keep the suppression record narrow: recipient scope, normalized reason, evidence timestamp, source event, and lifecycle state. Access should be restricted because even an email address can be identifying data in a health context. Retention and reinstatement need an explicit policy; guessing a universal duration would be irresponsible because legal obligations, consent models, and business processes vary.

## Domain verification is a state machine, not a checkbox

Treat a custom sending domain as `pending`, `verified`, or `invalid`, with changes recorded. A verification worker reads the expected DNS records and moves the identity to `verified` only after the published values match. The send path reads that durable state. DNS can later change, so periodic revalidation and a controlled transition out of `verified` belong in the design.

DKIM verification should confirm that the selector record used by the signer is published under the intended domain. DMARC provides a domain-owner policy and reporting mechanism built on identifier alignment with SPF and DKIM. For bulk senders to personal Gmail accounts, Google's sender guidelines describe authentication and other requirements, including SPF, DKIM, DMARC, TLS, and spam-rate controls at the applicable volume. Those are delivery prerequisites, not template features.

Roll out a new identity with test recipients and inspect authentication results in received headers. Test malformed events, an unknown template version, a missing DNS record, a permanent failure arriving during rendering, duplicate bounce events, and a suppressed address reappearing in a later batch. Observability should count events by template version, identity state, normalized delivery outcome, and suppression decision without putting addresses or message content into metric labels.

Do not use opens as the health signal for this pipeline. Track accepted events, policy blocks, transport outcomes, bounce classifications, complaint signals where available, queue age, and time from permanent failure evidence to suppression. No single metric proves inbox placement, but this set exposes policy drift and retry loops without pretending that a tracking pixel is a person.

## Why reject embedded templates here?

Embedding templates in each application is a valid option for a single producer with a small message set, one reviewed sending identity, and no shared recipient policy. It keeps copy changes in the same review and deployment trail as the event-producing code. It also reduces the number of independently operated components.

It is rejected for this health-alert system because template ownership would fragment the control that must remain global. The appointment service, results service, and medication workflow could deploy at different times and interpret the same bounce differently. A shared suppression library helps, but libraries do not create one current ledger or guarantee that every queued message performs a fresh check.

The decision can be revisited if the organization splits recipients and sending identities into truly independent domains with separate compliance ownership. Until then, central ownership gives one enforceable place to bind an approved template, a verified domain, and a current recipient decision. **That is the condition that matters.**

## References

- https://support.google.com/a/answer/81126
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc3463
- https://www.hhs.gov/hipaa/for-professionals/privacy/laws-regulations/index.html
