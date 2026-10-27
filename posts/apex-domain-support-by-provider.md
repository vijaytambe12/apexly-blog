---
title: "Which custom domain providers support apex domains? (2026)"
slug: apex-domain-support-by-provider
description: "Checked 26 Oct 2026: most custom domain providers support apex domains with an A record to fixed IPs; Cloudflare for SaaS needs Enterprise apex proxying."
date: 2026-10-27
updated: 2026-10-27
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [apex domain support, custom domain providers apex domain, cloudflare for saas apex proxying, root domain custom domain saas, a record apex domain, cname at apex, custom domain api comparison, naked domain saas]
faq:
  - q: "Which custom domain providers support apex domains?"
    a: "On their own pages checked 26 October 2026, Approximated, Domainee, Entri, Vercel, Render, Fly.io, CoAlias and a self-hosted Caddy setup all document apex (root) domains, mostly through an A record to fixed IPs. Cloudflare for SaaS supports them through apex proxying, listed as a paid Enterprise add-on."
  - q: "Does Cloudflare for SaaS support apex domains on the Free, Pro or Business plan?"
    a: "Not through apex proxying. Cloudflare's Cloudflare for SaaS plans page lists Apex proxying/BYOIP as No on Free, Pro and Business and as a paid add-on on Enterprise. Without it, the usual pattern is a subdomain or `www` for those customers."
  - q: "Why do custom domain providers use A records for apex domains?"
    a: "Because DNS does not allow a CNAME at the apex: a name with a CNAME can have no other records (RFC 1034 and RFC 2181), and every apex already has SOA and NS records. An A record to an IP address the provider keeps fixed works at every DNS provider."
  - q: "Is ALIAS or CNAME flattening a good way to point an apex domain at a SaaS?"
    a: "It works where the customer's DNS provider offers it, and Render lists it first for root domains. It is provider-specific, though, and Domainee's docs note that flattening resolves their hostname to one address, pinning the apex to one of their regions, so they recommend A records."
  - q: "How do I test whether a provider's apex domain setup works?"
    a: "Check that the apex has no CNAME, that two public resolvers return only the provider's A records, that there is no stale AAAA record, that any ownership TXT record and CAA record are right, and that the certificate's SAN list includes the apex. dig, openssl and curl cover all of it."
---

# Which custom domain providers support apex domains? (2026)

**Short answer:** as of 26 October 2026, every provider checked here supports customer apex domains such as `acme.com`, but not in the same way. Approximated, Domainee, Entri, Vercel, Render, Fly.io, CoAlias and a self-hosted Caddy setup ask the customer for A records to the provider's IPs. Cloudflare for SaaS uses apex proxying, a paid Enterprise add-on.

*Last updated 27 October 2026 · vendor pages checked 26 October 2026 · Vijay Tambe, founder of Apexly*

---

## Why is apex support its own question?

A subdomain like `careers.acme.com` can point at a SaaS with one CNAME. The apex, `acme.com` itself, can't. A name that has a CNAME can't hold any other data ([RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2), restated in [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1)), and the apex of every zone must carry SOA and NS records. So a provider that only offers "add a CNAME" has no answer for a customer who wants their product on `acme.com`.

Providers fill that gap in different ways: A records to fixed IPs, anycast A records, ALIAS or flattened records, static IP prefixes, or a redirect to `www`. Those choices decide which of your customers can finish setup without a support ticket. The "why" and the four general workarounds are covered in [Apex domains for SaaS](/blog/apex-domains-for-saas). This post answers the buying question: who supports the apex today, how, and on which plan.

## Which providers support apex domains? The matrix

Every row comes from the vendor's own public page, fetched on 26 October 2026. "Not stated" means that page doesn't say; it does not mean the feature is missing. No prices are quoted. Check the linked page before you decide, because docs change.

