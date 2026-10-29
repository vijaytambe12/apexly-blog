---
title: "Customer domain on Cloudflare: fix 1014, 525 and SSL pending"
slug: cloudflare-proxy-errors-custom-domains
description: "If your customer's custom domain is proxied by Cloudflare (orange cloud), set that one record to DNS only. That clears most 1014, 525 and SSL-pending errors."
date: 2026-10-29
updated: 2026-10-29
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [cloudflare error 1014 custom domain, cloudflare 525 custom domain, custom domain ssl pending cloudflare, orange cloud custom domain, cloudflare proxy cname saas, dns only cloudflare custom domain, cloudflare too many redirects custom domain, cloudflare o2o]
faq:
  - q: "Why does my customer's custom domain show Cloudflare error 1014?"
    a: "Error 1014 (CNAME Cross-User Banned) means a CNAME points from a zone in one Cloudflare account to a hostname in another account. Cloudflare only allows that when the owner of the target uses Cloudflare for SaaS. If you don't, your customer must point at a target that isn't proxied in another Cloudflare account."
  - q: "Should my customer's CNAME be proxied (orange cloud) or DNS only?"
    a: "DNS only (grey cloud), unless your platform runs on Cloudflare for SaaS and supports O2O. A proxied record hides your CNAME target behind Cloudflare's IPs, so your verification and certificate checks can't see it, and the customer's own Cloudflare settings sit in front of your app."
  - q: "What does error 525 mean on a custom domain?"
    a: "Error 525 means Cloudflare's edge could not complete a TLS handshake with your server while the customer's zone is in Full or Full (strict) mode. On a new custom domain it usually means your server has no certificate yet for that hostname, often because verification failed while the record was proxied."
  - q: "How can I tell if a customer's record is proxied by Cloudflare?"
    a: "Look the hostname up. A proxied record returns Cloudflare anycast IPs (in the ranges Cloudflare publishes at cloudflare.com/ips) and no CNAME, even though the customer created a CNAME to your target. A DNS-only record returns your CNAME target."
  - q: "Why does a custom domain on Cloudflare loop with too many redirects?"
    a: "Usually the customer's zone uses Flexible mode, so Cloudflare talks to your server over plain HTTP, and your server redirects every HTTP request to HTTPS. Each redirect comes back over HTTP again. Setting the record to DNS only, or the zone to Full (strict), ends the loop."
---

# Customer domain on Cloudflare: fix 1014, 525 and SSL pending

**Short answer:** when a customer's custom domain fails and their DNS is at Cloudflare, the cause is almost always the orange cloud. A proxied record hides your CNAME target, puts the customer's Cloudflare settings in front of your app, and blocks verification and certificate issuance. Ask them to set that one record to DNS only. That clears most 1014, 525 and "SSL pending" cases.

*Last updated 29 October 2026 · Vijay Tambe, founder of Apexly*

This post is for the team running the SaaS, not the end customer. It explains what each error means on your side, how to detect the cause before the customer opens a ticket, and what to tell them. For the full lifecycle of a custom domain, start with [How to add custom domains to your SaaS](/blog/custom-domains-for-saas).

## What does the orange cloud actually change?

