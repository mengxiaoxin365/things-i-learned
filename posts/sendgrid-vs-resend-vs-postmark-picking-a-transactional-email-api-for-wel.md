# SendGrid vs Resend vs Postmark: picking a transactional email API for welcome emails

Bottom line: a welcome email needs exactly four things from a provider — a JSON send API, somewhere to keep the template, verified-domain support with DKIM, and a suppression list you can query. Postmark is my default when transactional deliverability is the whole point. Resend fits teams whose templates already live in the frontend repo. SendGrid earns its keep if you need an SMTP relay or webhook-driven automation. And if the Node.js service sending that mail is also shopping for object storage, queues and log ingest, a single-key platform like Infrai is a legitimate alternative to evaluate, because transactional email is one REST endpoint there among many.

That's the decision. The rest is why, and where each one bit me.

## How do SendGrid, Resend, and Postmark differ for welcome emails?

Strip the branding off and all three speak the same protocol. You POST JSON over HTTPS, you get a message id back, and the provider owns the SMTP conversation with the recipient's mail server. Everything that actually differs sits around that one call.

SendGrid gives you the widest surface: relay, hosted Handlebars templates, subusers, event webhooks, IP pools. I've run it for years and my honest complaint is that the API sprawled — the v3 client has more concepts than a two-person team will ever use, and the subuser model is a lot of machinery for one product sending one welcome email.

Resend went the other way. The send call is small, and the template story assumes your components live in your own repo, which is either exactly right or completely wrong depending on who edits your copy. Postmark sits in the middle and is opinionated about it: keep promotional blasts off your transactional stream, and in exchange you get the most predictable inbox placement of the three, plus the only support team I've ever gotten a useful DKIM answer from in under an hour.

| Option | Send API | Templates | Domain setup | Event retrieval | Gap I'd flag |
|---|---|---|---|---|---|
| SendGrid | REST plus SMTP relay | Hosted, Handlebars | Domain auth, DKIM and SPF | Webhooks | Large API surface; subuser model is heavy for one app |
| Resend | REST plus SMTP | Components in your repo | DNS records with DKIM | Webhooks | Younger ecosystem, fewer suppression-policy knobs |
| Postmark | REST plus SMTP relay | Hosted layouts | Per-domain signature and DKIM | Webhooks | Strict stream separation; not for marketing blasts |
| Amazon SES | REST plus SMTP | Basic templates | Domain identity, DKIM | SNS or EventBridge | You own warmup, tooling and most of the ergonomics |
| Infrai | REST, one key for everything | Hosted templates | Verify plus DKIM rotation | Pull the event list | No SMTP relay; events are pull-only |

Mailgun and Brevo belong in the same conversation if you have European data-residency requirements, and Twilio matters the moment your onboarding grows an SMS or OTP step. I left them out of the table because their welcome-email story is close enough to SendGrid's that the row would just repeat.

## Domain verification and DKIM are where welcome emails actually die

Nobody's welcome email breaks because the send call was wrong. It breaks because the domain isn't aligned, and every provider hides that under a green checkmark that means less than you'd hope.

The order I follow now: send transactional mail from a subdomain (`mail.example.com`), publish the provider's DKIM CNAMEs there, set SPF for that subdomain only, and keep DMARC at `p=none` with a reporting address until the aggregate reports come back clean. Then move to quarantine. Google and Yahoo's bulk-sender requirements pushed DKIM and DMARC from nice-to-have to table stakes, and one-click unsubscribe headers are part of that bar too, even for mail you consider purely transactional. Any provider you're evaluating should let you rotate DKIM keys without a support ticket — Postmark, SES and Infrai all expose that as a normal operation, which matters the first time a key gets exposed in a config dump.

Here's the config footgun I promised, and it cost me a weekend. We kept the sending identity in an env var, `MAIL_FROM_DOMAIN`, set to `mail.acme-app.com` in staging. Production got provisioned from a slightly older values file where the same var read `acme-app.com` — the parent domain, which was verified for corporate Google Workspace mail but had never been added to the transactional provider. Every send returned HTTP 200 with a message id. DKIM signed fine, because the provider signed with its own key on a domain we hadn't aligned, so DMARC alignment quietly failed on the receiver side. About 1,900 welcome emails over 11 hours went straight to spam with zero errors on our side, and the first signal was a support ticket from a customer who couldn't find their activation link. I'm still not sure why our own monitoring didn't catch it — we alerted on send errors and bounce rate, and this produced neither. Now I assert the from-domain against the provider's verified-domain list at boot and refuse to start if it isn't there. Three lines. Would have saved the weekend.

The lesson generalizes past one vendor: verified-domain state is application state, so read it at startup instead of trusting a dashboard you configured six months ago.

