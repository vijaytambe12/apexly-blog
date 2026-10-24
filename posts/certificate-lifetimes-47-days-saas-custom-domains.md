---
title: "47-day TLS certificates: what SaaS teams issuing certificates for customer domains must change (2026)"
slug: certificate-lifetimes-47-days-saas-custom-domains
description: "Public TLS certificate lifetimes drop to 200 days in 2026, 100 days in 2027 and 47 days in 2029 (CA/Browser Forum ballot SC-081), and Let's Encrypt moves to 64 and then 45 days. What SaaS teams that issue certificates for customer custom domains must change: renewal timing, ARI, monitoring, CAA, rate limits and DNS alerts."
date: 2026-10-24
updated: 2026-10-24
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [47 day certificates, sc-081 certificate lifetime, tls certificate lifetime 2026, lets encrypt 45 day certificates, acme renewal information ari, saas custom domain ssl renewal]
faq:
  - q: "When do 47-day TLS certificates start?"
    a: "Under CA/Browser Forum ballot SC-081, publicly trusted TLS certificates issued on or after 15 March 2029 may be valid for at most 47 days. Earlier steps cap them at 200 days from 15 March 2026 and 100 days from 15 March 2027. The limits apply by issuance date, so existing certificates run until they expire."
  - q: "When should I renew certificates for customer domains?"
    a: "Use ACME Renewal Information (ARI) and renew at a random time inside the window the CA suggests. Without ARI, renew at about two-thirds of the certificate's lifetime, or 30 days before expiry, whichever comes first. Never hard-code a fixed number of days since issuance."
  - q: "Do Let's Encrypt rate limits apply to renewals?"
    a: "Renewals coordinated through ARI are exempt from all Let's Encrypt rate limits. Other renewals with the exact same set of names are exempt from the new-orders and per-registered-domain limits, but still count against the failed-authorization and exact-set limits."
  - q: "Why do shorter lifetimes make customer DNS problems worse?"
    a: "Domain validation reuse shrinks too, to 10 days by 2029 and to 7 hours at Let's Encrypt by 2028, so almost every renewal needs a fresh validation. If a customer's DNS no longer points at you, the renewal fails, and the time you have to warn them shrinks with the lifetime."
---

# 47-day TLS certificates: what SaaS teams issuing certificates for customer domains must change (2026)

**Short answer:** CA/Browser Forum ballot SC-081 caps public TLS certificates at 200 days from 15 March 2026, 100 days from 15 March 2027 and 47 days from 15 March 2029, and Let's Encrypt's default drops from 90 to 64 days in February 2027 and to 45 days in February 2028. If you issue certificates for customer custom domains, renewals stop being a rare background job and become constant traffic. Renew through ACME Renewal Information (ARI), or at about two-thirds of the lifetime, or 30 days before expiry. Watch the certificate that is actually served, not just the renewal job. Check CAA and rate limits before you order, and tell customers as soon as their DNS stops pointing at you.

*Last updated 24 October 2026 · Vijay Tambe, founder of Apexly*

---

## What is changing, and when

Two schedules matter. The first is the industry rule. The second is what your CA actually issues.

