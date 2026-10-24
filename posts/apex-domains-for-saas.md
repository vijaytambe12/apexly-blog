---
title: "Apex domains for SaaS: why customers can't CNAME acme.com, and 4 ways around it"
slug: apex-domains-for-saas
description: "Why a root domain like acme.com can't use a CNAME, and how SaaS products support apex custom domains anyway: A records to static IPs, ALIAS/ANAME and CNAME flattening, www redirects, and verification with TXT records."
date: 2026-10-20
updated: 2026-10-20
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [apex domain saas, cname at root domain, cname flattening, alias record, aname record, root domain custom domain, naked domain saas, a record vs cname]
faq:
  - q: "Can I put a CNAME on an apex (root) domain?"
    a: "No. A CNAME cannot coexist with any other record at the same name (RFC 1034 and RFC 2181), and every apex domain already has SOA and NS records. Some DNS providers simulate it with ALIAS, ANAME or CNAME flattening, which resolve the target and answer with A/AAAA records."
  - q: "How do SaaS products support apex domains like acme.com?"
    a: "Usually with A and AAAA records pointing at IP addresses the SaaS promises never to change (anycast or reserved IPs), plus a TXT record to prove ownership. Alternatives are ALIAS/flattened records where the customer's DNS provider supports them, or redirecting the apex to www."
  - q: "What is CNAME flattening?"
    a: "CNAME flattening (also called ALIAS or ANAME) is a DNS provider feature that lets you enter a hostname at the apex; the provider looks up that hostname's IP addresses and serves them as A/AAAA records. It is provider-specific and not part of the DNS standard."
  - q: "Why do apex domains need a TXT record for verification?"
    a: "Because A records only prove a domain points at your IP addresses, and anyone can point any domain at those IPs. A TXT token unique to the customer proves that the person adding the domain in your app controls its DNS."
  - q: "Should I just tell customers to use www?"
    a: "It's the simplest fallback: CNAME www to you and redirect the apex to www at the registrar. But registrar redirects often lack HTTPS, so https://acme.com may show a certificate error. Supporting the apex directly is a better experience."
---

# Apex domains for SaaS: why customers can't CNAME acme.com, and 4 ways around it

**Short answer:** a root domain like `acme.com` can't have a CNAME, because DNS forbids a CNAME next to other records and every apex already has SOA and NS records. To serve a customer's apex domain, a SaaS has four options: **(1) A/AAAA records to IP addresses that never change, plus a TXT ownership token**; (2) ALIAS/ANAME or CNAME flattening, where the customer's DNS provider supports it; (3) redirect the apex to `www` and CNAME `www`; or (4) have the customer move DNS to a provider that flattens. Option 1 works at every DNS provider, so it's the one to build first.

*Last updated 20 October 2026 · Vijay Tambe, founder of Apexly*

---

## Why a CNAME can't live at the apex

A CNAME says "this name is an alias; everything about it lives over there." The DNS spec says a name with a CNAME can't have any other data ([RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2), clarified in [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1)). The apex of every zone *must* have an SOA record and NS records. So:

```
acme.com.   SOA    ns1.provider.com. …     ← required
acme.com.   NS     ns1.provider.com.       ← required
acme.com.   CNAME  yourapp.com.            ← not allowed here
acme.com.   MX     mail.acme.com.          ← and it would also break email
```

Most DNS panels refuse the record outright. A few accept it and then quietly break the domain's email and nameserver delegation. That's why "just add a CNAME" instructions fail for `acme.com` but work for `careers.acme.com`.

## The 4 ways to support apex custom domains

| Option | Works at every DNS provider? | You need | Customer adds | Watch out for |
|---|---|---|---|---|
| **A/AAAA to static IPs** | Yes | IPs that never change (anycast or reserved) | 1–2 A records (+ AAAA) and a TXT | Your IPs become a promise forever |
| **ALIAS / ANAME / flattening** | No (provider-specific) | A hostname target | One ALIAS record | Each provider's UI and behaviour differ |
| **Redirect apex → www** | Mostly (registrar forwarding) | Nothing extra | Forwarding rule + CNAME on www | Registrar forwarding often has no HTTPS |
| **Move DNS to a flattening provider** | n/a | Nothing | A nameserver change | Huge ask for a non-technical customer |

### Option 1: A and AAAA records to static IPs (build this first)

Publish a small set of IPv4 and IPv6 addresses and promise in your docs that they'll never change. Customers add:

```
acme.com.         A      203.0.113.10
acme.com.         A      203.0.113.11
acme.com.         AAAA   2001:db8::10
_yourapp.acme.com. TXT   "yourapp-verify=8f3a1c…"
```

