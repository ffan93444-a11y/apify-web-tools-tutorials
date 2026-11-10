# Turn a documentation site into Markdown for an LLM knowledge base

If you want an LLM or RAG pipeline to answer questions from a public docs site, help center, or blog, you first need clean text. Raw HTML is noisy. This walkthrough uses two Digital-Tools Actors on the Apify Store:

1. **[Sitemap URL Extractor](https://apify.com/Digital-Tools/sitemap-url-extractor)** – list every page in `sitemap.xml`
2. **[Webpage to Markdown](https://apify.com/Digital-Tools/webpage-to-markdown)** – convert each URL to LLM-ready Markdown

Optional: if some URLs are PDFs (manuals, whitepapers), run them through **[Document to Text](https://apify.com/Digital-Tools/document-to-text)** instead.

You can do the same steps in the Apify Console, via the API, or from an automation tool such as Make, Zapier, or n8n.

## 1. List the pages you care about

Start the Sitemap URL Extractor with the site homepage (or a direct sitemap URL). For a documentation site you often only want `/docs/` or `/blog/` URLs, so add an include pattern.

```json
{
  "startUrls": [{ "url": "https://docs.apify.com" }],
  "maxUrlsPerSite": 2000,
  "includePattern": "/platform/"
}
```

The dataset is one row per URL, with `lastmod`, `priority`, and the sitemap it came from. Download it as JSON or CSV, or pass the URL list straight into the next Actor.

If the site has no sitemap, collect start URLs by hand or from a crawl, and skip this step.

## 2. Convert each page to Markdown

Run Webpage to Markdown on that URL list. Keep **Main content only** on so navigation and cookie banners are dropped. Set `maxCharacters` if you need to fit a context window.

```json
{
  "startUrls": [
    { "url": "https://docs.apify.com/platform/actors" }
  ],
  "mainContentOnly": true,
  "includeLinks": true,
  "maxCharacters": 20000
}
```

Each result looks like this:

```json
{
  "url": "https://docs.apify.com/platform/actors",
  "title": "Actors | Platform | Apify Documentation",
  "markdown": "# Actors\n\nActors are serverless cloud programs that ...",
  "wordCount": 260,
  "truncated": false,
  "error": null
}
```

Failed URLs are returned with an `error` and are not charged.

## 3. Chunk, embed, and store

From here the work is in your stack, not on Apify:

- Split `markdown` on headings (`##`) or into overlapping token windows.
- Keep `url` and `title` as metadata so the model can cite the source.
- Embed the chunks and upsert them into your vector store (Pinecone, Chroma, pgvector, and so on).

A practical chunk is 500–1,000 tokens with a short header, for example:

```text
Source: Actors | Platform | Apify Documentation
URL: https://docs.apify.com/platform/actors

# Actors
...
```

## 4. What it costs (approximate)

Both Actors use pay-per-event pricing; failed items are free. Plus a $0.0005 start fee per run.

| Step | Typical volume | Approx. cost |
|---|---|---|
| Sitemap URL Extractor | 1,000 URLs | about $0.20 |
| Webpage to Markdown | 1,000 pages | about $1.00 |

A mid-size docs site of a few hundred pages is well under a dollar to ingest. Apify's free plan includes $5 of monthly credit, which is enough to try the whole flow.

## Tips

- Schedule a weekly sitemap run and convert only URLs whose `lastmod` is new, so you refresh the knowledge base without re-embedding everything.
- JavaScript-only pages may come back thin or empty; static docs, blogs, and Wikipedia generally work well.
- For PDFs linked from the same site, collect those URLs and run Document to Text with **Page-by-page output** on, then chunk by page for citations.
- Only convert sites you are allowed to access, and respect each site's terms and crawl budget.

## Related Actors

- [URL Metadata Extractor](https://apify.com/Digital-Tools/url-metadata-extractor) – grab title, description, and Open Graph tags if you want a preview card per page
