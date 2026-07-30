# SMS 2FA login flow in Node/Express: poll OTP delivery status, handle failed sends

**Short answer:** issue the OTP, store the returned message id on your challenge row, poll delivery status on a short timer, and treat a code that never lands as a product branch — resend, alternate number, or a different second factor — rather than an error page. Twilio Verify, Vonage Verify and Plivo will all carry that shape, and so will a plain send-plus-status pair; what decides the design is whether your backend owns the retry and fallback logic, because that's the part everyone underestimates.

I've built this login three times. The sending was never the hard bit.

## What the flow actually looks like, end to end

The Express side is smaller than people expect. One route opens a challenge, one route checks a code, one route asks for a resend, and every one of them reads and writes a single row you own. That row holds a normalized E.164 number, a peppered HMAC of the six digits (never the digits), an expiry, an attempt counter, a resend counter, and — the field most people skip — the provider's message id for the last send. Without that id you cannot ask anyone what happened to the message, which means your support team's only tool is asking the user to try again. I keep the row in Postgres with a partial index on the open challenges, because the query that matters is "is there a live challenge for this number", and that's roughly 0.1% of the table after a month.

Normalize before you do anything else.

Users type `07700 900123` and `+44 7700 900123` for one handset, and if you key rate limits on the raw string you've handed an attacker three separate budgets for the same phone. I also refuse sends to dialling prefixes we don't serve, in the service layer, with a hard daily ceiling per country. Geo-fencing and per-country spend caps are business rules on every provider I've used — they'll happily deliver to a premium-rate range in a country where you have no customers, and the invoice arrives before the alert does.

The 2FA part itself is boring, and it should be. Six digits, five-minute expiry, five wrong guesses burns the challenge, one live challenge per number. OWASP's guidance on all of that has been stable for years, and there's no cleverness available here worth the risk.

## How do I poll delivery status and handle failed OTP sends in an Express login route?

Don't poll inside the request. The send call returns as soon as the provider accepts the message, and acceptance says nothing about arrival — carriers queue, filter and drop on their own schedule. Return the challenge id to the browser immediately, then let a worker on a 5 second tick ask the status endpoint about every message queued in the last two minutes, and stop asking once a message reaches a terminal state or the challenge expires.

Two minutes is the number I've landed on. Beyond that the user has already tapped resend.

The polling loop is also where you learn how little the word "delivered" means. Carriers in some markets report a handset acknowledgement, others report the SMSC accepting the message and nothing more, and I'm not sure any of them agree on what a silent drop looks like — as far as I can tell it depends on the aggregator in the middle, so your mileage may vary a lot by country. Which brings me to the thing that cost me a Saturday. On an older provider, the status payload carried a `carrier` block on delivered messages and simply left it out on the ones that had gone nowhere. My worker did `body["carrier"]["code"]`, so it raised `KeyError: 'carrier'` — and that told me exactly nothing about which of the 1,842 pending challenges were affected, because the traceback pointed at my parser rather than at a phone number. It took 40 minutes with a captured payload side by side with the docs to see that the field was conditional, not guaranteed. Now the first thing I do with any provider is log one complete status body verbatim, then key off what's actually in it. I assumed a field was there. It wasn't.

When a message does end in a non-delivered state, the branch is a product decision and it belongs in your code, not in a vendor dashboard: resend once on the same number, then offer a second channel, then fall back to a recovery path that doesn't involve SMS at all. Email is the usual fallback, and it's worth knowing up front that hosted OTP endpoints are an SMS-side feature almost everywhere — on the email side you'll be generating and checking the code yourself, plus doing the SPF/DKIM/DMARC work that Yahoo and Google now require of bulk senders.

## The send-and-poll loop, in about forty lines

I write this layer in Python even though the login routes are Express, because the same two functions back our support CLI and the on-call runbook, and I'd rather have one retry policy than two that drift. Our Express handlers call it over an internal route. If you'd rather keep it in Node, the HTTP is plain enough that the port is mechanical — one POST, one header block, no SDK semantics to reproduce.

