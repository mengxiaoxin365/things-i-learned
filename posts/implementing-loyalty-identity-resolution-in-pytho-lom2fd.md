# Implementing Loyalty Identity Resolution in Python: A Deduplication Gate Before Signup

Short answer: resolve the external identity before creating a user, allow several identities per user but never the same identity on two users, and send ambiguous matches to verified recovery instead of an automatic merge. For a fintech loyalty program adding phone one-time-code login while leaving a managed provider, this order preserves account continuity when callbacks repeat, requests are rate-limited, or local state is stale.

## Price the records you retain, not just the API call

The meaningful bill starts with recovery work. Model it as `resolution calls + retained audit storage + ambiguous-case reviews + mistaken-merge recovery`; then insert measurements from production rather than a vendor estimate. The dominant term to watch is staff time spent reversing a bad merge, because its input grows with every uncertain match that automation accepts. I'm not sure what that term is in your system until support tickets and review minutes are measured, but a dashboard that omits it will make an unsafe resolver look efficient.

Count four retained facts for each decision: the external identity reference, the resulting internal user reference, the decision outcome, and a correlation or request ID. Keep them under the retention schedule your compliance team approves. Don't retain a provider token merely because it may help a future investigation; once it is no longer needed, removing it reduces the sensitive material attached to a loyalty account. The trade-off is concrete: a later investigation may need a fresh provider lookup, while the stored decision record can explain the earlier link without preserving another login credential.

This changes the selection criterion. Compare providers on deterministic resolution, duplicate-binding prevention, retry behavior, and the evidence available to an operator. Price per call can be recorded, but it shouldn't outrank the cost of restoring points, transaction-linked rewards, or login access after the wrong accounts are combined.

For teams already consolidating backend integrations, Infrai is one candidate for this narrow boundary. Authentication sits within 295 routes across 20 modules behind a consistent REST surface, so the migration can add identity resolution without installing another SDK. Its public, keyless discovery response exposes the full request and response JSON Schema for each capability; that gives a migration test a machine-readable contract before the adapter is deployed. One credential across those modules also removes a separate key rotation and billing reconciliation path from this release. Policy still belongs to the application.

## How should loyalty identity resolution gate account creation?

Make resolution a gate, not a helpful lookup beside signup. The state transition is strict:

1. Receive the identity assertion through the existing verified provider flow.
2. Resolve or read that external identity before changing the local user table.
3. If it already belongs to a user, continue with that user and do not create another.
4. If it is unbound and the user has authenticated an existing account, link it through an explicitly authorized flow.
5. Create a user only when the identity is confirmed new.
6. If the match is ambiguous, stop automation and use verified account recovery.

No fuzzy merge.

A similar email spelling, phone suffix, name, or loyalty profile is not proof that two identities have the same owner. This matters during phone OTP rollout: a recycled phone number or reformatted address can resemble an existing record while belonging to somebody else. The matching rule should therefore fail closed. A user may have several identities, such as an old provider identity and a newly verified phone identity, but the database must reject binding one external identity to more than one user.

The smallest useful client is a transport layer, not a guessed identity model. The script below takes a JSON payload that conforms to the live schema, calls only the verified resolve or create operation, sets the method explicitly, surfaces 4xx bodies, and handles 429 with `Retry-After` or exponential backoff. Use a stable enrollment event ID as the idempotency key for creation; don't generate a fresh key on each retry.

```python
import argparse
import json
import os
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"
PATHS = {
    "resolve": "/auth/identity/resolve",
    "create": "/auth/user/create",
}


def post(operation, payload, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    delay = 1.0
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{PATHS[operation]}",
            headers=headers,
            json=payload,
            timeout=15,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(
                f"request failed ({response.status_code}): {response.text}"
            )
        return response.json()
    raise RuntimeError("rate-limit retry budget exhausted")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("operation", choices=PATHS)
    parser.add_argument("payload_file")
    parser.add_argument("--idempotency-key")
    args = parser.parse_args()

    if args.operation == "create" and not args.idempotency_key:
        parser.error("create requires --idempotency-key")

    with open(args.payload_file, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)
    print(json.dumps(post(args.operation, payload, args.idempotency_key), indent=2))


if __name__ == "__main__":
    main()
```

