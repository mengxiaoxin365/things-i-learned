# SMS OTP API Constraints: 2FA Login and Revocation

Short answer: an SMS API belongs behind an application-owned, revocable OTP challenge, because resend and cancel are authentication-state operations rather than message-delivery features. For a US and EU app built in Node.js, the useful evaluation test is whether a transport can sit behind that contract without controlling code validity, retry policy, or verification state.

That changes the buying question. A provider can accept an SMS, but the login service must decide whether the code is still valid when it arrives. Keep those responsibilities separate and a delayed first message, a fast second tap, or a canceled login has one deterministic result. The least complex version is a small state machine plus a narrow delivery adapter.

## Start with the constraint, not the SMS API

An OTP has two timelines. The authentication timeline starts when the challenge is issued and ends when it is verified, canceled, expired, or locked. The delivery timeline starts when a send is requested and may continue after the authentication timeline has ended. Treating those as one timeline creates the dangerous assumption that canceling a request can somehow make an already disclosed secret valid or invalid at the handset. Cancel must therefore revoke the challenge in the login database. It should be idempotent: canceling an already canceled challenge returns the same terminal state, while canceling a verified challenge must not undo the completed authentication. Verification must read the current state before comparing the submitted value. The SMS transport doesn't get to make that decision. Resend has a similarly narrow meaning: request another delivery for the same active challenge, subject to a cooldown and a cap chosen for the application's threat model. The login service should not assume message order. It must also decide explicitly whether resend reuses the current code or replaces it. Reuse reduces the chance that an older delivery contains a superseded code; replacement limits how long one disclosed value remains useful. Neither policy is universally best. The important part is that the state transition and its consequences are defined in one place.

This is where deliverability work gets unglamorous. A UI label such as `sent` should describe an accepted application command, not promise that a person has read a message. I've learned to distrust any login design that turns transport progress into authentication truth — delivery gaps and retries make that coupling visible at the worst time.

No guesswork.

## How should 2FA login systems revoke a delayed SMS OTP?

Give the app builder a small command contract: `issue`, `resend`, `cancel`, and `verify`. Each command takes a challenge identifier and an idempotency key where a network retry could duplicate work. The persistent record needs a status, an expiry, an attempt counter, a send counter, and timestamps used by the chosen policy. Store a keyed digest of the code rather than the plaintext value, and never place the code in logs.

The example is Python because all code in this note uses one language, but the boundary is intentionally portable to a Node.js handler or generated app action. The strings such as `OTP-409` are application result codes, not claims about a provider response. They give the UI stable branches without exposing internal detail.

```python
from dataclasses import dataclass
from enum import Enum
import hashlib
import hmac
import time


class Status(str, Enum):
    ACTIVE = "active"
    VERIFIED = "verified"
    CANCELED = "canceled"
    EXPIRED = "expired"
    LOCKED = "locked"


@dataclass
class Challenge:
    challenge_id: str
    code_digest: str
    expires_at: float
    status: Status = Status.ACTIVE
    attempts: int = 0
    sends: int = 1
    last_send_at: float = 0.0


def digest(code: str, secret: bytes) -> str:
    return hmac.new(secret, code.encode(), hashlib.sha256).hexdigest()


def resend(challenge, now, cooldown, max_sends, deliver):
    if challenge.status != Status.ACTIVE or now >= challenge.expires_at:
        return {"ok": False, "code": "OTP-409"}
    if challenge.sends >= max_sends:
        return {"ok": False, "code": "OTP-429"}
    if now - challenge.last_send_at < cooldown:
        return {"ok": False, "code": "OTP-425"}

    next_send = challenge.sends + 1
    deliver(idempotency_key=f"{challenge.challenge_id}:{next_send}")
    challenge.sends = next_send
    challenge.last_send_at = now
    return {"ok": True, "code": "OTP-SENT"}


def cancel(challenge):
    if challenge.status == Status.ACTIVE:
        challenge.status = Status.CANCELED
    return {"ok": True, "status": challenge.status.value}


def verify(challenge, submitted_code, secret, now, max_attempts):
    if challenge.status != Status.ACTIVE:
        return {"ok": False, "code": "OTP-409"}
    if now >= challenge.expires_at:
        challenge.status = Status.EXPIRED
        return {"ok": False, "code": "OTP-410"}

    challenge.attempts += 1
    matches = hmac.compare_digest(
        challenge.code_digest,
        digest(submitted_code, secret),
    )
    if matches:
        challenge.status = Status.VERIFIED
        return {"ok": True, "code": "OTP-VERIFIED"}
    if challenge.attempts >= max_attempts:
        challenge.status = Status.LOCKED
        return {"ok": False, "code": "OTP-LOCKED"}
    return {"ok": False, "code": "OTP-INVALID"}
```