**Why the TXT record matters:** an A record only proves `acme.com` points at your IPs, and so do thousands of other customers' domains. Anyone could type `acme.com` into your app. A unique TXT token, under a prefix like `_yourapp`, proves the person adding the domain controls its DNS. Check both before you issue a certificate.

**Making the IPs permanent.** Single-server setups can use reserved addresses (AWS Elastic IPs, GCP static IPs). Multi-region setups use **anycast** (Fly.io dedicated IPs, AWS Global Accelerator, BYOIP), so the same addresses reach the nearest region. Either way, treat changing them as a breaking change for every apex customer.

**Mind other A records.** If the customer's apex already has A records pointing at their old website host, leaving them in place means visitors are randomly split between the two. Your verification should flag "found extra A records" as its own state, with the records to delete.

### Option 2: ALIAS, ANAME and CNAME flattening

Several DNS providers let you enter a hostname at the apex and resolve it for you, answering visitors with A/AAAA records:

| Provider | Feature name | Points at any hostname? |
|---|---|---|
| Cloudflare | CNAME flattening (automatic at the apex) | Yes |
| DNSimple | ALIAS | Yes |
| NS1 | ALIAS | Yes |
| Amazon Route 53 | Alias record | **No**, only AWS resources (CloudFront, ELB, S3…) |
| GoDaddy | Not offered in standard DNS | n/a |

Where it works, it's the nicest option, because you can move IPs freely. But you can't count on it: a large share of small businesses' domains sit at registrars without it. Offer it as "if your DNS is at Cloudflare, DNSimple or NS1, add this instead", detected from the domain's NS records.

### Option 3: redirect the apex to www

Ask for a CNAME on `www.acme.com` and a registrar redirect from `acme.com` to `https://www.acme.com`. It's easy, but:

- Many registrar forwarding services only serve **HTTP**, so `https://acme.com` throws a certificate warning before it redirects.
- Path and query handling differs per registrar.
- You can't tell from your side whether the redirect exists or works.

Fine as a fallback. Not great as the only answer.

### Option 4: move DNS to a provider that flattens

Technically clean; practically a big ask. Changing nameservers moves *all* of a company's DNS, including email. Non-technical customers (the people who usually do custom-domain setup) shouldn't be asked to do it to try your product.

## Serve both acme.com and www.acme.com

Customers will type both. Treat the pair as one setup: verify and issue certificates for both names, serve the primary one, and 308-redirect the other to it **over HTTPS on your side**, so the redirect has a valid certificate. Let the customer choose which is primary.

## What a good apex flow looks like

1. Customer enters `acme.com`. You detect it's an apex (use the [Public Suffix List](https://publicsuffix.org/), not "has one dot": `acme.co.uk` is also an apex).
2. You look up the NS records, detect the provider, and show **either** an ALIAS instruction (if it's supported there) **or** the A/AAAA + TXT records.
3. You check from multiple resolvers and report precisely: "A record found ✓, TXT record missing ✗, extra A record 198.51.100.4 → delete it."
4. Once everything matches, you issue the certificate (HTTP-01 works fine on the apex) and serve.
5. You offer to add `www` as a pair.

## How Apexly handles apex domains

Apexly supports apex domains on every plan with **A records and a `_apexly` TXT token**. The connect page detects the customer's DNS provider and shows exactly those records, and the API reports precise reasons (`apex_a_mismatch`, `apex_txt_missing`) when something's off. Add `www` as its own domain and set `apex_redirect: "www"`, and Apexly 308-redirects the apex to `https://www.` with a valid certificate. [See how it works](https://apexly.dev/how-it-works).

## FAQ

**Can I put a CNAME on an apex domain?**
No. A CNAME can't coexist with other records, and every apex has SOA and NS records. ALIAS, ANAME and CNAME flattening are provider features that simulate it.

**How do SaaS products support apex domains?**
Mostly with A/AAAA records to IPs they promise never to change, plus a TXT token for ownership. ALIAS/flattening and apex-to-www redirects are alternatives.

**What is CNAME flattening?**
A DNS provider feature that accepts a hostname at the apex and answers with that hostname's current A/AAAA records. It's provider-specific, not part of the DNS standard.

**Why do apex domains need a TXT record?**
Because a matching A record proves nothing about ownership: anyone can point a domain at your IPs. A unique TXT token proves the person adding it controls the DNS.

**Should I just tell customers to use www?**
It's an acceptable fallback, but registrar redirects often lack HTTPS, so `https://acme.com` may show a certificate error. Supporting the apex directly is better.

## Sources

- [RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2), [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1)
- [Amazon Route 53: choosing between alias and non-alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [Cloudflare: CNAME flattening](https://developers.cloudflare.com/dns/cname-flattening/)
- [Public Suffix List](https://publicsuffix.org/)