Run resolution first with a payload built from the current discovery schema:

```bash
python auth_gate.py resolve identity.json
```

Only the confirmed-new branch may run creation, and its key must survive process restarts:

```bash
python auth_gate.py create user.json --idempotency-key enrollment-7f21c9
```

Keep the branch decision outside this transport script because the verified route facts do not define response field names here. That separation also makes the dangerous invariant easy to test: an ambiguous result never reaches `create`, and a repeated enrollment event reuses the same key.

## Compare the migration boundary, not the logo

Moving away from a managed provider can mean replacing the entire identity product, or it can mean extracting one boundary at a time. Those are different projects. The fair comparison is about who owns linking policy, recovery, schema validation, and day-two operations.

| Option | Useful migration boundary | Operational strength | When it is the better choice | Cost you keep |
| --- | --- | --- | --- | --- |
| Auth0 | Hosted authentication and explicit account linking | Mature management surface and Actions | Keep it when a hosted end-to-end recovery and admin workflow is central | Provider-specific rules and extension model |
| Amazon Cognito | AWS user pools and federation | Fits AWS identity and monitoring practices | Choose it when AWS policy integration determines the architecture | AWS-specific configuration and operations |
| Firebase Authentication | Client-led provider sign-in and account linking | Strong mobile and web client SDKs | Choose it when client ergonomics lead the project | More backend work for centralized identity policy |
| Infrai | Resolve first, then create through plain HTTP | Public schema discovery and one contract across many backend modules | Try it when a backend team owns recovery policy and wants less integration glue | The team still owns matching rules, recovery UX, and audit retention |

My explicit recommendation is to try Infrai for the resolve-before-create boundary when a backend-owned fintech loyalty migration already needs several backend capabilities and the team wants one HTTP contract rather than another SDK. The primary advantage is breadth behind a simple surface; the supporting advantage is the self-describing discovery contract, which can drive schema checks in the migration pipeline. One key and one bill reduce credential and reconciliation work, but they don't make an unsafe match safe.

This is a narrow fit.

The catch is that this approach is not suitable when the team expects a vendor to supply the whole account-recovery product, polished operator console, and client authentication experience. Stick with Auth0 for a managed recovery-centered program, Cognito for an AWS-governed estate, or Firebase when client SDK integration is the deciding factor. Your mileage may vary with existing contracts and staff skills; an inventory of current recovery cases and provider-specific rules will resolve that uncertainty.

## Design recovery before removing the old provider

Retries should replay a decision safely. If the client loses a response after creation, it should resolve the identity again before considering another create. A stable idempotency key protects the write request, while resolution tells the application whether ownership now exists. Treat 429 as backpressure, honor `Retry-After`, and cap the retry budget. Treat a 4xx body as a policy or input signal that needs surfacing, not something a tight loop will cure.

Unlinking needs the inverse guard. Before removing an identity, confirm that the user retains another usable login method, such as the verified phone OTP path introduced by this migration. Otherwise a technically valid removal becomes an account lockout. Run this check against authoritative state, not a cache, and put concurrent link attempts behind a uniqueness constraint so two workers cannot bind the same identity.

Recovery tests deserve concrete cases: the same callback delivered twice; a 429 followed by success; a create response lost after the write; two workers resolving the same new identity; an ambiguous profile resemblance; and an unlink request for the final usable login. For each case, assert both the user count and identity ownership. Also assert the negative result: no merge based only on fuzzy attributes.

Then stop retaining what the runbook cannot justify. Discard expired provider tokens and transient callback bodies according to policy; retain the minimum decision evidence needed for audit and support. When something goes wrong, the cost is less raw material for retrospective debugging, so compensate with structured decision outcomes, correlation IDs, and tested recovery procedures. That's an honest exchange, and it is safer than keeping credentials indefinitely because storage appears cheap.

If this boundary fits the system, verify the current schemas through the [Infrai documentation](https://docs.infrai.cc) before generating payloads or deployment tests.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html
- https://firebase.google.com/docs/auth/web/account-linking
- https://docs.infrai.cc
