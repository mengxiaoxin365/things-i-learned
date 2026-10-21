# Postgres Marketplace Login OTP — Choosing Email Code or Magic Link During SMS Outages

TL;DR: When SMS is unavailable, an email verification code is usually the less surprising recovery path for a marketplace because it works across devices and avoids link scanners consuming a credential. A magic link can reduce typing on the same device, but its GET request must never complete authentication. In either case, email proves control of a mailbox; it is not a drop-in equivalent to a possession factor. NIST SP 800-63B explicitly excludes email from out-of-band authentication.

Own the template, challenge state, and audit schema in the application. Let the delivery service transport a versioned render. Use the same security decision in the US and EU, then vary retention, disclosure, and localized content through reviewed policy rather than regional authentication folklore.

That answer matters in a marketplace with two adjacent flows. A seller must receive a compliance notice with an auditable delivery record, and the seller may need to regain account access when SMS cannot arrive. Reusing one generic notification template for both looks efficient. It quietly mixes legal copy, authentication secrets, retention periods, and proof claims that have different owners. The durable boundary is template ownership.

## Should email fallback replace an unavailable SMS login OTP?

A fallback changes the threat model. SMS out-of-band authentication and access to an email inbox rely on different accounts, recovery procedures, devices, and attack surfaces. If a password reset or marketplace login already depends on that inbox, sending the second step there can collapse two nominal factors into one recovery domain. Calling the result "2FA" in logs or user copy creates a stronger assurance claim than the system can defend.

NIST SP 800-63B is unusually direct: email shall not be used for out-of-band authentication. That does not prohibit every email sign-in flow. It means an engineering team should classify email recovery as a lower-assurance route, apply a documented risk decision, and avoid representing it as equivalent to an approved authenticator. High-impact actions such as payout-bank changes, tax-identity edits, or exporting buyer data should require an independent authenticator or a staffed recovery process after email fallback. A user who only needs to read a compliance notice may proceed under a different policy.

Do not trigger fallback merely because an SMS has not produced a success event after a few seconds. Delivery receipts have channel-specific meanings, and delayed delivery is not the same as permanent failure. The state machine needs explicit reasons such as user-selected recovery, an invalid or unreachable phone destination, or a policy-approved timeout. It also needs a loop guard. Otherwise SMS and email retries can alternate while producing several valid secrets.

One challenge, one purpose, one account. Keep those boundaries boring.

The downgrade must be visible.

US and EU deployment does not change whether a six-digit code or a link proves control of a mailbox; both arrive through the same mailbox. Regional law and policy can change the lawful basis for processing, required notices, data-subject handling, retention, and which events counsel wants preserved. The GDPR principles of purpose limitation and data minimization argue against retaining raw credentials or a full email body merely because the audit database has room. Legal review must set the actual schedule for the marketplace and jurisdiction.

## Own the message contract before choosing the interaction

Template ownership belongs with the team that owns authentication and compliance semantics, not with a transport adapter. That team should version the subject, plain-text body, HTML body, required variables, locale, purpose, and approval metadata in source control. A provider-side template can still be a deployment artifact, but it should be generated from or checked against that canonical contract. Otherwise a dashboard edit can change security wording without a code review, and a transport migration can subtly change which variables are escaped or omitted.

A logic-less renderer such as Mustache is useful precisely because it limits what a template can decide. It does not make untrusted HTML safe by itself. Validate every input before rendering, escape according to the output context, and keep policy branches in application code. The template should receive `marketplace_name`, `masked_account`, `expires_at_text`, `support_url`, and either a code or a confirmation URL. It should not receive an entire user object.

Less data, fewer surprises.

Compliance notices deserve a separate template family from login recovery. Record the notice type, template version, locale, recipient identity reference, rendered-content hash, transport message identifier, submission time, and later transport events. Preserve the approved source and enough rendering inputs to reproduce the notice under the retention policy. Do not store an OTP, magic-link token, or token-bearing rendered body in that record. Also label evidence honestly: "accepted by transport" and "reported delivered" do not prove that a human opened, understood, or legally received a message.

The authentication audit record is smaller and shorter-lived. It needs a challenge identifier, account reference, purpose, route transition, template version, issue and expiry times, attempt count, terminal outcome, and coarse failure reason. Secrets should be represented by a keyed verifier or cryptographic digest appropriate to the design, never by plaintext. Access to both audit streams should itself be logged.

## Code or link is an operational choice, not an assurance upgrade

Both options prove access to the destination mailbox. The useful differences are interaction shape and failure modes.

| Constraint | Email code | Magic link |
|---|---|---|
| User changes from phone to desktop | Easy to transcribe | May open on the wrong device |
| Automated link inspection | No credential in a clickable URL | GET may be fetched before the user acts |
| Phishing resistance | User can type a code into a spoofed page | User can follow a spoofed link |
| URL leakage | None for the secret | Token may reach history, logs, or referrers if mishandled |
| Accessibility and typing | Requires short manual entry | Reduces typing on the intended device |
| Binding to an existing browser flow | Challenge ID can bind the form | Token and final POST need binding to the intended flow |

For a broad marketplace audience, a code is a defensible default when cross-device completion is common. Six decimal digits provide one million possible values, which is not enough protection without a strict attempt limit, short lifetime, one-time consumption, and rate limits by account, destination, challenge, and network risk signal. The precise lifetime and limits are policy choices; publish them in user copy and keep them consistent with server state. A resend should not leave a fan-out of valid codes. Either return the same active challenge safely or revoke the previous challenge atomically.