**The rule.** SC-081 passed in April 2025 with no votes against ([ballot page](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/)). It is now section 6.3.2 of the [TLS Baseline Requirements](https://github.com/cabforum/servercert/blob/main/docs/BR.md). It also shrinks how long a CA may reuse a past domain validation (section 4.2.1):

| Certificate issued | Maximum validity | Domain validation reuse |
|---|---|---|
| Before 15 March 2026 | 398 days | 398 days |
| 15 March 2026 to 14 March 2027 | 200 days | 200 days |
| 15 March 2027 to 14 March 2029 | 100 days | 100 days |
| From 15 March 2029 | 47 days | 10 days |

The limits apply by issuance date. A certificate issued the day before a cutoff keeps its full lifetime.

**Let's Encrypt.** Most custom-domain setups use Let's Encrypt, which already issues 90-day certificates. Its published plan ([Decreasing Certificate Lifetimes to 45 Days](https://letsencrypt.org/2025/12/02/from-90-to-45), [64-Day Certificate Lifetimes Coming Feb 2027](https://letsencrypt.org/2026/10/07/64-day-certs.html)):

| Date | Change |
|---|---|
| 13 May 2026 | The opt-in `tlsserver` profile issues 45-day certificates |
| 14 October 2026 | Staging issues 64-day certificates by default, for testing |
| 10 February 2027 | The default `classic` profile issues 64-day certificates; authorization reuse drops from 30 days to 10 |
| 16 February 2028 | `classic` issues 45-day certificates; authorization reuse drops to 7 hours |

Let's Encrypt expects the last 90-day certificate to expire on 11 May 2027. If you renew on a hard-coded schedule, you have until February 2027 to fix it.

## Why this hits custom-domain SaaS harder

Ten certificates for your own sites can survive manual renewal. A certificate for every customer domain can't, for three reasons.

1. **Volume multiplies.** Here's the arithmetic for 10,000 customer domains. Renewing 90-day certificates 30 days before expiry is one renewal every 60 days per domain, about 167 a day. Renewing 45-day certificates at two-thirds of their lifetime is one every 30 days, about 333 a day. Renewing them 30 days before expiry is one every 15 days, about 667 a day.
2. **You don't control the DNS.** Every renewal now needs a fresh validation, because the reuse window is shrinking to 10 days (and to 7 hours at Let's Encrypt). If the customer has moved DNS providers or deleted your record since the last renewal, it fails. With a 398-day certificate you might have found out next year. With a 45-day certificate you find out this month.
3. **Failures are quiet.** An expired certificate on `careers.acme.com` doesn't page anyone on your team. Your customer's visitors see a browser warning, and your customer finds out from them.

The work is in timing, monitoring, preflight checks and talking to customers. For the full picture of how customer domains fit together, start with the [complete guide to custom domains for SaaS](/blog/custom-domains-for-saas).

## 1. Renew on a schedule that scales with the lifetime

The most common bug Let's Encrypt calls out is a renewal hard-coded to a number of days: "renew at day 60", or a cron job with `80` in it. Their advice is to grep cron jobs, wrapper scripts and runbooks for numbers like 83, 80 or 60, then renew at about two-thirds of the lifetime instead.

Better still, ask the CA. **ARI** ([RFC 9773](https://www.rfc-editor.org/rfc/rfc9773)) lets the CA publish a suggested renewal window for each certificate. Your client fetches it, picks a uniform random time inside the window, and renews then. The CA can move the window earlier, for example before a mass revocation, and your fleet follows without a deploy. When you place the renewal order, include the `replaces` field with the old certificate's ID so the CA can link the two.

A renewal planner that prefers ARI and falls back sensibly:

```ts
const DAY = 86_400_000;

type Cert = {
  notBefore: Date;
  notAfter: Date;
  ari?: { start: Date; end: Date }; // suggestedWindow from the CA's renewalInfo endpoint
};

export function renewAt(cert: Cert): Date {
  // 1. ARI: a uniform random time inside the CA's window (RFC 9773, section 4.2)
  if (cert.ari) {
    const { start, end } = cert.ari;
    return new Date(start.getTime() + Math.random() * (end.getTime() - start.getTime()));
  }
  // 2. No ARI: two-thirds of the lifetime or 30 days before expiry, whichever is first
  const life = cert.notAfter.getTime() - cert.notBefore.getTime();
  const twoThirds = cert.notBefore.getTime() + (life * 2) / 3;
  const thirtyDaysLeft = cert.notAfter.getTime() - 30 * DAY;
  return new Date(Math.min(twoThirds, thirtyDaysLeft));
}
```

The math works from the certificate's own `notBefore` and `notAfter`, so the same code handles 90, 64, 45 and 47 days. Renewing 30 days before expiry is conservative on short certificates: for a 45-day certificate it means renewing on day 15, a third of the way in. You get more renewals, but a full month to fix a failure before anyone sees it.

## 2. Monitor the certificate you serve, not the job that renews it

"The renewal job exited 0" is not the same as "visitors get a fresh certificate". Things that fail silently:

- The renewal succeeded, but the new certificate never reached shared storage, or one node keeps serving a cached copy.
- The worker crashed, was scaled to zero, or its queue backed up behind a bulk import.
- Renewal for one domain has failed every day for two weeks, and the only record is a log line.

What to do instead:

- **Probe from outside.** For every *active* domain, open a TLS connection with that SNI name on a schedule and record the served certificate's `notAfter`. That's the number that matters.
- **Alert on a fraction of the lifetime, not a fixed day count.** A 14-day threshold is fine for 90-day certificates and too late for 47-day ones. Alert when less than about a quarter of the lifetime remains: that's past every renewal point above, so it should never fire.

Let's Encrypt recommends monitoring that alerts when certificates aren't renewed when expected, and keeps a list of [monitoring service options](https://letsencrypt.org/docs/monitoring-options/).

## 3. Check CAA and rate limits before you order

Every extra renewal is another chance to hit a problem that used to come up once a year.

**CAA.** If the customer's domain, or any parent, publishes CAA records that don't allow your CA, issuance fails ([Let's Encrypt on CAA](https://letsencrypt.org/docs/caa/)). Customers add CAA records months after setup, often when their security team standardises on one CA. Look up CAA before each order, not only the first one. If it blocks you, don't place the order. Mark the domain and show the exact record to add, such as `0 issue "letsencrypt.org"`.

**Rate limits.** Let's Encrypt's [rate limits](https://letsencrypt.org/docs/rate-limits/) (last updated 5 August 2026) that matter here:

| Limit | Value | Applies to renewals? |
|---|---|---|
| New orders per account | 300 every 3 hours | Not to ARI renewals or exact-set renewals |
| New certificates per registered domain | 50 every 7 days | Not to ARI renewals or exact-set renewals |
| Authorization failures per identifier per account | 5 per hour | Yes, unless renewed through ARI |
| ARI-coordinated renewals | Exempt from all limits | |

The pattern is clear: renew through ARI. And never retry a failing domain in a tight loop. Five failures in an hour locks that hostname out for your account, and the cause is almost always DNS or CAA, which retrying won't fix. Back off and fix the cause. More on these limits in [on-demand TLS for customer domains](/blog/on-demand-tls-for-customer-domains).

## 4. Tell the customer when DNS breaks, before the certificate does

With shrinking validation reuse, a renewal is effectively a re-verification of the customer's DNS. So keep checking DNS between renewals:

- Re-check every active domain's record on a schedule, from more than one resolver, so one stale cache doesn't trigger a false alarm.
- When the record stops pointing at you, keep serving the current certificate. It's still valid. Start the clock: the time you have left is the time until `notAfter`.
- **Email the customer straight away**, with the exact record that's missing or wrong and the date their HTTPS will break if it isn't fixed. Send a reminder when a third of that time is left.
- Show the state (`dns_record_missing`, `caa_blocks_issuance`) to your support team, so the ticket starts with an answer.

This is where shorter lifetimes hurt most: with 47-day certificates, a DNS change on day 20 leaves you weeks, not months, to reach someone at the customer who can edit DNS.

## A checklist before February 2027

- [ ] No renewal logic hard-codes days since issuance; it uses ARI, or two-thirds of the lifetime, or 30 days before expiry.
- [ ] Your ACME client supports ARI and sends `replaces` on renewal orders.
- [ ] You tested against Let's Encrypt staging, which has issued 64-day certificates since 14 October 2026.
- [ ] An external probe records the served `notAfter` for every active domain.
- [ ] Expiry alerts scale with the lifetime.
- [ ] CAA is checked before every order, with a customer-facing fix message.
- [ ] Failing domains back off instead of retrying in a loop.
- [ ] DNS is re-checked between renewals, and customers are emailed with a deadline when it breaks.

## How Apexly handles renewals

If you'd rather not run this yourself, [Apexly](https://apexly.dev) is a custom-domains API for B2B SaaS. It issues Let's Encrypt certificates automatically and renews them automatically, 30 days before expiry. It checks customer DNS from 3 resolvers. Signed webhooks (9 events, 7 retries) tell your app when a domain's state changes, so you can tell your customer. Setup is one API call per domain. The details are in the [docs](https://docs.apexly.dev).

## FAQ

### When do 47-day TLS certificates start?

Under CA/Browser Forum ballot SC-081, publicly trusted TLS certificates issued on or after 15 March 2029 may be valid for at most 47 days. Earlier steps cap them at 200 days from 15 March 2026 and 100 days from 15 March 2027. The limits apply by issuance date, so existing certificates run until they expire.

### When should I renew certificates for customer domains?

Use ACME Renewal Information (ARI) and renew at a random time inside the window the CA suggests. Without ARI, renew at about two-thirds of the certificate's lifetime, or 30 days before expiry, whichever comes first. Never hard-code a fixed number of days since issuance.

### Do Let's Encrypt rate limits apply to renewals?

Renewals coordinated through ARI are exempt from all Let's Encrypt rate limits. Other renewals with the exact same set of names are exempt from the new-orders and per-registered-domain limits, but still count against the failed-authorization and exact-set limits.

### Why do shorter lifetimes make customer DNS problems worse?

Domain validation reuse shrinks too, to 10 days by 2029 and to 7 hours at Let's Encrypt by 2028, so almost every renewal needs a fresh validation. If a customer's DNS no longer points at you, the renewal fails, and the time you have to warn them shrinks with the lifetime.

## Sources

- [CA/Browser Forum: Ballot SC081v3, Introduce Schedule of Reducing Validity and Data Reuse Periods](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/)
- [TLS Baseline Requirements](https://github.com/cabforum/servercert/blob/main/docs/BR.md), sections 4.2.1 and 6.3.2
- [Let's Encrypt: Decreasing Certificate Lifetimes to 45 Days](https://letsencrypt.org/2025/12/02/from-90-to-45) (2 December 2025)
- [Let's Encrypt: 64-Day Certificate Lifetimes Coming Feb 2027](https://letsencrypt.org/2026/10/07/64-day-certs.html) (7 October 2026)
- [Let's Encrypt: Rate Limits](https://letsencrypt.org/docs/rate-limits/) (updated 5 August 2026)
- [Let's Encrypt: CAA](https://letsencrypt.org/docs/caa/)
- [RFC 9773: ACME Renewal Information (ARI) Extension](https://www.rfc-editor.org/rfc/rfc9773)
