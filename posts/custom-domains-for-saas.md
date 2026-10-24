---
title: "How to add custom domains to your SaaS: the complete guide (2026)"
slug: custom-domains-for-saas
description: "How to let your SaaS customers use their own domain: DNS records (CNAME vs A), verification, TLS certificates, routing by Host header, renewals, apex domains, abuse and the failure states you will hit. With code and a checklist."
date: 2026-10-20
updated: 2026-10-20
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [saas custom domains, custom domains for saas, let customers use their own domain, white label domain saas, multi-tenant custom domains, customer custom domain ssl]
faq:
  - q: "How do SaaS custom domains work?"
    a: "The customer adds a DNS record (usually a CNAME) that points their hostname, such as careers.acme.com, at your infrastructure. You verify the record, issue a TLS certificate for that hostname, and route each request to the right tenant by reading the Host header."
  - q: "Should customers use a CNAME or an A record?"
    a: "Use a CNAME for subdomains like app.acme.com, because you can change your IP addresses without asking every customer to edit DNS. Apex domains like acme.com cannot have a CNAME, so they need A/AAAA records to stable IPs, or an ALIAS/flattened record if their DNS provider supports one."
  - q: "How do I get SSL certificates for customer domains?"
    a: "Use an ACME certificate authority such as Let's Encrypt. Issue a certificate per hostname only after you have verified the customer's DNS, store certificates in shared storage, and renew them automatically about 30 days before expiry."
  - q: "How long does it take for a custom domain to go live?"
    a: "Once the DNS record exists, verification and certificate issuance usually take under two minutes. Most of the waiting is DNS: the customer finding where to add the record, and caches expiring if an old record existed."
  - q: "Can I use Cloudflare for SaaS for custom domains?"
    a: "Yes. Cloudflare for SaaS (custom hostnames) handles certificates and routing at low per-hostname cost, but your own zone must be on Cloudflare and you still build the customer-facing setup instructions, verification UI and failure handling yourself."
  - q: "What is the hardest part of custom domains for SaaS?"
    a: "Not the certificate. It is everything around it: customers adding the wrong record at the wrong provider, apex domains, CAA records blocking issuance, renewals failing quietly, abuse, and support tickets. Plan for failure states from day one."
---

# How to add custom domains to your SaaS: the complete guide (2026)

**Short answer:** to let a customer serve your product on their own domain, (1) have them add one DNS record, a CNAME for a subdomain or A records for an apex domain, pointing at your servers; (2) verify that record from more than one resolver; (3) issue a TLS certificate for that exact hostname from an ACME CA such as Let's Encrypt; (4) route each request to the right tenant by its `Host` header; and (5) keep checking DNS and renewing certificates for as long as the domain exists. The certificate is the easy part. The work is in verification, apex domains, failure states and the customer's DNS panel.

*Last updated 20 October 2026 · Vijay Tambe, founder of Apexly*

---

## What "custom domains" means for a SaaS

Your customer Acme uses your applicant-tracking product. Today their careers page lives at `acme.yourapp.com`. They want it at `careers.acme.com`, with a padlock and no mention of you. Every B2B product whose customers put their name on a page you host gets this request eventually: help centres, status pages, booking pages, storefronts, docs portals, course platforms and link pages.

From the outside it's one feature. Underneath, it's five systems:

| System | Job | What goes wrong |
|---|---|---|
| DNS instructions | Tell the customer exactly which record to add, where | Wrong record type, wrong host field, record added at the wrong provider |
| Verification | Prove the hostname points at you before doing anything else | One stale resolver says yes, another says no |
| Certificates | Issue, store and renew a TLS certificate per hostname | CAA records, CA rate limits, renewals failing quietly |
| Routing | Send each request to the right tenant | Unknown hosts, spoofed headers, absolute URLs built from the wrong base |
| Lifecycle | Notice when DNS changes later and tell someone | The customer moves DNS providers and the site breaks with no alert |

The rest of this guide walks through each one, with the decisions you'll face.

## Step 1: choose the DNS record your customers add

### Subdomains: use a CNAME

For `careers.acme.com`, `help.acme.com` or `status.acme.com`, ask for a **CNAME** to a hostname you control:

```
careers.acme.com.   CNAME   d-k7qm.cname.yourapp.com.
```

Give each customer domain **its own target** (`d-k7qm…`) rather than one shared `custom.yourapp.com`. It costs nothing, it lets you move or block a single domain without touching DNS for anyone else, and it makes a misconfigured record easy to tell apart from a correct one.

A CNAME also means you can change your IP addresses, add regions or switch providers without asking every customer to edit DNS again. That alone is why A records to your IP should be the last resort.

### Apex domains: A/AAAA records (and why CNAME is impossible)

`acme.com` itself (the "apex" or "root") **cannot have a CNAME**. The DNS standards say a CNAME can't coexist with any other record at the same name ([RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2), [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1)), and every apex already has SOA and NS records. Your options:

