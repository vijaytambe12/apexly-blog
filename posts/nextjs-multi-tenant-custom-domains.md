---
title: "Custom domains in a multi-tenant Next.js app: routing by Host header (Next.js 16)"
slug: nextjs-multi-tenant-custom-domains
description: "How to serve customer custom domains from one Next.js app: resolve the tenant from the Host header in proxy.ts (middleware.ts before Next.js 16), reject unknown hosts, build absolute URLs, handle auth and cookies across domains, cache tenant lookups, and test locally."
date: 2026-10-20
updated: 2026-10-20
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [next.js custom domains, next.js multi tenant, nextjs host header tenant, next.js middleware custom domain, proxy.ts next 16, multi-tenant saas next.js, vercel platforms alternative]
faq:
  - q: "How do I support custom domains in a Next.js app?"
    a: "Point each customer domain at your app (directly or through a proxy that terminates TLS), read the Host header in proxy.ts (middleware.ts before Next.js 16) or in Server Components, look up the tenant by hostname, and rewrite or render that tenant's content. Return 421 or 404 for hosts you don't recognise."
  - q: "Is it middleware.ts or proxy.ts in Next.js 16?"
    a: "Next.js 16 renamed the middleware file to proxy.ts with an exported proxy function. Older versions use middleware.ts with an exported middleware function. Both run before routes and can read the Host header."
  - q: "How do I get the hostname in a Next.js Server Component?"
    a: "Call headers() from next/headers and read the host header, then strip any port. Look the tenant up by that hostname and call notFound() if there isn't one."
  - q: "How do logins work on customer custom domains?"
    a: "Cookies are scoped per domain, so a session on app.yourapp.com does not exist on careers.acme.com. Either run auth on each custom domain (its own session cookie), or hand off from your main domain with a short-lived one-time token exchanged for a session on the custom domain."
  - q: "Do I need Vercel to do multi-tenant custom domains in Next.js?"
    a: "No. Vercel Platforms is one option. Any host works if something terminates TLS for customer domains and forwards the original Host header to your Next.js app, such as Caddy, a load balancer, or a custom-domains API."
---

# Custom domains in a multi-tenant Next.js app: routing by Host header (Next.js 16)

**Short answer:** serve every customer domain from one Next.js app by reading the **`Host` header**. In `proxy.ts` (`middleware.ts` before Next.js 16), reject hosts you don't know, and rewrite known ones to a tenant route if your app needs it. In Server Components, call `headers()` to get the host and look up the tenant. Build every absolute URL from the request host, scope auth per domain, cache the host→tenant lookup, and make sure whatever terminates TLS forwards the original `Host`.

*Last updated 20 October 2026 · Vijay Tambe, founder of Apexly*

---

## The architecture in one picture

```
visitor ── https://careers.acme.com ──▶ TLS terminator ──▶ your Next.js app
                                        (Caddy, a CDN,       reads Host: careers.acme.com
                                         or a custom-        → tenant "acme"
                                         domains API)        → renders Acme's careers page
```

Next.js doesn't issue certificates or verify DNS. Something in front of it does (see the [complete custom domains guide](/blog/custom-domains-for-saas)). Next.js's job is narrower: **turn a hostname into a tenant, safely.**

## Step 1: resolve the tenant from the Host header

```ts
// lib/tenants.ts
import { cache } from "react";
import { db } from "./db";

export const normalizeHost = (h: string) => h.replace(/:\d+$/, "").toLowerCase();

// Deduped per request by React's cache(); add a short TTL cache (Redis/LRU) for hot paths.
export const tenantByHost = cache(async (host: string) =>
  db.tenant.findFirst({ where: { domains: { some: { hostname: normalizeHost(host), active: true } } } }),
);
```

```tsx
// app/page.tsx
import { headers } from "next/headers";
import { notFound } from "next/navigation";
import { tenantByHost } from "@/lib/tenants";

export default async function Home() {
  const host = (await headers()).get("host") ?? "";
  const tenant = await tenantByHost(host);
  if (!tenant) notFound();
  return <h1>Careers at {tenant.name}</h1>;
}
```

Store hostnames lowercase, without a trailing dot, in punycode (`xn--…`) for internationalised domains, and index the column.

## Step 2: guard and route in proxy.ts

Next.js 16 calls it `proxy.ts` with a `proxy` export; earlier versions use `middleware.ts` with `middleware`. Same idea.

```ts
// proxy.ts
import { NextResponse, type NextRequest } from "next/server";

const OWN_HOSTS = new Set(["app.yourapp.com", "yourapp.com", "localhost:3000"]);

export function proxy(req: NextRequest) {
  const host = req.headers.get("host") ?? "";

  // Your own marketing site and dashboard.
  if (OWN_HOSTS.has(host)) return NextResponse.next();

  // Everything else is a customer domain: send it to the tenant-facing routes.
  const url = req.nextUrl.clone();
  url.pathname = `/_sites${url.pathname}`;           // app/_sites/[...path]/page.tsx
  return NextResponse.rewrite(url);
}

export const config = { matcher: "/((?!_next/static|_next/image|favicon.ico).*)" };
```

Keep database calls **out** of `proxy.ts` where you can. It runs on every request, including assets. Rewrite by host shape here, and do the tenant lookup in the page or layout, where React's `cache()` dedupes it.

## Step 3: refuse hosts you don't serve

