---
title: "On-demand TLS for customer domains: Caddy, certmagic and Let's Encrypt limits (2026)"
slug: on-demand-tls-for-customer-domains
description: "How to issue SSL certificates for thousands of customer domains automatically: on-demand TLS with Caddy and certmagic, the ask endpoint, shared storage and locking, Let's Encrypt rate limits, CAA records, renewals and the 47-day certificate future."
date: 2026-10-20
updated: 2026-10-26
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [on demand tls, caddy on_demand_tls ask, certmagic custom domains, ssl for customer domains, let's encrypt rate limits saas, automatic https custom domains, saas ssl certificates]
faq:
  - q: "What is on-demand TLS?"
    a: "On-demand TLS means the server obtains a certificate for a hostname the first time a client connects to it during the TLS handshake, instead of configuring every hostname in advance. Caddy and the Go library certmagic support it, which makes it a common way to serve customer custom domains."
  - q: "What is the Caddy on_demand_tls ask endpoint?"
    a: "An internal HTTP endpoint Caddy calls with ?domain=<hostname> before obtaining a certificate. Return 200 only if the hostname belongs to a verified customer. Without it, anyone can point domains at your server and make it request certificates until you hit CA rate limits."
  - q: "What are Let's Encrypt's rate limits for SaaS?"
    a: "As of August 2026: 300 new orders per account per 3 hours, 50 certificates per registered domain per 7 days, 5 certificates per exact set of names per 7 days, and 5 failed validations per hostname per account per hour. Renewals through ACME Renewal Information (ARI) are exempt."
  - q: "How do I share certificates across multiple servers?"
    a: "Use a shared storage backend with locking, for example certmagic's storage interface on S3 with conditional writes, Redis or Postgres, so every node can serve every certificate and only one node issues or renews a given name at a time."
  - q: "Why did certificate issuance fail for one customer domain?"
    a: "The usual causes are DNS not pointing at you yet, a CAA record that doesn't allow your CA, port 80 blocked for the HTTP-01 challenge, an old AAAA record pointing elsewhere, or a rate limit after repeated failures."
---

# On-demand TLS for customer domains: Caddy, certmagic and Let's Encrypt limits (2026)

**Short answer:** to give every customer domain HTTPS automatically, run a TLS server that **obtains certificates on demand** (Caddy, or the Go library certmagic that powers it), and **gate issuance with an "ask" check** that only approves hostnames you've already verified. Store certificates in **shared storage with locks** so every node can serve them, respect Let's Encrypt's rate limits (300 orders per account per 3 hours; 5 failed validations per hostname per hour), check CAA records before ordering, keep a second CA as fallback, and renew through ARI about 30 days before expiry.

*Last updated 20 October 2026 · Vijay Tambe, founder of Apexly*

---

## Why on-demand TLS fits custom domains

The classic setup lists every hostname in config and reloads. With customer domains you don't know the hostnames in advance, there may be tens of thousands of them, and new ones arrive all day. On-demand TLS flips it: when a TLS handshake arrives with SNI `careers.acme.com` and there's no certificate yet, the server gets one right then (a few seconds) and caches it.

## The minimal Caddy setup, and why it isn't enough

```caddyfile
{
	on_demand_tls {
		ask http://localhost:5555/allowed
	}
}

https:// {
	tls {
		on_demand
	}
	reverse_proxy app:3000
}
```

Caddy calls `http://localhost:5555/allowed?domain=careers.acme.com` before each new certificate ([Caddy docs](https://caddyserver.com/on-demand-tls)). **The `ask` endpoint is not optional.** Without it, anyone can point any domain at your IP, or spray random SNI names, and your server will try to get certificates for all of them until your CA account is rate-limited and real customers can't go live.

## Write the ask endpoint properly

```ts
// GET /allowed?domain=careers.acme.com  → 200 to allow, anything else to refuse
app.get("/allowed", async (req, res) => {
  const host = String(req.query.domain ?? "").toLowerCase().replace(/\.$/, "");
  const domain = await db.domain.findUnique({ where: { hostname: host } });
  const ok =
    domain &&
    domain.verifiedAt !== null &&       // DNS verified from several resolvers
    domain.deletedAt === null &&
    domain.heldReason === null &&       // not held for abuse review
    domain.account.status === "ACTIVE";
  res.status(ok ? 200 : 404).end();
});
```

Rules that keep it safe:

- **Only verified hostnames.** "Exists in the database" isn't enough. Verify DNS first (see the [complete custom domains guide](/blog/custom-domains-for-saas#step-2-verify-before-you-do-anything-else)).
- **Fast and local.** It runs inside the handshake; cache answers for a few seconds and never call a remote service from it.
- **Fail closed.** If the database is down, refuse. Existing certificates keep serving from storage.

## Share certificates across nodes, with locks

One server is easy. Two or more need:

1. **Shared storage**, so a certificate issued on node A is served by node B. certmagic has a `Storage` interface with community backends (S3, Redis, Consul, Postgres).
2. **Distributed locks**, so two nodes that receive the first handshake at the same moment don't both order the same certificate (that's how you burn the *5 per exact set per week* limit).
3. **ACME challenge sharing.** For HTTP-01, the CA's request can land on any node behind your load balancer, so the challenge token must be in shared storage too.

On S3 you can build locks with **conditional writes** (`If-None-Match: *`): create `locks/<name>` only if it doesn't exist, with an expiry so a crashed node can't hold it forever. Keep private keys in that storage, encrypted, and **never** in your app database or logs.

## Know the Let's Encrypt rate limits (August 2026 values)

| Limit | Value | What it means for SaaS |
|---|---|---|
| New orders per account | 300 / 3 hours (refills 1 per 36 s) | Pace bulk imports; consider several accounts |
| Certificates per registered domain | 50 / 7 days | Only bites if many tenants share one domain you own |
| Certificates per exact set of names | 5 / 7 days | Don't re-issue the same name in a loop; locks matter |
| Failed validations per hostname per account | 5 / hour | Back off on broken domains; never hot-retry |
| Consecutive failures per hostname | pauses issuance for that name, up to 10 days | Stop retrying names that never validate |
| ARI renewals | **exempt from all limits** | Use an ACME client with ARI support |

Source: [letsencrypt.org/docs/rate-limits](https://letsencrypt.org/docs/rate-limits/) (updated 5 August 2026). If you build this yourself, configure **ZeroSSL or a second CA** as fallback so an outage or rate limit at one CA doesn't block new domains.

## Check CAA before ordering

A CAA record such as `acme.com. CAA 0 issue "digicert.com"` means only DigiCert may issue for `acme.com` *and every subdomain*. If you order anyway, validation fails and counts against your failure limit. So:

1. Look up CAA for the hostname, walking up the tree to the registrable domain.
2. If records exist and none allow your CA, don't order. Mark the domain `cert_failed` with the exact fix: *add `0 issue "letsencrypt.org"` at `acme.com`*.
3. Re-check with backoff (1 min, 5 min, 30 min, 2 h, then every 12 h) and order as soon as it's allowed.

## The failure modes you'll actually see

| Symptom | Likely cause | Fix to show the customer |
|---|---|---|
| Validation timeout | Port 80 blocked, or traffic not reaching you yet | Open port 80 (HTTP-01) or switch to TLS-ALPN-01 |
| `unauthorized` from CA | An old AAAA record points elsewhere, so the CA checked over IPv6 | Delete the stale AAAA record |
| CAA error | CAA doesn't list your CA | Add `0 issue "letsencrypt.org"` |
| `rateLimited` | Repeated failures or a bulk import | Wait, back off, use ARI and (if you run your own stack) a fallback CA |
| Works for some visitors only | Old A records still present next to yours | Remove the extra A records |

## Renewals in a world of 47-day certificates

Let's Encrypt certificates last 90 days today. Under CA/Browser Forum ballot SC-081, maximum lifetimes for public certificates drop to **200 days from 15 March 2026, 100 days from 15 March 2027 and 47 days from 15 March 2029**. Renewals will be routine and frequent, so:

- Renew at about **two-thirds of the lifetime** (day 60 of 90), using ARI's suggested window where available.
- Alert when any *active* domain's certificate is within 14 days of expiry. That alert should never fire.
- Keep serving the current certificate while renewals retry; never take a domain down because renewal is late.

## Or skip running it

If running TLS for strangers' domains isn't your product, a custom-domains API runs it for you. Apexly does on-demand issuance gated by DNS verification from three resolvers, Let's Encrypt certificates spread across three ACME accounts, CAA checks with exact fix messages, shared S3 storage with server-side AES-256 encryption, and renewals that start 30 days early. Your origin receives the request with the original `Host` header and a signature. [Quickstart](https://docs.apexly.dev/quickstart).

## FAQ

**What is on-demand TLS?**
Getting a certificate for a hostname during the first TLS handshake for it, rather than configuring hostnames in advance. Caddy and certmagic support it.

**What is the Caddy `on_demand_tls` ask endpoint?**
An internal URL Caddy calls with `?domain=` before issuing. Return 200 only for verified customer hostnames; otherwise anyone can make you request certificates until you're rate-limited.

**What are Let's Encrypt's rate limits for SaaS?**
300 orders per account per 3 hours, 50 certificates per registered domain per week, 5 per exact set of names per week, 5 failed validations per hostname per hour. ARI renewals are exempt.

**How do I share certificates across servers?**
Shared storage plus locking: a certmagic storage backend on S3 (conditional writes), Redis or Postgres.

**Why did issuance fail for one domain?**
Usually DNS not pointing at you yet, a CAA record, port 80 blocked, a stale AAAA record, or a rate limit after repeated failures.

## Sources

- [Caddy: On-Demand TLS](https://caddyserver.com/on-demand-tls)
- [Let's Encrypt: Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [Let's Encrypt: CAA](https://letsencrypt.org/docs/caa/)
- [RFC 8555 (ACME)](https://www.rfc-editor.org/rfc/rfc8555), [RFC 9773 (ACME Renewal Information)](https://www.rfc-editor.org/rfc/rfc9773)
- [certmagic](https://github.com/caddyserver/certmagic)
- CA/Browser Forum ballot SC-081 ([summary by DNSimple](https://support.dnsimple.com/articles/announcement-ssl-certificate-validity-changes/))
