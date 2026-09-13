# Per-Tenant Sending Domain Records in One Idempotent TXT Job (and What Proves It)

The constraint in a multi-tenant developer-tools product is that subdomain creation sits on the signup path rather than in a change window: a customer types `acme`, and a minute later your platform has to be authorized to send mail from `acme.notify.example-tools.dev`, with nobody watching and no operator to finish the job by hand if half of it lands. Use one idempotent job keyed by that sending domain to upsert all three TXT records — SPF, DKIM, DMARC — and then verify the domain inside the same job, storing the verdict as the deliverability evidence you will be asked for the first time a tenant complains about the spam folder.

Verification is the part teams skip.

Writing three records is trivial. Proving that the records you meant to publish are the records now serving, and keeping that proof attached to the tenant row, is the part that decides whether your support engineer can answer "is this domain actually authorized to send?" in ten seconds or in an afternoon of `dig` archaeology.

## Three records, one write path, and no read-your-writes

DNS is a replicated store you write to through one authority and read from through thousands of caches you don't control, which means a 200 on the write is an acknowledgement of durability at the authoritative side and nothing else. No read-your-writes. A resolver that already has a negative answer cached for `_dmarc.acme.notify.example-tools.dev` will keep serving it for the remainder of the negative TTL, and your job cannot make that go faster.

So the job has exactly two jobs: make the intended state durable, then obtain an independent statement about what is observable. Everything else — retries, alerting, the tenant's onboarding screen — hangs off those two facts.

The three records are three different names, and treating them as one blob is where most implementations go wrong. SPF is a TXT record at the sending domain itself. DKIM is a TXT record at `<selector>._domainkey.<sending-domain>`, which means the selector is part of your key rotation design, not a constant. DMARC is a TXT record at `_dmarc.<sending-domain>`, and per RFC 7489 a subdomain with no policy of its own inherits the organizational domain's policy, including whatever `sp=` says — so a tenant subdomain can be silently governed by a record you wrote for a different purpose two years ago.

Keep those three names in configuration, generated from the tenant's sending domain and the active DKIM selector. Inline string concatenation at three call sites is how a rotation ends up writing the new key to the old selector.

## How should one job publish the SPF, DKIM and DMARC TXT records and verify a sending domain?

Make the whole job a function of the sending domain, so a rerun is a no-op rather than a second copy. Upsert is doing real work here: `create` semantics turn a partial failure into a duplicate record on the retry, and duplicates at the SPF name are not a cosmetic problem — RFC 7208 says an evaluator that finds more than one applicable SPF record returns `permerror`, which is a worse outcome than having published nothing at all.

The order I'd ship: derive a stable key per record from the domain and the record name, upsert each record, verify the domain once, then persist both the verification verdict and the exact content string you wrote. That last part matters more than it looks — deliverability debugging always starts with what the record actually says, and a job that logs "3 records written" tells you nothing a week later.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
ZONE_ID = os.environ["TENANT_ZONE_ID"]
SENDING_DOMAIN = os.environ["TENANT_SENDING_DOMAIN"]
# [{"name": "acme.notify.example-tools.dev", "content": "v=spf1 include:..."}, ...]
RECORDS = json.loads(os.environ["TENANT_TXT_RECORDS"])


def idempotency_key(label: str) -> str:
    seed = f"{SENDING_DOMAIN}:{label}".encode("utf-8")
    return hashlib.sha256(seed).hexdigest()