Anyone can point a domain at your IP or send any `Host` header. Never fall back to a default tenant:

- In pages, `notFound()` when `tenantByHost` returns nothing.
- If a proxy terminates TLS in front of you, **only accept requests that came through it.** Restrict by private network, or verify a signature the proxy adds. For example, with Apexly in front, each request carries `X-Apexly-Timestamp` and `X-Apexly-Signature` (HMAC-SHA256 of `timestamp.host` with your signing secret):

```ts
// lib/apexly.ts  (same helper as the Apexly Next.js guide)
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyApexlyOrigin(host: string, ts?: string | null, sig?: string | null,
  secret = process.env.APEXLY_SIGNING_SECRET!, now = Math.floor(Date.now() / 1000)) {
  if (!ts || !sig || !/^\d{1,12}$/.test(ts) || Math.abs(now - Number(ts)) > 300) return false;
  const name = host.replace(/:\d+$/, "").toLowerCase();
  const expected = createHmac("sha256", secret).update(`${ts}.${name}`).digest();
  const got = Buffer.from(sig, "hex");
  return got.length === expected.length && timingSafeEqual(got, expected);
}
```

Call it in `proxy.ts` for non-own hosts and return `421 Misdirected Request` when it fails.

## Step 4: absolute URLs, redirects and metadata

Anything built from a fixed `NEXT_PUBLIC_BASE_URL` will send customers back to your domain mid-flow. Build from the request instead:

```ts
const h = await headers();
const origin = `${h.get("x-forwarded-proto") ?? "https"}://${h.get("host")}`;
```

Use it for `metadataBase`, canonical URLs, Open Graph images, sitemaps, password-reset and magic-link emails, and OAuth `redirect_uri`s. Each provider needs every custom domain registered as a callback, so most teams run OAuth on their main domain and hand off (next step).

## Step 5: auth and cookies across domains

Cookies are per domain. A session on `app.yourapp.com` is invisible on `careers.acme.com`, and you can't set a cookie for someone else's domain. Two patterns work:

1. **Separate sessions per domain.** The custom domain has its own login, and sets its own host-only cookie (`Secure; HttpOnly; SameSite=Lax`, no `Domain` attribute).
2. **Token handoff.** Sign in on your main domain, then redirect to `https://careers.acme.com/auth/callback?t=<one-time token>`. The custom domain exchanges the token server-side (single use, 60-second expiry, bound to that host) for its own session cookie.

Server Actions compare `Origin` with `Host`. Keep both exactly as the visitor sent them (don't rewrite `Host` in your proxy) and they work on custom domains unchanged.

## Step 6: caching and ISR per tenant

If you statically render or cache tenant pages, **the cache key must include the host**, or Acme's page gets served on Globex's domain. With `fetch` caching or `unstable_cache`/`"use cache"`, pass the hostname as part of the key or tag (`tenant:${host}`), and revalidate by tag when a tenant edits content.

## Step 7: test locally

`*.localhost` resolves to 127.0.0.1 in modern browsers, so `acme.localhost:3000` works with no hosts-file edits. For real DNS-style names, use `*.localtest.me` (public DNS that resolves to 127.0.0.1). Seed a tenant per test hostname, and test the unknown-host path too.

## Checklist

- [ ] Tenant lookup by normalised hostname, indexed and cached
- [ ] Unknown hosts get 404/421, never a default tenant
- [ ] Only requests from your TLS proxy are trusted (network or signature)
- [ ] Absolute URLs, emails and OAuth built from the request host
- [ ] Auth designed per domain (separate sessions or token handoff)
- [ ] Cache keys include the hostname
- [ ] Local tests on `*.localhost` including an unknown host

## Where the domains come from

Everything above assumes customer domains already reach your app with valid HTTPS. That part (DNS instructions, verification, certificates, renewals, apex domains) is the bigger project. Apexly does it behind one API call per domain and forwards requests with the original `Host` and a signature, so your Next.js app only needs Steps 1–7. [Next.js guide](https://docs.apexly.dev/guides/nextjs).

## FAQ

**How do I support custom domains in Next.js?**
Get customer domains to your app with TLS terminated in front, read `Host` in `proxy.ts` or Server Components, look up the tenant by hostname, and 404 unknown hosts.

**Is it `middleware.ts` or `proxy.ts`?**
`proxy.ts` (exporting `proxy`) in Next.js 16; `middleware.ts` (exporting `middleware`) before that.

**How do I get the hostname in a Server Component?**
`(await headers()).get("host")` from `next/headers`, with the port stripped.

**How do logins work on custom domains?**
Cookies are per domain: run a session per custom domain, or hand off from your main domain with a short-lived one-time token.

**Do I need Vercel for this?**
No. Any setup works if something terminates TLS for customer domains and forwards the original Host header to Next.js.

## Sources

- [Next.js: proxy.ts (formerly middleware)](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)
- [Next.js: headers()](https://nextjs.org/docs/app/api-reference/functions/headers)
- [Vercel: Multi-tenant platforms, middleware and routing](https://vercel.com/docs/multi-tenant-platforms/concepts)
- [RFC 9110 §15.5.20: 421 Misdirected Request](https://www.rfc-editor.org/rfc/rfc9110#section-15.5.20)
- [Apexly Next.js guide](https://docs.apexly.dev/guides/nextjs)