A magic link is reasonable when the expected path stays on one device and the product can handle scanners correctly. The emailed URL should open an informational landing page. Authentication completes only after an explicit POST, with the one-time token checked server-side and bound to the intended account, purpose, and challenge. Avoid putting account data in the URL. Set a restrictive referrer policy on the landing page, load no third-party resources, and make replay fail.

This is the trap: consuming the token on GET makes security software and mailbox previews indistinguishable from the user. A redirect does not fix it. The first request still spent the credential.

Teams often start by treating that first GET as the user's click. Scanner traffic exposes the mistake: the mailbox reports a fresh message, the user opens it seconds later, and the server says the credential was already used. Extending the expiry cannot repair that sequence. Neither can sending a second link automatically, because retries multiply live credentials and train users to click whichever message arrived last. A non-consuming landing page followed by an explicit POST changes the state transition instead of hiding the race.

The compact Python example below shows the shared server-side invariant. It deliberately leaves transport and HTML rendering behind interfaces; neither belongs in the challenge transaction. In production, `consume` must be one atomic database operation so two concurrent requests cannot both win.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


class ChallengeStore(Protocol):
    def replace_active(self, challenge: "Challenge") -> None: ...

    def consume_if_valid(
        self, *, challenge_id: str, verifier: bytes, now: datetime
    ) -> bool: ...


@dataclass(frozen=True)
class Challenge:
    challenge_id: str
    account_id: str
    purpose: str
    channel: str
    verifier: bytes
    expires_at: datetime
    template_version: str


def complete_email_challenge(
    store: ChallengeStore, challenge_id: str, verifier: bytes
) -> bool:
    return store.consume_if_valid(
        challenge_id=challenge_id,
        verifier=verifier,
        now=datetime.now(timezone.utc),
    )
```

The store operation should verify the secret in constant-time code supplied by a reviewed cryptographic library, confirm the expected purpose, reject expired or consumed rows, increment failed attempts where applicable, and mark one successful row consumed in the same transaction. Postgres can enforce the single-winner transition with a conditional update and a uniqueness rule for the active challenge model. Do not log the `verifier` argument.

## Delivery evidence has limits

Email acceptance is not inbox placement, and inbox placement is not human action. Build metrics around transitions the system can observe: challenge requested, message submitted, transport accepted or rejected, bounce classified, user completed, challenge expired, and recovery escalated. Correlate those events with an opaque challenge identifier rather than the secret or raw address. Separate security telemetry from the longer-lived compliance-notice ledger.

A bounce is evidence. It is not proof of every other state.

Deliverability basics still apply to authentication mail. Authenticate the sending domain with SPF and DKIM, publish and monitor a DMARC policy, keep From-domain alignment intentional, and process bounces. These controls establish domain-level handling and reporting; they do not guarantee inbox placement. RFC 7208, RFC 6376, and RFC 7489 define the relevant mechanisms and their boundaries.

Instrument latency as a distribution, split by route, domain class, template version, and region only where the privacy model permits it. A global average can hide a mailbox-specific delay that expires most challenges. Watch resend rate, completion before expiry, invalid-attempt rate, link landing-to-POST conversion, and fallback escalation. Do not use opens as authentication proof. Remote-image blocking, proxying, and privacy features make that signal unsuitable for deciding whether a secret was seen.

Abuse controls need equally careful semantics. Rate limiting solely by IP punishes offices and carrier gateways; limiting solely by account allows destination flooding across many accounts. Layer several keys, return non-enumerating responses, and alert on changes rather than one universal threshold. Keep the user-facing response stable enough that an attacker cannot discover registered marketplace accounts from timing or wording.

For compliance messages, test the exact approved render in plain text and HTML, including long marketplace names, missing optional fields, right-to-left locales if supported, and URLs after tracking is disabled or constrained by policy. For auth messages, test duplicate submissions, expired challenges, two simultaneous consumes, resend races, scanner GETs, and a delayed SMS arriving after email fallback starts. The winner should close sibling challenges according to the documented assurance policy.

## Roll out the ownership boundary in four moves

First, inventory every SMS and email template that participates in login, recovery, and compliance notice delivery. Assign a purpose and owner to each; identical prose does not make two purposes identical. Freeze ad hoc dashboard edits while canonical versions are created.

Second, introduce the versioned message contract and shadow-render it without sending. Compare normalized subjects and bodies against the current approved output, but review escaping and missing variables manually. Snapshot tests catch drift; they cannot approve legal meaning.

Third, deploy the challenge state machine behind a small cohort. Start with audit-only evaluation of fallback eligibility, then enable email recovery for accounts and actions within the accepted assurance tier. Track route transitions and terminal outcomes. Never infer success from a transport event.

Finally, migrate compliance notices to their own ledger and retention policy, verify that a past render can be reproduced from approved artifacts, and rehearse transport replacement. The migration is complete when changing a delivery adapter does not change template semantics, challenge rules, or the meaning of audit evidence. **Transport is replaceable; the security and compliance contract is not.**

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://mustache.github.io/mustache.5.html
- https://www.rfc-editor.org/rfc/rfc7208
- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc9110
- https://owasp.org/www-project-cheat-sheets/cheatsheets/Authentication_Cheat_Sheet.html
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
