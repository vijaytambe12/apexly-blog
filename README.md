# Publishing a post to apexly.dev/blog

For the Apexly Editor bot. Copy this file to `~/gtm/blog/PUBLISHING.md` on the bot computer. It is also the README of the content repo.

Posts live in the public GitHub repo **`vijaytambe12/apexly-blog`**, one file per post at `posts/<slug>.md`. apexly.dev reads that repo every couple of minutes. A commit to `main` is the whole publish: no build or deploy. Within about 5 minutes, the post appears on `/blog`, at `/blog/<slug>` and `/blog/<slug>.md`, and in `/blog/rss.xml`, `/sitemap.xml`, `/llms.txt` and `/llms-full.txt`.

## One-time setup on the bot computer

Vijay gives you a fine-grained GitHub token that can only write to this one repo. Store it, then clone:

```bash
mkdir -p ~/.config/apexly && chmod 700 ~/.config/apexly
printf 'https://x-access-token:%s@github.com\n' 'PASTE_TOKEN_HERE' > ~/.config/apexly/git-credentials
chmod 600 ~/.config/apexly/git-credentials
git clone -c credential.helper="store --file $HOME/.config/apexly/git-credentials" https://github.com/vijaytambe12/apexly-blog.git ~/gtm/blog/apexly-blog
cd ~/gtm/blog/apexly-blog && git config user.name "Apexly Editor" && git config user.email "support@apexly.dev"
```

Never print the token, paste it into a post, or commit it.

## The post file

Path: `posts/<slug>.md`. The file name must be exactly the `slug` plus `.md`. Copy the shape of an existing post in `posts/`:

```markdown
---
title: "How to verify a customer domain with a TXT record (2026)"
slug: verify-domain-txt-record
description: "One or two sentences, under 400 characters. This is the search snippet and the RSS summary."
date: 2026-10-27
updated: 2026-10-27
author: Vijay Tambe
authorRole: Founder, Apexly
keywords: [domain verification txt record, verify custom domain, saas domain verification]
faq:
  - q: "Question one?"
    a: "A complete answer in one or two sentences."
  - q: "Question two?"
    a: "Another complete answer."
---

# How to verify a customer domain with a TXT record (2026)

**Short answer:** the answer in 2 to 4 sentences, first, for answer engines.

*Last updated 27 October 2026 · Vijay Tambe, founder of Apexly*

## First section
...

## FAQ

### Question one?

The same answer as in the front matter.
```

Rules. A post that breaks one is **not published** (it is skipped, the rest of the blog is unaffected):

- `title`, `slug`, `description`, `date`, `updated`, `author`, `authorRole` are all required. Title up to 200 characters, description up to 400.
- `slug`: lowercase letters, digits and single hyphens (`verify-domain-txt-record`), and it matches the file name. Never change a published slug: it is the URL.
- `date` and `updated` are `YYYY-MM-DD`, and `updated` is not before `date`.
- Wrap any value that contains a colon, `#`, quotes or starts with `[` in double quotes, as above. Use `\"` for a quote inside.
- `keywords` is one line: `[a, b, c]`.
- `faq` items are exactly `  - q: "…"` then `    a: "…"` (2 and 4 spaces). Every item needs both. Use at least 3: they become the FAQPage rich result.
- Nothing else in the front matter except these keys, one per line.
- Body is Markdown (GitHub style: tables, task lists, fenced code with `ts`, `tsx` or `caddyfile` highlighted). Start with the `# Title` line and the `**Short answer:**` paragraph; the page renders the h1 and byline itself.
- Raw HTML is shown as plain text, not rendered. Links must be `https://…`, `/path`, `#anchor` or `mailto:`; anything else renders as plain text. No images (the site only loads images it hosts).
- Link to the pillar guide `/blog/custom-domains-for-saas` and at least one other post.

## Publish

```bash
cd ~/gtm/blog/apexly-blog
git pull --rebase
# write posts/<slug>.md
git add posts/<slug>.md
git commit -m "post: <slug>"
git push
```

To edit a published post: change the file, set `updated` to today, commit `edit: <slug>`, push. Do not delete or rename posts unless Vijay asks.

## Check it is live

The site checks GitHub every 2 minutes. Poll the status endpoint every 30 seconds, for up to 10 minutes:

```bash
curl -fsS https://apexly.dev/blog/status.json
# {"posts":["verify-domain-txt-record", ...],"skipped":[{"file":"bad.md","error":"slug does not match file name"}]}
```

- Your slug is in `posts`: it is live. Run the checks below.
- Your file is in `skipped`: the `error` says what is wrong. Fix the file, push again, and poll again.
- Neither, after 10 minutes: see "When something fails".

Then confirm every surface (each command must print a line):

```bash
S=<slug>
curl -fsS https://apexly.dev/blog/$S | grep -c "<h1"
curl -fsS https://apexly.dev/blog/$S.md | grep -m1 "slug: $S"
curl -fsS https://apexly.dev/blog | grep -c "/blog/$S\""
curl -fsS https://apexly.dev/blog/rss.xml | grep -m1 "/blog/$S<"
curl -fsS https://apexly.dev/sitemap.xml | grep -m1 "/blog/$S<"
curl -fsS https://apexly.dev/llms-full.txt | grep -m1 "URL: https://apexly.dev/blog/$S "
```

A surface you checked early can lag a minute or two behind the others: wait a minute and check that one again.

## Tell search engines (IndexNow)

Once the post is live, ping IndexNow once (Bing, Yandex and others; ChatGPT search uses Bing):

```bash
S=<slug>
curl -sS -o /dev/null -w "%{http_code}\n" -X POST https://api.indexnow.org/indexnow \
  -H "Content-Type: application/json; charset=utf-8" \
  -d "{\"host\":\"apexly.dev\",\"key\":\"4450cb457c3a3e62ca381a551c445851\",\"keyLocation\":\"https://apexly.dev/4450cb457c3a3e62ca381a551c445851.txt\",\"urlList\":[\"https://apexly.dev/blog/$S\",\"https://apexly.dev/blog\"]}"
```

`200` or `202` is success. Anything else: retry once in an hour, then note it in your report. Don't retry in a loop.

## When something fails

| What you see | What to do |
|---|---|
| `git push` rejected (non-fast-forward) | `git pull --rebase`, then `git push` again. |
| `git push` says 401, 403 or "Authentication failed" | The token expired or was revoked. Stop and tell Vijay. Don't try other credentials. |
| Your file is in `skipped` | Fix what `error` says, commit `fix: <slug>`, push, poll again. |
| After 10 minutes, the slug is in neither `posts` nor `skipped` | Check the commit is on GitHub: `git log origin/main -1`. If it is there, the site can't reach GitHub and is serving the posts it already had. Nothing is broken. Tell Vijay, and don't push again. |
| `status.json` or `/blog` returns a 5xx or doesn't load | The site itself is down. That is not caused by a post. Tell Vijay. |
| A live post has a mistake | Edit it as above. The fix is live within about 5 minutes. |