In Cloudflare's DNS dashboard, every A, AAAA and CNAME record has a proxy status. Cloudflare's [proxy status docs](https://developers.cloudflare.com/dns/proxy-status/) describe the two states:

- **Proxied (orange cloud):** DNS queries are answered with Cloudflare anycast IP addresses, not the record's real value. Proxied CNAMEs are flattened, so the CNAME to your target never appears in public DNS. HTTP and HTTPS traffic goes through Cloudflare first, with the customer's own SSL mode, redirects, WAF and cache rules.
- **DNS only (grey cloud):** DNS returns the real record. Your CNAME target is visible, and visitors connect straight to you.

Cloudflare recommends proxying records that serve web traffic, and proxying is on by default when a domain is onboarded in the dashboard. So a customer who follows your "add a CNAME" instructions at Cloudflare can end up proxied without choosing to.

Four things break for you when that happens:

1. **Your DNS check fails.** You expect `careers.acme.com CNAME domains.yourapp.com`. Public DNS shows two Cloudflare IPs instead. Your domain stays "pending" forever.
2. **The certificate authority can't reach you directly.** Let's Encrypt connects to Cloudflare, not to your server. [TLS-ALPN-01](https://letsencrypt.org/docs/challenge-types/) can't work at all, because Cloudflare terminates TLS. HTTP-01 depends on settings in the customer's Cloudflare account that you can't see.
3. **The visitor's TLS session ends at Cloudflare.** The certificate the browser sees is the customer's Cloudflare edge certificate, not yours. Cloudflare then opens a second connection to your server, using the customer's SSL mode.
4. **The customer's settings can change under you.** They can switch the SSL mode, add redirect or WAF rules, or turn the proxy on months after setup. Cloudflare's [Automatic SSL/TLS](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/) can also change a zone's encryption mode on its own. None of this shows up on your side until something fails.

## Which error means what? The decision table

Each row is a symptom you or your customer will see, what is happening, and who has to act.

| Symptom | What is happening | Who fixes it | Fix |
|---|---|---|---|
| **Error 1014: CNAME Cross-User Banned** | The customer's record CNAMEs to a hostname proxied in a *different* Cloudflare account (yours), and you don't use Cloudflare for SaaS. Cloudflare prohibits that ([Error 1014](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1014/)). | You | Use Cloudflare for SaaS custom hostnames, or give customers a target that isn't proxied by your Cloudflare account. |
| **Error 1000: DNS points to prohibited IP** | The customer's A record points at a Cloudflare IP, or your setup sends the request back through Cloudflare a second time. With a Cloudflare for SaaS provider on BYOIP, it also appears when no custom hostname exists for the domain ([Error 1000](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1000/)). | Customer or you | Customer: replace the A record with the values you gave them. You: create the custom hostname before telling the customer to point DNS. |
| **Domain stuck on "pending" or "verifying"** | Your DNS check sees Cloudflare IPs and no CNAME, so it never matches. | Customer | Set the record to DNS only. |
| **Error 525: SSL handshake failed** | The customer's zone is in Full or Full (strict) mode, so Cloudflare connects to you over HTTPS. Your server has no certificate for that hostname yet, often because verification never passed ([Error 525](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-525/)). If you run on-demand TLS with an ask endpoint, it refuses the unverified name and the handshake fails. | Customer | DNS only, then let your normal verification and issuance run. |
| **Error 526: invalid SSL certificate** | Full (strict) mode, and the certificate on your server for that hostname is missing, expired or doesn't list it ([Error 526](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-526/)). | Customer, sometimes you | DNS only. If the record must stay proxied, you need a valid public certificate for the exact hostname. |
| **ERR_TOO_MANY_REDIRECTS** | Flexible mode: Cloudflare talks to you over plain HTTP, your app redirects HTTP to HTTPS, and the redirect comes back over HTTP again ([redirect loops](https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/)). | Customer | DNS only, or Full (strict) if they insist on proxying. |
| **Error 522: connection timed out** | Cloudflare can't open a TCP connection to your server. The most common cause Cloudflare lists is a firewall that rate limits or blocks Cloudflare's IP ranges ([Error 522](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-522/)). | You | Check your firewall or WAF for rules against Cloudflare ranges. Or ask for DNS only. |
| **Certificate error only on deeper names** like `help.eu.acme.com` | In a full-setup zone, Universal SSL covers the root and first-level subdomains only. A proxied `help.eu.acme.com` gets no valid edge certificate ([Universal SSL limitations](https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/)). | Customer | DNS only, so your certificate is the one served. |
| **Certificate issued once, renewal fails later** | The record was DNS only when you issued, then the customer turned the proxy on (or their SSL mode changed). Your renewal check now reaches Cloudflare. | Customer | DNS only. Watch for this between renewals: see [47-day certificates](/blog/certificate-lifetimes-47-days-saas-custom-domains). |

The pattern: nearly every row ends with "set the record to DNS only". The exceptions are 1014 and 522, which are about your own setup.

## How do I detect a proxied record before the customer complains?

A proxied record has a clear fingerprint: the hostname resolves to addresses inside [Cloudflare's published IP ranges](https://www.cloudflare.com/ips/), and a CNAME query returns nothing, even though the customer created a CNAME.

From a terminal:

```bash
# 1. Does the CNAME show up? A DNS-only record prints your target.
dig +short CNAME careers.acme.com @1.1.1.1

# 2. What addresses does the name resolve to?
dig +short A careers.acme.com @1.1.1.1
dig +short AAAA careers.acme.com @1.1.1.1

# 3. Are those addresses Cloudflare's? Compare with the published ranges.
curl -s https://www.cloudflare.com/ips-v4/
curl -s https://www.cloudflare.com/ips-v6/
```

If step 1 is empty and step 2 returns addresses inside those ranges, the record is proxied.

To run the same check from your backend before you try to issue a certificate, here is a dependency-free Node.js script (Node 18 or later). It asks two public resolvers over DNS-over-HTTPS, so a local resolver or cache can't skew the result, and compares the answers against Cloudflare's ranges. I ran it on 28 October 2026 against `www.cloudflare.com` (verdict `proxied_cname_hidden`) and `docs.github.com` (verdict `not_cloudflare`).

```ts
// cf-proxy-check.mjs: is a customer's hostname behind Cloudflare's proxy?
// Usage: node cf-proxy-check.mjs careers.acme.com [domains.yourapp.com]
const [host, expectedTarget] = process.argv.slice(2);
if (!host) {
  console.error("usage: node cf-proxy-check.mjs <hostname> [expected-cname-target]");
  process.exit(2);
}

const RESOLVERS = {
  google: "https://dns.google/resolve",
  cloudflare: "https://cloudflare-dns.com/dns-query",
};
const TYPES = { A: 1, AAAA: 28, CNAME: 5 };

async function lookup(resolver, name, type) {
  const url = `${RESOLVERS[resolver]}?name=${encodeURIComponent(name)}&type=${type}`;
  const res = await fetch(url, { headers: { accept: "application/dns-json" } });
  const body = await res.json();
  // Keep only answers of the requested type (A/AAAA lookups also return the CNAME chain).
  return (body.Answer || []).filter((r) => r.type === TYPES[type]).map((r) => r.data.replace(/\.$/, ""));
}

async function cloudflareRanges() {
  const get = async (u) => (await (await fetch(u)).text()).split("\n").map((s) => s.trim()).filter(Boolean);
  const [v4, v6] = await Promise.all([get("https://www.cloudflare.com/ips-v4/"), get("https://www.cloudflare.com/ips-v6/")]);
  return { v4, v6 };
}

const v4ToInt = (ip) => ip.split(".").reduce((n, o) => n * 256 + Number(o), 0);
function inV4(ip, cidr) {
  const [base, bits] = cidr.split("/");
  const size = 2 ** (32 - Number(bits));
  return Math.floor(v4ToInt(ip) / size) === Math.floor(v4ToInt(base) / size);
}
function v6ToBigInt(ip) {
  const [head, tail = ""] = ip.split("::");
  const h = head ? head.split(":") : [];
  const t = tail ? tail.split(":") : [];
  const parts = ip.includes("::") ? [...h, ...Array(8 - h.length - t.length).fill("0"), ...t] : h;
  return parts.reduce((n, p) => (n << 16n) + BigInt(parseInt(p || "0", 16)), 0n);
}
function inV6(ip, cidr) {
  const [base, bits] = cidr.split("/");
  const shift = 128n - BigInt(bits);
  return v6ToBigInt(ip) >> shift === v6ToBigInt(base) >> shift;
}

const ranges = await cloudflareRanges();
const report = { host, resolvers: {} };
let proxiedSomewhere = false;

for (const r of Object.keys(RESOLVERS)) {
  const [cname, a, aaaa] = await Promise.all([lookup(r, host, "CNAME"), lookup(r, host, "A"), lookup(r, host, "AAAA")]);
  const cloudflareIps = [
    ...a.filter((ip) => ranges.v4.some((c) => inV4(ip, c))),
    ...aaaa.filter((ip) => ranges.v6.some((c) => inV6(ip, c))),
  ];
  if (cloudflareIps.length) proxiedSomewhere = true;
  report.resolvers[r] = { cname, a, aaaa, cloudflareIps };
}

const anyCname = Object.values(report.resolvers).flatMap((x) => x.cname.map((c) => c.toLowerCase()));
if (expectedTarget) report.cnameMatchesTarget = anyCname.includes(expectedTarget.toLowerCase());

report.verdict = !proxiedSomewhere
  ? "not_cloudflare"
  : anyCname.length === 0
    ? "proxied_cname_hidden" // orange cloud: Cloudflare IPs, no visible CNAME
    : "cloudflare_ips_with_visible_cname"; // e.g. your own target is on Cloudflare

console.log(JSON.stringify(report, null, 2));
process.exit(report.verdict === "not_cloudflare" ? 0 : 1);
```

How to use the verdict:

- `proxied_cname_hidden`: stop. Don't start an ACME order: it will fail and count against your [failed-validation limit](/blog/on-demand-tls-for-customer-domains). Show the customer the DNS-only message below.
- `cloudflare_ips_with_visible_cname`: the CNAME is visible but the chain lands on Cloudflare. That is expected if your own target is on Cloudflare. If it isn't, look at what the target resolves to.
- `not_cloudflare`: carry on with your normal checks, including `cnameMatchesTarget`.

Fetch the range lists once a day and cache them rather than on every check. Apexly's Domain doctor runs a Cloudflare proxy check alongside its DNS, CAA and certificate checks, and names the one blocking check to fix first ([Domain doctor docs](https://docs.apexly.dev/troubleshooting/domain-doctor)).

## Can I support customers who want to keep the proxy on?

Sometimes a customer has a real reason to keep the orange cloud, such as their own WAF rules or a policy that every record is proxied. Whether you can support it depends on how you serve custom domains:

| Your setup | Customer record proxied | What happens |
|---|---|---|
| Cloudflare for SaaS custom hostnames | Supported | This is [O2O](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/saas-customers/how-it-works/) (orange-to-orange). The customer's zone settings apply first, then yours. You can spot it from the `cf-connecting-o2o: 1` request header. O2O works only with a CNAME, not with A records at the apex. |
| Your target proxied by your own Cloudflare account, no Cloudflare for SaaS | Fails | Error 1014. |
| Your own proxy or load balancer (Caddy, nginx, a managed service) issuing Let's Encrypt certificates | Unreliable | Verification and issuance both depend on the customer's SSL mode, redirect settings and WAF. It may work today and break after a settings change. |

If you are on your own proxy, the honest answer is to require DNS only. Say so in your setup instructions, before the customer adds the record, rather than after the first failure.

One more trap with apex domains: if a name has several A records and *any* of them is proxied, Cloudflare treats all of them as proxied ([Mix proxied and unproxied](https://developers.cloudflare.com/dns/proxy-status/#mix-proxied-and-unproxied)). A customer who turns off the cloud on one of two apex A records has changed nothing. For how apex records work in general, see [Apex domains for SaaS](/blog/apex-domains-for-saas).

## What should I tell the customer?

Customers on Cloudflare usually don't know their record is proxied. Give them one exact change, with the record named. You can paste this into your help docs or your "pending" screen:

> Your domain `careers.acme.com` is set to **Proxied** in Cloudflare, so we can't see the record or issue your certificate. In Cloudflare, open **DNS → Records** for `acme.com`, edit the `careers` record and set **Proxy status** to **DNS only** (grey cloud). Leave the target as `domains.yourapp.com`. We check again automatically, and you don't need to do anything else.

Three tips from building these messages:

- Name the exact record and zone. "Turn off the orange cloud" leaves the customer guessing which of their 40 records you mean.
- Say what stays the same (the target), so they don't delete and recreate the record.
- Don't tell customers to change their zone-wide SSL mode. It affects every site they run on Cloudflare. A per-record proxy change only affects your hostname.

If you'd rather customers on Cloudflare didn't edit records by hand at all, Apexly offers one-click Cloudflare: when the customer's DNS is at Cloudflare, they can connect in one click, with a scoped Cloudflare token that is used once and never stored.

## A checklist for Cloudflare-hosted customer domains

Run these in order. Each step depends on the one before it.

1. **Proxy:** CNAME query returns your target, and the A/AAAA answers are not Cloudflare IPs. If not, ask for DNS only.
2. **Other records:** no leftover A or AAAA records on the same name next to the CNAME. On the apex, every A record is DNS only.
3. **Your target:** if your target is proxied in your own Cloudflare account, you are on Cloudflare for SaaS. Otherwise expect 1014.
4. **CAA:** the customer's CAA records, if any, allow your CA ([Let's Encrypt CAA docs](https://letsencrypt.org/docs/caa/)).
5. **Certificate:** your certificate for the exact hostname issued, and `openssl s_client -connect careers.acme.com:443 -servername careers.acme.com` shows your certificate, not a Cloudflare edge certificate.
6. **Origin:** a request with the customer's Host header reaches the right tenant.
7. **Re-check before every renewal:** proxy status can change at any time. Run step 1 again before each renewal and alert on change.

## FAQ

### Why does my customer's custom domain show Cloudflare error 1014?

Error 1014 (CNAME Cross-User Banned) means a CNAME points from a zone in one Cloudflare account to a hostname in another account. Cloudflare only allows that when the owner of the target uses Cloudflare for SaaS. If you don't, your customer must point at a target that isn't proxied in another Cloudflare account.

### Should my customer's CNAME be proxied (orange cloud) or DNS only?

DNS only (grey cloud), unless your platform runs on Cloudflare for SaaS and supports O2O. A proxied record hides your CNAME target behind Cloudflare's IPs, so your verification and certificate checks can't see it, and the customer's own Cloudflare settings sit in front of your app.

### What does error 525 mean on a custom domain?

Error 525 means Cloudflare's edge could not complete a TLS handshake with your server while the customer's zone is in Full or Full (strict) mode. On a new custom domain it usually means your server has no certificate yet for that hostname, often because verification failed while the record was proxied.

### How can I tell if a customer's record is proxied by Cloudflare?

Look the hostname up. A proxied record returns Cloudflare anycast IPs (in the ranges Cloudflare publishes at cloudflare.com/ips) and no CNAME, even though the customer created a CNAME to your target. A DNS-only record returns your CNAME target.

### Why does a custom domain on Cloudflare loop with too many redirects?

Usually the customer's zone uses Flexible mode, so Cloudflare talks to your server over plain HTTP, and your server redirects every HTTP request to HTTPS. Each redirect comes back over HTTP again. Setting the record to DNS only, or the zone to Full (strict), ends the loop.

## Sources

- Cloudflare: [Proxy status](https://developers.cloudflare.com/dns/proxy-status/), [Encryption modes](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/), [Universal SSL limitations](https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/), [Redirect loops](https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/), [IP ranges](https://www.cloudflare.com/ips/)
- Cloudflare errors: [1000](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1000/), [1014](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1014/), [522](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-522/), [525](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-525/), [526](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-526/)
- Cloudflare for SaaS: [How O2O works](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/saas-customers/how-it-works/), [Getting started](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/start/getting-started/)
- Let's Encrypt: [Challenge types](https://letsencrypt.org/docs/challenge-types/), [CAA](https://letsencrypt.org/docs/caa/)
- ACME: [RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555), TLS-ALPN-01 in [RFC 8737](https://datatracker.ietf.org/doc/html/rfc8737)

If you'd rather not build this, Apexly does it in one API call. Free for 20 domains, no card.