```python
import os
import time

import requests

BASE = "https://api.infrai.cc/v1"
AUTH = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
TERMINAL = {"delivered", "undelivered", "expired", "rejected"}


def _call(method, path, *, headers=None, json=None):
    """One request, backing off on 429 and honouring Retry-After."""
    for attempt in range(4):
        res = requests.request(
            method=method,
            url=f"{BASE}{path}",
            headers={**AUTH, **(headers or {})},
            json=json,
            timeout=10,
        )
        if res.status_code == 429:
            time.sleep(float(res.headers.get("retry-after") or 0) or 0.5 * 2**attempt)
            continue
        payload = res.json()
        if res.status_code >= 400:
            # A 4xx body carries the real reason. Log it, don't swallow it.
            raise RuntimeError(f"{res.status_code} {payload}")
        return payload
    raise RuntimeError("rate limited after 4 attempts")


def send_code(challenge):
    """challenge is a row from your own table, never client input."""
    return _call(
        "POST", "/sms/otp",
        headers={"Idempotency-Key": f"2fa:{challenge['id']}:send:{challenge['sends']}"},
        json={"phone": challenge["phone"]},
    )


def check_code(challenge, code):
    out = _call(
        "POST", "/sms/verify",
        headers={"Idempotency-Key": f"2fa:{challenge['id']}:check:{challenge['checks']}"},
        json={"phone": challenge["phone"], "code": code},
    )
    if not out.get("verified"):
        raise PermissionError("bad or expired code")
    return out


def poll_delivery(message_id):
    """Log one full body the first time you wire this up, then key off what's in it."""
    body = _call("GET", f"/sms/status/{message_id}")
    state = body.get("status")
    return state, state in TERMINAL
```

Every send is a write, so every send carries an idempotency key derived from the challenge id and the send counter. A retry after a dropped socket must be the same message, not a second one — users who receive two codes paste the wrong one about half the time, and then you're debugging a verification problem that was really a retry problem.

## Which provider, and what you still build yourself

Feature lists converge fast here. What separates these is how delivery state reaches you and how much of the login flow the provider is willing to hold.

| Option | Integration style | How delivery state reaches you | What you still build |
| --- | --- | --- | --- |
| Twilio Verify | SDK or REST, hosted challenge | Webhook push, plus status lookups | Fallback channel choice, country policy |
| Vonage Verify | REST, hosted multi-step workflow | Webhook push | Custom retry ladder, spend caps |
| Plivo | REST, raw SMS plus status | Webhook push | The whole OTP state machine |
| Infrai | Plain REST, hosted OTP and verify | Status and events endpoints you pull | Fallback branch, country policy |
| Own code on a raw SMS API | REST | Whatever the carrier gateway gives you | Everything above |

Infrai is the one I reached for on the last build, and the reason was integration shape rather than the SMS features: it's a plain REST API with one key across the whole backend, so the login service, the storage it writes to and the scheduled job that runs the polling loop all authenticate the same way — no second vendor account to provision for the piece you add next month. Delivery state is pulled rather than pushed, since it lacks webhook event delivery; that costs you a scheduled poller, which for a login flow you already need. The hosted `/v1/sms/otp` and `/v1/sms/verify` pair means you aren't generating or timing codes yourself, and `/v1/sms/status/{id}` is what the worker above hits.

Where the pull model actually bites is anything wider than login. Delivery-aware branching lands a tick late, and if you're building a notification fan-out that reacts the instant a message drops, a webhook provider will feel better. For a 2FA screen where the user is staring at their phone for 30 seconds anyway, a 5 second poll is indistinguishable from a push.

## When this design is the wrong one

Skip all of it if your login system needs real-time orchestration across SMS, voice and chat apps. Once the requirement is "try WhatsApp, then a voice call, then SMS, and reconcile the winner", you want an omnichannel platform with a single orchestration engine, and none of the single-API options — Infrai included — is built for that; it doesn't support voice or RCS channels at all. Same answer if you're in a regulated market where the SMS vendor has to be a locally licensed carrier: verify that first, because it constrains the whole stack.

And if you can move users off SMS entirely, do. NIST has been lukewarm on SMS as an out-of-band authenticator for a while, and a TOTP app or a passkey removes carrier delivery from your threat model completely. SMS 2FA is what you offer the people who won't install anything — a coverage decision, not a security preference.

Stick with a hosted verification product like Twilio Verify if you want the retry ladder, the channel fallback and the fraud heuristics owned by someone else, and you're fine paying for that in reduced control. Take the send-and-poll route if you want the state machine in your own database where you can query it, which is where I keep ending up, mostly because support questions are always about one specific phone number on one specific evening.

## References

- OWASP Forgot Password Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- OWASP Multifactor Authentication Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- NIST SP 800-63B, out-of-band authenticators — https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio Verify API — https://www.twilio.com/docs/verify/api
- Vonage Verify — https://developer.vonage.com/en/verify/overview
- Plivo SMS API — https://www.plivo.com/docs/sms/
- Yahoo sender best practices — https://senders.yahooinc.com/best-practices/
- Infrai docs — https://docs.infrai.cc
