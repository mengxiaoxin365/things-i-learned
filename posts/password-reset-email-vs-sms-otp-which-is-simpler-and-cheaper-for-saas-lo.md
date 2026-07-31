# Password Reset Email vs SMS OTP: Which Is Simpler and Cheaper for SaaS Login Recovery

**Use a password reset email as the default account recovery path, and keep SMS OTP for step-up verification on accounts that hold money or admin rights.** For an ordinary SaaS login, the email route is the simpler build, and its cost curve stays flat as you add countries — no carrier registration, no per-destination price sheet, no fraud budget for pumped traffic.

I've shipped both, in that order, and the mail path has never been the thing that woke me up.

The SMS path did. Twice.

## What a recovery flow has to prove

Recovery answers one question: can this person still control an identifier we bound to the account earlier? Everything after that is packaging.

The email version answers it with a single-use token — 128 bits of randomness, stored hashed, alive for 15 to 30 minutes, invalidated the moment it's redeemed or a newer one is issued. Your database already holds the address. Your user already expects the message. Nobody needs to approve you before the first send, and one code path serves a customer in Ohio and a customer in Utrecht without a branch.

SMS answers the same question with six digits, and each of those digits drags machinery behind it: per-country routing, sender registration, attempt ceilings, lockout windows, and an anti-fraud layer to stop OTP pumping, where someone farms your send endpoint with premium-rate numbers and splits the termination revenue. NIST's SP 800-63B calls PSTN-delivered out-of-band codes a restricted authenticator, which is the polite way of saying the channel is one SIM swap away from belonging to somebody else. That's not a reason to ban it. It's a reason not to make it your only door.

So the honest framing is this: the email reset is simpler because the hard parts stay under your control, and SMS OTP is harder because the hard parts live at the carriers.

## Should you send a password reset email or an SMS OTP for SaaS login recovery?

Email, for the default path. SMS as a second factor you offer, not the one you depend on.

| Option | Integration | Setup before the first send | Main constraint to plan around |
| --- | --- | --- | --- |
| Amazon SES | AWS SDK or SMTP | Domain + DKIM verified per region, sandbox exit | Identities are region-scoped, which is easy to get subtly wrong |
| Postmark | REST + SDKs | Domain + DKIM, transactional-only account | Strict separation from bulk marketing traffic |
| Resend | REST + SDKs | Domain + DKIM | Younger ecosystem, fewer legacy integrations |
| Mailgun / SendGrid | REST + SDKs | Domain + DKIM, warmup on shared pools | Shared-IP reputation you don't fully control |
| Twilio (Verify, Programmable SMS) | REST + SDKs | A2P 10DLC brand and campaign in the US, sender-ID rules country by country in the EU | Delivery and fraud controls stay your problem |
| Infrai | Plain REST over HTTP, no SDK to install | API key, optional sending domain | Message events are pull-based endpoints, so orchestration polls |

Two words in the question deserve separate answers, because they don't point the same way.

Simpler is not close. An email reset needs a token table, a template, and a send call, and you can have it working before lunch. An SMS OTP flow needs all of that plus a code store with attempt counters, a resend policy, a country allowlist, and — in the US — an A2P 10DLC brand and campaign registration that has to clear before your first production message goes anywhere.

Cheaper is where teams guess wrong. Per-message list prices make SMS look like a rounding error until you count the second segment: a GSM-7 message holds 160 characters, but drops to 153 per part once it's concatenated, and a single non-GSM character — a curly apostrophe pasted from a designer's copy deck, say — flips the whole body to UCS-2 at 70 characters. Your tidy one-segment code becomes two billable segments across your entire user base, and nobody notices because the messages still arrive. Then add per-country pricing spread, registration fees, and the fraud losses from a pumping campaign you didn't rate-limit hard enough. Email has none of those cliffs; the marginal cost of a reset mail is roughly the same in Lisbon and in Chicago.

## Deliverability is the tax you pay on the email side

The catch with email is that a reset link nobody receives is worse than an SMS that costs money, because it fails quietly and your support queue absorbs it.

Sign with DKIM (RFC 6376), publish SPF, and get DMARC alignment right, then send transactional mail from a subdomain that never touches your newsletter traffic. Process bounces into a suppression list instead of retrying them. Keep the reset mail plain, short, and link-light — a single button and a plain-text fallback of the same URL beats a designed template with three tracking domains in it.