1. **A and AAAA records** to IP addresses you promise never to change (anycast or Elastic IPs), plus a TXT record to prove ownership.
2. **ALIAS / ANAME / CNAME flattening**, if the customer's DNS provider offers it (Cloudflare, DNSimple and NS1 do; GoDaddy's standard DNS doesn't offer ALIAS; Route 53 aliases only point at AWS resources).
3. **Redirect the apex to `www`** and add a CNAME on `www`.

We cover the trade-offs in detail in [Apex domains for SaaS](/blog/apex-domains-for-saas).

### Tell them *where*, not just *what*

Most failed setups aren't technical. The customer doesn't know their DNS lives at GoDaddy rather than at the company that built their website, or they type `careers.acme.com` into a "Host" field that wanted `careers`. Look up the domain's NS records to detect the provider, and show instructions for **that** provider, in plain words, with the exact values to copy. This one step removes most custom-domain support tickets.

## Step 2: verify before you do anything else

Never request a certificate or route traffic for a hostname you haven't verified. Two rules:

- **Check from several resolvers.** Query at least two public resolvers (e.g. `1.1.1.1` and `8.8.8.8`) *and* the domain's own authoritative nameservers, and require them to agree. A single resolver can serve a cached answer for hours.
- **Poll on a schedule, then give up.** A reasonable cadence: every 15 seconds for the first hour (customers usually add the record right away), then every 5 minutes, and expire the request after about 72 hours.

Store a reason with every non-green state, such as `dns_record_missing`, `dns_record_mismatch` (with what you expected and what you found), or `dns_resolvers_disagree`. That reason is your support team's first line of defence, and the customer's.

For apex domains with A records, a matching A record doesn't prove the customer controls the domain (someone else's domain could point at your IPs too). Require a **TXT token**, e.g. `_yourapp.acme.com TXT "yourapp-verify=8f3a…"`, before issuing anything.

## Step 3: issue TLS certificates per hostname

### Use ACME, and pick a challenge

Let's Encrypt and ZeroSSL issue free certificates over the ACME protocol ([RFC 8555](https://www.rfc-editor.org/rfc/rfc8555)). For customer domains you'll almost always use:

| Challenge | How it proves control | Fits custom domains? |
|---|---|---|
| **HTTP-01** | CA fetches `http://host/.well-known/acme-challenge/…` on port 80 | Yes. Works as soon as DNS points at you. Keep port 80 open. |
| **TLS-ALPN-01** | CA connects on port 443 with a special ALPN protocol | Yes. Good when port 80 is closed. |
| **DNS-01** | You publish a TXT record in the customer's zone | Only if the customer delegates `_acme-challenge` to you with a CNAME. Needed for wildcards. |

### Respect the CA's rate limits