The example leaves durations and limits as inputs on purpose. I'm not sure one fixed cooldown can be justified across every risk profile, carrier path, and user population; production evidence should resolve it. What should remain fixed is the invariant: only an active, unexpired challenge can be resent or verified, and cancel moves it out of that state exactly once.

For an AI app builder or tool-using agent, expose those commands as separate tools with precise input schemas rather than one ambiguous `manage_otp` action. A resend command has a side effect, while verification consumes a user-provided secret. Keep the agent away from raw transport credentials and let the backend enforce every transition.

## Compare control boundaries before feature lists

Once the lifecycle is stable, compare candidate APIs by the control boundary they impose. A hosted verification workflow can reduce the authentication code your team owns, while a generic messaging transport leaves the full challenge lifecycle in your database. A second transport can improve routing options, but it also adds sender configuration, event normalization, and operational paths that have to be tested. Those are architectural trade-offs, not a ranking.

| Decision axis | Hosted verification workflow | Messaging transport behind your state machine | Multiple transports behind one adapter |
| --- | --- | --- | --- |
| Resend and cancel semantics | Defined partly by the external workflow | Defined by the login service | Defined by the login service |
| Portability | Coupled to workflow concepts | Coupled to a narrow send interface | Highest adapter burden |
| Operational ownership | Less application logic | More state and policy to test | More routing and event normalization |
| Best fit | Small team accepting fixed semantics | Team needing explicit revocation and policy control | Team with evidence that one route is insufficient |

Ask candidates for test credentials or a non-production environment, documented idempotency behavior, stable message identifiers, delivery-event authentication, regional sender configuration, data-handling terms, and an export path for operational records. Then run the same acceptance suite through every adapter. One test deserves more than a checkbox: hold a delivery call open, submit cancel, release the delivery call, and then submit the received code. The final verification must fail because the database state is terminal, even though the transport completed later. Repeat the sequence with two identical resend commands carrying the same idempotency key and confirm that application state records one logical send. Then invert the completion order. This exercise does not predict carrier behavior; it proves that transport timing cannot revive a secret. The rest of the suite should cover expiry at the boundary, repeated wrong codes, an out-of-order delivery event, and a transport timeout whose final delivery status is unknown.

The catch is ownership. A custom state machine is not suitable when the team cannot maintain an authentication service, monitor abuse, or test terminal-state races; stick with a managed verification workflow whose documented semantics meet the product requirement. Multiple transports are also a poor default. Add one only when measured delivery behavior or a concrete regional constraint justifies the extra routing surface.

Email fallback deserves a separate reputation boundary and policy. If an OTP is sent by email, DKIM provides a way for a signing domain to take responsibility for a message by attaching a cryptographic signature that a verifier can validate. RFC 6376 does not make the message an authentication factor by itself, and it does not repair the OTP lifecycle. It addresses a different layer, which is exactly why delivery authentication and challenge verification should stay separate.

## Roll out the contract in one narrow slice

Start with one login path and put the state transition tests ahead of the transport adapter tests. Record command outcomes by stable reason code, but redact phone numbers and codes. Useful signals include issue-to-verify completion, resend attempts per challenge, terminal-state verification attempts, adapter acceptance latency, and delivery events that cannot be correlated to a known send. Choose alert thresholds from the application's baseline instead of copying somebody else's percentages.

During migration, let challenges already issued under the old verifier complete under that verifier. Route newly issued challenges to exactly one implementation and store the implementation identifier with the record; this avoids guessing which verifier owns a submitted code. Rollback then means stopping new issuance on the new path, not trying to translate live secrets between incompatible stores.

The final selection can now be pleasantly dull: choose the API whose documented contract passes the lifecycle suite, meets the required regional and data-handling constraints, and fits the team's operational capacity. Resend and cancel remain application guarantees. The transport remains replaceable.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)

## Further reading

- [Anthropic: Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
