# Weekly website health check: broken links and expiring SSL certificates

Broken links waste crawl budget and frustrate users. An expired SSL certificate takes a site offline in every browser. Both are easy to miss when you manage more than one domain.

This tutorial sets up a lightweight weekly health check with two Digital-Tools Actors:

1. **[Broken Link Checker](https://apify.com/Digital-Tools/broken-link-checker)** – crawl a site and find 404s and dead links
2. **[SSL Certificate & DNS Checker](https://apify.com/Digital-Tools/ssl-dns-checker)** – bulk-check certificate expiry, plus DNS, SPF, and DMARC

You can then add **[URL Status & Redirect Chain Checker](https://apify.com/Digital-Tools/url-status-redirect-checker)** if you already have a fixed list of important URLs (homepage, checkout, docs) and only need status codes and redirect hops — no crawl.

## 1. Find broken links

Start the Broken Link Checker at the homepage. Cap the crawl and the number of links so the cost stays predictable.

```json
{
  "startUrls": [{ "url": "https://www.example.com" }],
  "maxPagesToCrawl": 100,
  "maxLinksToCheck": 2000,
  "checkExternalLinks": true,
  "onlyBrokenLinks": true
}
```

With `onlyBrokenLinks` on, the dataset is just the problems. Each row tells you the dead URL, its HTTP status, the anchor text, and which pages link to it:

```json
{
  "url": "https://www.example.com/old-page",
  "status": "broken",
  "statusCode": 404,
  "anchorText": "Old page",
  "foundOnPages": ["https://www.example.com/", "https://www.example.com/blog"],
  "ok": false
}
```

Read `status` carefully: `blocked-or-auth` (401/403/429/999) is usually bot protection or a login, not a dead page. `unreachable` means a timeout or DNS failure. `broken` is the one you almost always want to fix.

In the Apify Console, open the Actor, click **Schedule**, and run it every Monday. Connect a Slack, email, or webhook integration so you only hear about it when the dataset is not empty.

## 2. Check SSL expiry and DNS in bulk

Paste every domain you operate (or every client domain, if you are an agency) into the SSL Certificate & DNS Checker.

```json
{
  "domains": ["example.com", "www.example.com", "api.example.com"],
  "checkSsl": true,
  "checkDns": true,
  "expiryWarningDays": 30
}
```

Each domain comes back with a single `status`:

- `ok` – certificate is valid and not close to expiry
- `expiring-soon` – expires inside your warning window (30 days here)
- `expired`, `invalid`, or `no-ssl` – fix these first

You also get `daysUntilExpiry`, `issuerOrg`, `tlsVersion`, DNS records, and parsed `spf` / `dmarc` strings. A missing SPF record or `p=none` DMARC policy is worth a follow-up even when the certificate is fine.

Schedule this one daily or weekly. Filter the dataset on `status != "ok"` before you send an alert, or you will be notified about healthy domains too.

## 3. Optional: check a watchlist of URLs

If you do not need a full crawl — just “are these 50 URLs still 200, and do they still redirect to HTTPS?” — run the URL Status & Redirect Chain Checker on that list.

```json
{
  "startUrls": [
    { "url": "http://example.com" },
    { "url": "https://example.com/login" }
  ],
  "maxRedirects": 10
}
```

Look at `ok`, `finalUrl`, `hopCount`, and `schemeChanged`. A homepage that still starts on `http://` and never lands on `https://` is a common, easy fix.

## 4. What it costs (approximate)

All three Actors charge per successful result, plus a $0.0005 start fee per run.

| Check | Volume | Approx. cost |
|---|---|---|
| Broken Link Checker | 1,000 unique links | about $0.50 |
| SSL & DNS Checker | 50 domains | about $0.05 |
| URL status & redirects | 50 URLs | about $0.03 |

A weekly run for a typical marketing site is usually well under a dollar. Apify's free plan includes $5 of monthly credit.

## Tips

- Start Broken Link Checker with `maxPagesToCrawl: 1` if you only want to validate links on a few known pages.
- After a site migration, run the status/redirect checker on the old URL list and confirm every `finalUrl` is the new canonical page.
- Please only crawl sites you own or are allowed to audit.