| Provider | Apex supported? | What the customer adds at the apex | Ownership proof stated for apex | Plan needed | Source (checked 26 Oct 2026) |
|---|---|---|---|---|---|
| Cloudflare for SaaS | Yes, through apex proxying | A records to static IP prefixes Cloudflare assigns to your account, or your own via BYOIP | Not stated on the apex proxying page | Plans page: No on Free, Pro and Business; paid add-on on Enterprise | [Apex proxying](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/start/advanced-settings/apex-proxying/), [Plans](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/plans/) |
| Approximated | Yes | An A record to your proxy cluster's IPv4 address | Not stated | Not stated | [How Approximated works](https://approximated.app/docs/#how-approximated-works) |
| Domainee | Yes | Two anycast A records (recommended); ALIAS, ANAME or flattening also documented as working | Not stated in the apex section | Not stated | [Quickstart: apex / root domains](https://domainee.dev/docs/quickstart) |
| Entri (Power) | Yes | An A record whose value Entri fills in with its server IPs (`{ENTRI_SERVERS}`) | Not stated | Not stated | [Power configuration](https://developers.entri.com/power/configuration), [Power endpoints](https://developers.entri.com/power/endpoints) |
| Vercel | Yes | An A record with the value shown on the project's domain card, or Vercel nameservers | A TXT record at `_vercel` when the domain is already in use on Vercel | Not stated | [Add a domain](https://vercel.com/docs/domains/working-with-domains/add-a-domain), [A records KB](https://vercel.com/kb/guide/a-record-and-caa-with-vercel), [Multi-tenant domains](https://vercel.com/docs/platforms/multi-tenant-platforms/configuring-domains) |
| Render | Yes | An ALIAS or ANAME to the service's `onrender.com` name where supported, otherwise an A record to Render's load balancer IP; remove AAAA records | Not stated (a Verify button in the dashboard) | Custom domains included on each listed plan; count varies by plan | [Custom domains](https://render.com/docs/custom-domains), [Other DNS providers](https://render.com/docs/configure-other-dns) |
| Fly.io | Yes | A and AAAA records (recommended at the apex); CNAME only with flattening | At least one of: AAAA to the app, an `_acme-challenge` CNAME, or a `_fly-ownership` TXT record | Not stated | [Custom domains](https://fly.io/docs/networking/custom-domain/) |
| CoAlias | Yes | An `@` A record to the IP on its help page | Not stated | Not stated | [Configure your DNS](https://helpcenter.coalias.com/hc/coalias/articles/1737396794-configure-your-dns) |
| Caddy (self-hosted) | Yes, you run it | A/AAAA records to your own servers' IPs, which you keep stable | Yours to build: Caddy's `ask` endpoint must approve each domain before on-demand TLS issues | Self-hosted | [Automatic HTTPS: On-Demand TLS](https://caddyserver.com/docs/automatic-https#on-demand-tls) |
| Apexly | Yes | A records plus a `_apexly` TXT record | The `_apexly` TXT record | Free plan covers 20 live domains, no card | [Pricing](https://apexly.dev/pricing) |

Two details from the vendor pages are easy to miss:

- **Render pairs the apex with `www` for you.** Add `example.org` and Render adds `www.example.org` and redirects it to the root, and the reverse if you add `www` first.
- **Entri and Render both care about AAAA.** Entri's renewal endpoint requires that a root domain resolve to Entri's A records "with no AAAA records present". Render asks you to remove AAAA records because it uses IPv4. Fly.io goes the other way and recommends both A and AAAA at the apex.

## How do the apex methods compare?

The matrix boils down to five methods. Each has a different cost for you and a different ask of your customer.

| Method | Who documents it here | Works at every DNS provider? | What you take on |
|---|---|---|---|
| A record to the provider's IPs | Approximated, Entri, Vercel, Render, Fly.io, CoAlias, Caddy | Yes | Keeping that IP forever; changing it means every apex customer edits DNS |
| Anycast A records | Domainee | Yes | Same promise, with routing to the nearest region done by the network |
| A records to static IP prefixes | Cloudflare for SaaS (apex proxying) | Yes | An Enterprise contract and the add-on, or your own IP range |
| ALIAS, ANAME or CNAME flattening | Render (listed first), Domainee and Fly.io (where supported) | No | Per-provider instructions; some flatten to a single address |
| Redirect apex to `www` | Render (automatic pairing), Vercel (redirect setting) | Mostly | An HTTPS redirect that must be served with a valid certificate |

A records keep working for a customer whose DNS sits at a registrar with no ALIAS support, and nothing else in the table does that without a contract or a redirect. That is why nearly every row uses it. The price of that portability is a permanent promise about IP addresses, and that promise is worth asking any provider about directly.

## Which apex method fits your customers?

This is the decision table we use when a SaaS asks which provider shape to pick. Start from who your customers are, not from the provider list.

| Your situation | Method that fits | Why |
|---|---|---|
| Customers are non-technical and their DNS is at GoDaddy, Namecheap or similar | A record to fixed IPs, plus a TXT ownership record | Works at every DNS provider; the TXT record stops someone claiming a domain they don't control |
| Most customers' DNS is at Cloudflare, DNSimple or NS1 | ALIAS or flattening is acceptable; A records are still safer | Flattening is provider-specific, and one provider's docs say it can pin the apex to one region |
| You already run Cloudflare for SaaS on Free, Pro or Business | Offer subdomains or `www` with a redirect, or budget for Enterprise apex proxying | Apex proxying is not on those plans per Cloudflare's plans page |
| Your customers' domains often carry old AAAA records | A provider whose docs say how they treat AAAA, and a pre-check for it | A stale AAAA sends IPv6 visitors and some validation traffic to the old host |
| You self-host and want full control | Caddy on-demand TLS behind stable IPs, with an `ask` endpoint | You own verification, IP stability and renewals monitoring; see [on-demand TLS for customer domains](/blog/on-demand-tls-for-customer-domains) |
| A customer insists on `www.acme.com` as the main address | Serve both names and redirect the apex to `www` on your side over HTTPS | Registrar forwarding often has no HTTPS; your own redirect does |

## How do you test a provider's apex support before you commit?

Docs tell you what a provider intends. A test domain tells you what happens. Point a spare domain's apex at the provider, then run this checklist. We ran every command below on 26 October 2026 with dig 9.20, OpenSSL 3.5 and curl 8.14. Replace `acme.com` with your test domain.

- [ ] **Find where the customer's DNS lives.** The answer decides whether ALIAS is even an option.

```bash
dig +short NS acme.com @1.1.1.1
```

- [ ] **Confirm there is no CNAME at the apex.** Empty output is correct. A CNAME here will break email and NS delegation.

```bash
dig +noall +answer CNAME acme.com @1.1.1.1
```

- [ ] **Check that two public resolvers agree and return only the provider's addresses.** Any extra A record (an old website host, a registrar parking IP) splits traffic.

```bash
dig +short A acme.com @1.1.1.1
dig +short A acme.com @8.8.8.8
```

- [ ] **Look for stale AAAA records.** If the provider is IPv4-only and this returns anything, IPv6 visitors reach the old host.

```bash
dig +short AAAA acme.com @1.1.1.1
```

- [ ] **Check the ownership record, if the provider uses one.** Use the provider's prefix, such as `_vercel` or `_fly-ownership`.

```bash
dig +short TXT _yourprovider.acme.com @1.1.1.1
```

- [ ] **Check CAA.** Empty output means any CA may issue. If records exist, one must name the provider's CA, for example `0 issue "letsencrypt.org"`.

```bash
dig +short CAA acme.com @1.1.1.1
```

- [ ] **Confirm the certificate covers the apex.** The `subjectAltName` line must include `acme.com`, and `notAfter` shows when renewal has to happen.

```bash
echo | openssl s_client -connect acme.com:443 -servername acme.com 2>/dev/null \
  | openssl x509 -noout -subject -enddate -ext subjectAltName
```

- [ ] **Check HTTPS on the apex and the `www` pairing.** Expect `200` on the primary name and a `301` or `308` with a `https://` target on the other.

```bash
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://acme.com/
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://www.acme.com/
```

If your network intercepts DNS on port 53 (some offices and CI runners do), add `+https` to each `dig` command to send the query to 1.1.1.1 or 8.8.8.8 over DNS-over-HTTPS. That is how we ran them.

Then ask the provider four questions the docs often leave out:

1. Will the apex IP addresses ever change, and how much notice will you give?
2. Do you serve IPv6 at the apex, and what should customers do with existing AAAA records?
3. How does your verification report an extra A record: as its own error, or as a generic "pending"?
4. Is apex support on every plan, or only some?

## What should you build if you do it yourself?

If you run the apex path on your own servers, the order matters: verify the TXT ownership record first, then check that every A record points at you, then let your TLS server issue. With Caddy that gate is the `ask` endpoint. The rest of the job (renewals, failure states, the customer's DNS panel) is laid out step by step in the pillar guide, [how to add custom domains to your SaaS](/blog/custom-domains-for-saas). For a worked example of how one API reports a missing apex TXT record, see Apexly's docs page for [apex_txt_missing](https://docs.apexly.dev/troubleshooting/apex_txt_missing).

## FAQ

### Which custom domain providers support apex domains?

On their own pages checked 26 October 2026, Approximated, Domainee, Entri, Vercel, Render, Fly.io, CoAlias and a self-hosted Caddy setup all document apex (root) domains, mostly through an A record to fixed IPs. Cloudflare for SaaS supports them through apex proxying, listed as a paid Enterprise add-on.

### Does Cloudflare for SaaS support apex domains on the Free, Pro or Business plan?

Not through apex proxying. Cloudflare's Cloudflare for SaaS plans page lists Apex proxying/BYOIP as No on Free, Pro and Business and as a paid add-on on Enterprise. Without it, the usual pattern is a subdomain or `www` for those customers.

### Why do custom domain providers use A records for apex domains?

Because DNS does not allow a CNAME at the apex: a name with a CNAME can have no other records (RFC 1034 and RFC 2181), and every apex already has SOA and NS records. An A record to an IP address the provider keeps fixed works at every DNS provider.

### Is ALIAS or CNAME flattening a good way to point an apex domain at a SaaS?

It works where the customer's DNS provider offers it, and Render lists it first for root domains. It is provider-specific, though, and Domainee's docs note that flattening resolves their hostname to one address, pinning the apex to one of their regions, so they recommend A records.

### How do I test whether a provider's apex domain setup works?

Check that the apex has no CNAME, that two public resolvers return only the provider's A records, that there is no stale AAAA record, that any ownership TXT record and CAA record are right, and that the certificate's SAN list includes the apex. dig, openssl and curl cover all of it.

## Sources

- [RFC 1034 §3.6.2](https://www.rfc-editor.org/rfc/rfc1034#section-3.6.2) and [RFC 2181 §10.1](https://www.rfc-editor.org/rfc/rfc2181#section-10.1): a CNAME can't coexist with other data
- Cloudflare: [Apex proxying](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/start/advanced-settings/apex-proxying/) and [Cloudflare for SaaS plans](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/plans/)
- Approximated: [How Approximated works](https://approximated.app/docs/#how-approximated-works)
- Domainee: [Quickstart](https://domainee.dev/docs/quickstart)
- Entri: [Power configuration](https://developers.entri.com/power/configuration) and [Power endpoints](https://developers.entri.com/power/endpoints)
- Vercel: [Add a domain](https://vercel.com/docs/domains/working-with-domains/add-a-domain), [A records and CAA](https://vercel.com/kb/guide/a-record-and-caa-with-vercel), [Multi-tenant domains](https://vercel.com/docs/platforms/multi-tenant-platforms/configuring-domains)
- Render: [Custom domains](https://render.com/docs/custom-domains) and [Configuring other DNS providers](https://render.com/docs/configure-other-dns)
- Fly.io: [Custom domains](https://fly.io/docs/networking/custom-domain/)
- CoAlias: [Configure your DNS](https://helpcenter.coalias.com/hc/coalias/articles/1737396794-configure-your-dns)
- Caddy: [Automatic HTTPS, On-Demand TLS](https://caddyserver.com/docs/automatic-https#on-demand-tls)

If you'd rather not build this, Apexly does it in one API call. Free for 20 domains, no card.