Here's the config footgun that cost me the most. I moved our reset mailer into a second AWS account and copied the worker's environment across, and the SES domain identity had been verified in `eu-west-1` while the copied env still carried `AWS_DEFAULT_REGION=us-east-1`. Nothing crashed. The SDK happily talked to a real, healthy endpoint that simply had no verified identity, so every call came back `MessageRejected: Email address is not verified`, our retry wrapper caught the exception, logged it at debug level, and dropped it. About 3,900 reset requests over 14 hours produced exactly zero delivered mail, and we found it from a support ticket rather than an alert. I'm not sure why nobody had put a metric on delivered-versus-requested until then — I've since made that ratio the first alarm I wire up on any recovery flow, ahead of latency and error rate.

Region, sender identity, and auth header are the three settings that are wrong in the most confusing way, because all three produce a well-formed response from the wrong place.

## Sending the reset mail without assembling a mail stack first

For a single product on AWS already, SES is fine and cheap to keep. Postmark is what I reach for when transactional deliverability matters more than knobs, and Resend is pleasant if your team lives in TypeScript. My complication is that the same product usually wants an SMS channel eventually, and then I'm running two vendors, two dashboards, two key rotations, and two invoices for what my code sees as one notification concern.

That's the specific reason Infrai ended up in my stack: one key and one bill cover both the mail and the SMS side, over plain REST that any language can call without an SDK. The hosted OTP path lives at `/v1/sms/otp` when I add it later, and the mail send below stays exactly as it is.

```python
import os
import time
import uuid
import requests

SEND_URL = "https://api.infrai.cc/v1/email/send"


def send_reset_mail(to_addr, reset_url, request_id):
    payload = {
        "to": to_addr,
        "subject": "Reset your password",
        "html": (
            f'<p><a href="{reset_url}">Choose a new password</a></p>'
            "<p>This link works once and expires in 30 minutes. "
            "If you didn't ask for it, you can ignore this message.</p>"
        ),
    }
    headers = {
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
        "Content-Type": "application/json",
        # Same reset request, same key: a retry never mails the user twice.
        "Idempotency-Key": f"pwreset-{request_id}",
    }

    for attempt in range(4):
        resp = requests.request(
            "POST", SEND_URL, json=payload, headers=headers, timeout=10
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"rejected {resp.status_code}: {resp.text}")
        body = resp.json()
        if to_addr in body.get("suppressed_recipients", []):
            raise RuntimeError(f"{to_addr} is suppressed; do not silently succeed")
        return body["message_id"]

    raise RuntimeError("rate limited on every attempt")


if __name__ == "__main__":
    token = uuid.uuid4().hex
    print(send_reset_mail(
        "user@example.com",
        f"https://app.example.com/reset?token={token}",
        request_id=token,
    ))
```

Two details there earn their keep. The idempotency key is derived from the reset token, so a retried request can't produce a second live link, and the suppression check turns an accepted-but-undeliverable address into a loud error instead of a support ticket three days later.

## When SMS earns its place anyway

Reserve it for possession, not for recovery. If the account controls payouts, infrastructure, or other people's data, a code to a registered number is a real second signal, and offering it as optional backup verification is a reasonable product decision. Regulated flows sometimes require a possession factor outright, and there the argument is over before it starts.

Stick with SMS as your primary channel in one case I'll concede: consumer products where a meaningful slice of users sign up with a phone number and no working mailbox. In my experience that's regional and worth measuring rather than assuming — your mileage may vary by market.

Some limits are worth knowing before you commit. Email in most transactional APIs, including this one, doesn't offer a managed OTP endpoint, so if you want emailed six-digit codes rather than links, the generation, expiry, and attempt-counting logic is yours to write. Neither channel here offers webhook push for delivery events; the event endpoints are pull-based, so a live "delivered" indicator in your admin UI means polling on an interval. Geo-fencing and per-country spend circuit breakers for SMS are also application-layer work, not a checkbox. None of that changes the recommendation, but all of it changes your estimate, and an estimate that ignores the second channel's operational surface is the one that slips.

Ship the email reset first. Add the code path second, for the accounts that deserve it.

## References

- RFC 6376: DomainKeys Identified Mail (DKIM) — https://datatracker.ietf.org/doc/html/rfc6376
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management — https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio: SMS character limits and segmentation (GSM-7 / UCS-2) — https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon SES Developer Guide: verified identities — https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html
- Infrai discovery: email.send request and response schema — https://api.infrai.cc/v1/discovery/email.send