Let's Encrypt's limits ([current values](https://letsencrypt.org/docs/rate-limits/), updated August 2026) that matter for SaaS:

- **300 new orders per account per 3 hours.** A bulk import of 1,000 customer domains needs pacing or several accounts.
- **50 certificates per registered domain per 7 days.** Usually fine, since each customer has their own registered domain, *unless* you put many tenants under one domain you own.
- **5 failed validations per hostname per account per hour.** Retrying a broken domain in a tight loop locks it out. Back off.
- Renewals coordinated through **ACME Renewal Information (ARI)** are exempt from these limits, so use a client that supports ARI.

Keep a **second CA** configured (ZeroSSL, or a second Let's Encrypt account) so one CA's bad day doesn't stop new domains going live.

### Check CAA before you order

If the customer's domain (or a parent) has CAA records that don't list your CA, issuance fails. Look up CAA first and, if it blocks you, tell the customer the exact record to add (`0 issue "letsencrypt.org"`) instead of failing with a generic error.

### Store certificates once, share them everywhere

If you run more than one server, every node needs every certificate, and two nodes must never renew the same one at once. Use shared storage with locking (S3 with conditional writes, Redis, or Postgres) and keep private keys out of your application database and logs.

### Renew early, and watch lifetimes shrink

Let's Encrypt certificates last 90 days today. Start renewing around **30 days before expiry** and alert if any active domain gets within 14 days. Maximum lifetimes for publicly trusted certificates are falling under CA/Browser Forum ballot SC-081: **200 days from 15 March 2026, 100 days from 15 March 2027 and 47 days from 15 March 2029**. So build renewal as a background job you never think about, not as a cron script.

The certificate is the part with the most sharp edges at scale: on-demand issuance, the `ask` check, storage locking and limits. Our [on-demand TLS guide](/blog/on-demand-tls-for-customer-domains) covers that in depth with Caddy and certmagic examples.

## Step 4: route requests by Host header

Once TLS terminates, the request still says which domain the visitor typed: the `Host` header. Your app maps host to tenant:

```ts
// One lookup per request; cache it.
const host = req.headers.host.replace(/:\d+$/, "").toLowerCase();
const tenant = await tenantByHost(host);   // careers.acme.com → Acme
if (!tenant) return res.status(421).send("Unknown host");
```

Four things people get wrong:

1. **Unknown hosts.** Return `421 Misdirected Request` (or `404`) for any hostname you don't recognise. Never fall back to a default tenant.
2. **Trusting forwarding headers from anyone.** If a proxy sits in front of your app, only trust `X-Forwarded-*` (or a tenant header) from that proxy. Sign them or restrict by network.
3. **Absolute URLs.** Build links, redirects, OAuth callbacks and emails from the request host, not a fixed `BASE_URL`. Otherwise customers bounce back to `yourapp.com` mid-flow.
4. **Cookies and auth.** Cookies are per domain. A session on `app.yourapp.com` doesn't exist on `careers.acme.com`; design login for custom domains explicitly (a token handoff, or separate sessions per domain).

For framework specifics, see [custom domains in a multi-tenant Next.js app](/blog/nextjs-multi-tenant-custom-domains).

## Step 5: keep watching after go-live

A domain that worked on Monday can break on Friday: the customer moves DNS providers, a new IT person "cleans up" records, a CAA record appears. Re-check DNS every few hours, keep serving the existing certificate while you investigate, and **tell your own system** with a webhook or event (`domain.dns_broken`, `domain.cert_failed`) so you can email the customer before their visitors notice.

## Abuse and security checklist

Issuing certificates and proxying traffic for strangers' domains makes you a target. Before launch:

- [ ] Never issue a certificate for a hostname that isn't verified and attached to a paying (or card-verified) account.
- [ ] Screen hostnames and origins against Google Safe Browsing; hold lookalikes of well-known brands for review.
- [ ] Cap new domains per account per day; alert on bursts.
- [ ] Don't let one tenant claim a hostname another tenant already verified.
- [ ] Keep a kill switch that drops a route in seconds.
- [ ] Publish an abuse contact and act on phishing reports within 24 hours.

## Build, buy, or bolt on?

| Approach | Good when | You still build |
|---|---|---|
| **Build it** (Caddy/certmagic + your own DNS checks) | Domains are core to your product and you have someone to own it | Customer instructions, verification, failure states, renewals monitoring, abuse, on-call |
| **Cloudflare for SaaS** | You already run on Cloudflare and want the lowest per-hostname price | Customer-facing setup, verification UI, failure messages; your zone moves to Cloudflare |
| **Your host's domains feature** (Vercel, Render, Fly.io…) | Your whole app lives on that platform | Customer instructions; you're tied to that host |
| **A custom-domains API** (Apexly, Approximated, SaaS Custom Domains) | You want the feature, not the project | The Host-header lookup in your app |

Full side-by-sides with sources: [Apexly vs the alternatives](/compare).

## How Apexly does it (if you'd rather not build it)

Apexly is a custom-domains API: one request per customer hostname, and the customer finishes setup on a hosted connect page that detects their DNS provider and shows the exact record to add (one CNAME, or A + TXT for an apex). Apexly verifies DNS with three resolvers, issues certificates from Let's Encrypt, renews them, supports apex domains on every plan, and sends a signed webhook when a domain is verified, goes live, breaks, fails issuance, renews or is deleted. Your app reads the `Host` header. [Read the quickstart](https://docs.apexly.dev/quickstart). It's free for 20 domains with no card, then $0.15 per domain a month with a $15 minimum.

## FAQ

**How do SaaS custom domains work?**
The customer adds a DNS record, usually a CNAME, pointing their hostname at your infrastructure. You verify it, issue a TLS certificate for that hostname, and route each request to the right tenant by its Host header.

**Should customers use a CNAME or an A record?**
Use a CNAME for subdomains, so you can change IPs without asking customers to edit DNS. Apex domains can't have a CNAME, so they need A/AAAA records to stable IPs or an ALIAS/flattened record where the DNS provider supports one.

**How do I get SSL certificates for customer domains?**
Use an ACME CA such as Let's Encrypt. Issue per hostname only after verifying DNS, store certificates in shared storage with locks, and renew about 30 days before expiry.

**How long does a custom domain take to go live?**
Usually under two minutes after the record exists. Most of the time goes into the customer finding where to add the record, plus cache expiry if an old record was there.

**Can I use Cloudflare for SaaS?**
Yes. It's inexpensive per hostname, but your zone must be on Cloudflare, and the customer-facing setup, verification UI and failure handling are yours to build.

**What's the hardest part?**
Everything around the certificate: wrong records at the wrong provider, apex domains, CAA, quiet renewal failures, abuse, and the support tickets they create.

## Sources

- [RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2) and [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1): CNAME can't coexist with other data
- [RFC 8555](https://www.rfc-editor.org/rfc/rfc8555): ACME
- [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/) (updated 5 August 2026)
- [CA/Browser Forum ballot SC-081](https://cabforum.org/): reduction of certificate validity periods
- [Caddy on-demand TLS](https://caddyserver.com/on-demand-tls)