def call(method: str, path: str, payload: dict, key: str) -> dict:
    data = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=data,
            method=method,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {exc.code} on {path}: {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry budget exhausted")


written = []
for record in RECORDS:
    call(
        "PUT",
        "/dns/record/upsert",
        {
            "zone_id": ZONE_ID,
            "type": "TXT",
            "name": record["name"],
            "content": record["content"],
        },
        idempotency_key(record["name"]),
    )
    written.append({"name": record["name"], "content": record["content"]})

verdict = call(
    "POST",
    "/email/domain/verify",
    {"domain": SENDING_DOMAIN},
    idempotency_key("verify"),
)
print(json.dumps({"domain": SENDING_DOMAIN, "records": written, "verification": verdict}))
```

Two routes, one process, no orchestration framework: `PUT /v1/dns/record/upsert` for each of the three names and `POST /v1/email/domain/verify` once at the end. The idempotency key is derived rather than random, because a random key regenerated on retry defeats the entire mechanism; deriving it from the domain and the record name gives you the same key on every attempt for the same intent.

What the program deliberately does not do is loop until the domain reports verified. Propagation is external, the process holding an HTTP connection open adds nothing to it, and a job that retries verification in a tight loop is spending call volume to learn what a scheduled recheck would have told it for free. Run the same job again later; it is safe to re-run by construction.

## The failure modes I plan for before the first tenant signs up

Partial completion is the default failure, not the exotic one. If the DKIM record lands and the SPF record does not, the mail is signed and unaligned, which produces the specific kind of intermittent delivery that gets escalated as "email is broken" when the actual state is "two of three records exist."

| Record | Name | What you lose if it is missing or wrong |
| --- | --- | --- |
| SPF | the sending domain | Path authorization; two records at this name yield `permerror` |
| DKIM | `<selector>._domainkey.<domain>` | Signature validation, and with it alignment under a strict DMARC policy |
| DMARC | `_dmarc.<domain>` | Policy control and aggregate reports — the subdomain silently inherits the parent policy |

The second failure mode is length. A 2048-bit DKIM public key exceeds the 255-octet limit on a single DNS character-string from RFC 1035, so the record has to be published as multiple character-strings that resolvers concatenate; whether your provider's API expects you to do that splitting or does it for you is the first thing to test, because a truncated key validates as a malformed signature rather than as a missing one.

Third: drift. You verified on day one, the tenant edited their zone on day ninety, and your database still says `verified` because nothing ever re-read it. Store the verification verdict with a timestamp, treat it as a cached observation with an expiry, and re-run the job on a schedule — the same job, unchanged, which is the payoff for making it idempotent in the first place.

And a smaller one I'd still test: the negative-cache TTL on the parent zone's SOA, because it bounds how long a resolver that asked too early keeps saying `NXDOMAIN` for a name you have already published.

## Which control plane should own the tenant subdomain?

The decision axis is not feature count. It is where the record write and the deliverability evidence live relative to each other, and how many credentials your onboarding path has to hold to produce both.

| Option | What it owns | Sensible when | Why you would pass |
| --- | --- | --- | --- |
| Cloudflare DNS | Zone and record writes | Cloudflare is already the authoritative control plane | Mail evidence still comes from a separate sending provider |
| Amazon Route 53 | Zone and record writes | AWS-native ownership is a stated platform constraint | The onboarding service must stay outside an AWS boundary |
| DNSimple | Zone and record writes, with a small API surface | A team wants a DNS-focused provider and nothing else | Sending-domain verification remains a second integration |
| octoDNS or external-dns | Declarative zone state from config or Kubernetes | Zones are reviewed like code and change on a deploy cadence | Per-tenant writes arrive at signup time, not at deploy time |
| Infrai | DNS records and sending-domain verification behind one key and one bill | You want one credential and one invoice for the record write and the verification, over plain REST with no SDK to install | You need provider-specific DNS controls that a common interface does not expose |

Reconciling credentials is a real cost that rarely shows up in a design document, and one key covering both halves of this job removes an entire class of onboarding failure where the DNS integration is healthy and the mail integration's key expired. The catch is the usual one for any unified interface: the moment you need a provider-specific control — a Route 53 alias record, a Cloudflare-proxied hostname, a registrar-level setting — you are back to the native API, and a common abstraction is not suitable for that work.

Stick with Cloudflare or Route 53 when DNS governance already belongs to an infrastructure team with its own review process. Choose octoDNS when zones are genuinely config-as-code and per-tenant churn is low enough to batch. I'm not sure any of these is wrong for a product doing a handful of tenant subdomains a month; the argument for consolidating only gets strong when signups are automatic and nobody is watching the failures.

## Rolling it out across tenants you already have

Do the backfill as a diff, not as a write. Run the job in a read-only mode first that computes the intended three records per tenant and compares them against what is published, because the interesting output of a migration is the list of tenants whose current records disagree with your configuration — those are your existing deliverability problems, and you want them on a list before you overwrite the evidence.

Then roll forward in batches, lowering TTLs on the affected names a propagation interval ahead of the change so a mistake expires quickly.

Start DMARC at `p=none` with an `rua` address, read aggregate reports for a couple of weeks, and tighten to quarantine or reject only once the reports show your own sending aligned. Publishing `p=reject` on day one for tenants whose historical mail flows you have never measured is how a migration turns into an incident. Keep the job scheduled after the backfill completes, alert on tenants that stay unverified past your onboarding SLA, and let the idempotent rerun handle everything else.

## References

- RFC 7208, "Sender Policy Framework (SPF) for Authorizing Use of Domains in Email, Version 1": https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, "DomainKeys Identified Mail (DKIM) Signatures": https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, "Domain-based Message Authentication, Reporting, and Conformance (DMARC)": https://datatracker.ietf.org/doc/html/rfc7489
- RFC 1035, "Domain Names — Implementation and Specification" (character-string length limit): https://datatracker.ietf.org/doc/html/rfc1035
- Cloudflare DNS records documentation: https://developers.cloudflare.com/dns/manage-dns-records/
- Amazon Route 53 Developer Guide: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- DNSimple zone records API: https://developer.dnsimple.com/v2/zones/records/
