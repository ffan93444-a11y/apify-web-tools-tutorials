# Build a daily news brief: RSS to JSON, then Markdown for an AI summary

A useful daily brief is just “what was published since yesterday, in clean text, ready for a model to summarize.” You do not need a custom crawler for that. This tutorial chains two Actors:

1. **[RSS to JSON Feed Reader](https://apify.com/Digital-Tools/rss-feed-reader)** – pull new items from RSS/Atom feeds
2. **[Webpage to Markdown](https://apify.com/Digital-Tools/webpage-to-markdown)** – fetch the full article as clean Markdown when the feed only has a teaser

You can then send the Markdown to your own LLM, or to an automation tool, and post the summary to Slack, email, or a Notion page.

## 1. Collect new feed items

The Feed Reader accepts either a direct feed URL or a website URL (it will try to discover `/feed`, `/rss.xml`, or a `<link rel="alternate">` tag). Use **Only items published after** so a scheduled run bills you only for new items.

```json
{
  "startUrls": [
    { "url": "https://feeds.bbci.co.uk/news/rss.xml" },
    { "url": "https://github.blog/" },
    { "url": "https://blog.python.org" }
  ],
  "maxItemsPerFeed": 20,
  "publishedAfter": "2026-11-10",
  "includeContent": true
}
```

On a daily schedule, set `publishedAfter` to yesterday's date (most automation tools can insert that dynamically). Each item is a flat JSON record:

```json
{
  "feedTitle": "Python Insider",
  "title": "Python 3.15.2 is out",
  "link": "https://blog.python.org/2026/10/python-3152/",
  "published": "2026-10-25T16:00:00+00:00",
  "author": "Hugo van Kemenade",
  "summary": "Security releases ...",
  "contentText": "..."
}
```

If `contentText` is already a full article, you can skip the next step for that item.

## 2. Fetch full article text where the feed is only a teaser

Many news feeds include a title and a one-paragraph summary, not the article. Take the `link` values and run Webpage to Markdown:

```json
{
  "startUrls": [
    { "url": "https://blog.python.org/2026/10/python-3152/" }
  ],
  "mainContentOnly": true,
  "includeLinks": false,
  "maxCharacters": 15000
}
```

You get `title`, `markdown`, and `wordCount` back. That Markdown is what you send to the model.

For PDFs linked from a feed (research, filings, reports), use **[Document to Text](https://apify.com/Digital-Tools/document-to-text)** instead, with **Page-by-page output** if you need citations.

## 3. Turn the batch into a brief

This part runs in your own workflow (an Apify integration, a small script, or an AI agent via the Apify MCP server). A simple prompt is enough:

```text
You are writing a daily briefing. For each article below, write 3 bullet points
and one "why it matters" sentence. Group items by feed. Do not invent facts
that are not in the text.

---
Source: Python Insider
Title: Python 3.15.2 is out
URL: https://blog.python.org/2026/10/python-3152/
Date: 2026-10-25

# Markdown article text
...
```

Keep the source URL next to every summary so a reader can open the original.

## 4. What it costs (approximate)

| Step | Volume | Approx. cost |
|---|---|---|
| RSS to JSON | 200 new items / day | about $0.10 |
| Webpage to Markdown | 50 full articles / day | about $0.05 |

A weekday brief from a dozen blogs is typically well under $0.20 a day, plus the $0.0005 start fee per run. Failed feeds and failed pages are not charged. Apify's free plan includes $5 of monthly credit.

## Tips

- Prefer official or first-party feed URLs over scraped homepages; they are faster, cheaper, and more stable.
- If a site has no feed, the run summary will say so. Don't hammer the homepage — add a different source.
- To attach a preview image or canonical URL to each item, run the links through [URL Metadata Extractor](https://apify.com/Digital-Tools/url-metadata-extractor) first.
- Respect feed providers' terms and don't republish full article text if you only have a license to summarize.