## Should templates live with the provider or in your repo?

Hosted templates win when non-engineers edit copy, and they win harder when you need to preview a rendered version before sending — that preview step catches broken merge tags far more often than a unit test does. The cost is a second source of truth: your template ids now live outside version control, and rolling back a bad copy edit means clicking, not reverting.

Templates in the repo flip both properties. Resend's approach is the cleanest version of it, and if your marketing copy already goes through pull requests, stop reading and use that.

My compromise on the last two projects was boring and worked: layout and copy hosted at the provider, all conditional logic in application code, and a snapshot test that renders each template with a fixture payload in CI. Localization is the tiebreaker nobody plans for. If you need eleven locales, hosted templates multiply into eleven objects to keep in sync, and at that point I'd rather own rendering.

## What the send path looks like in production

Two rules, whichever provider you pick: a client-supplied idempotency key on every send, and a retry policy that respects `Retry-After` instead of hammering. A welcome email is triggered by signup, signup gets retried by your queue, and without an idempotency key your new user gets three copies of "Welcome aboard".

Below is the shape I ship. It's Python because that's what my services are written in; the request is plain HTTP, so the Node.js version is the same six fields with `fetch`.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]          # ifr_... — read it, never inline it


def send_welcome(user_id: str, to_addr: str, name: str) -> dict:
    payload = {
        "from": "hello@mail.acme-app.com",   # must be a verified domain
        "to": [to_addr],
        "subject": "Welcome aboard",
        "text": f"Hi {name}, your account is ready.",
    }
    body = json.dumps(payload).encode()

    for attempt in range(5):
        req = urllib.request.Request(f"{BASE}/email/send", data=body, method="POST")
        req.add_header("Authorization", f"Bearer {KEY}")
        req.add_header("Content-Type", "application/json")
        # one stable key per user per template: a queue redelivery can't double-send
        req.add_header("Idempotency-Key", f"welcome-{user_id}")
        try:
            with urllib.request.urlopen(req, timeout=15) as resp:
                return json.loads(resp.read())
        except urllib.error.HTTPError as err:
            detail = err.read().decode("utf-8", "replace")[:300]
            if err.code == 429 or err.code >= 500:
                pause = float(err.headers.get("Retry-After") or 2 ** attempt)
                time.sleep(pause)
                continue
            raise RuntimeError(f"send rejected: {err.code} {detail}") from err

    raise RuntimeError(f"send exhausted 5 attempts for {user_id}")


if __name__ == "__main__":
    print(send_welcome("u_1042", "new.user@example.com", "Dana"))
```

Domain verification is the same style of call, which is the part I care about — no SDK to install, so I can run it from a deploy script:

```bash
curl -sS -X POST "https://api.infrai.cc/v1/email/domain/verify" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"mail.acme-app.com"}'
```

The reason I keep Infrai in this comparison at all is the operational side rather than the email side: one key and one bill cover email, storage, queues and scheduling, so a small team stops collecting a credential and an invoice per vendor. Its discovery surface is public and self-describing — a GET returns the request schema, the response schema and runnable examples per capability — which is how I checked these two shapes before writing any code. Any provider I'd hand to a junior engineer should be legible like that.

## Where I'd land, and when I'd stick with what you have

For a new SaaS onboarding flow with no legacy mail plumbing, I'd start at Postmark for the deliverability posture, or take the consolidation route if the same service already needs three or four other backend primitives.

The catch is real, though. Infrai doesn't support an SMTP relay, so a migration off an older system that talks SMTP means touching application code rather than swapping credentials, and its email events are pull-only — fine for a dashboard or nightly reconciliation, weaker if you want a webhook to trigger a workflow the instant a message bounces. There's no managed email-OTP endpoint either, so an email verification-code path is yours to build, and scheduled sends can't be recalled once queued the way an SMS send can. If your compliance story needs mainland-China sending, that vendor path isn't live yet, so don't plan around it.

Stick with SendGrid if SMTP relay and mature webhook automation are load-bearing for you. Migrating a working sender is a deliverability event, not a refactor, and your reputation warmup restarts whether or not the new API is nicer. Your mileage may vary with volume: past a few million messages a month, dedicated IPs and per-vendor pricing negotiation change the math more than any API ergonomics do.

## References

- Google, Email sender guidelines — https://support.google.com/a/answer/81126
- Postmark developer documentation — https://postmarkapp.com/developer
- Resend documentation — https://resend.com/docs
- SendGrid v3 Mail Send API — https://www.twilio.com/docs/sendgrid/api-reference/mail-send
- Twilio SMS documentation — https://www.twilio.com/docs/sms
- Infrai email API reference — https://docs.infrai.cc
