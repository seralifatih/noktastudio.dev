---
title: "How to Get Congressional Trading Data as JSON"
description: "Members of Congress publish every stock trade over $1,000. It arrives as PDFs and inconsistent web forms. Here's every way to get it as structured JSON, including the free ones."
date: 2026-07-31
author: Fatih Ilhan
---

# How to Get Congressional Trading Data as JSON

Every member of Congress has to disclose stock trades over $1,000 within 45 days. All 535 of them. The data is completely public, has been since 2012, and almost nobody uses the raw source.

The reason is not access. It's that the House publishes scanned PDFs and the Senate publishes a different web form, and the two chambers don't agree on a single field.

This post covers every way I know to get that data as usable JSON — the free ones, the paid ones, and the one I built. If you just want the JSON, skip to the last section.

---

## What the Data Actually Is

The Stop Trading on Congressional Knowledge Act of 2012 requires members of Congress, their spouses, and dependent children to publicly disclose any transaction over $1,000 within 45 days. The form is called a Periodic Transaction Report — a PTR.

A single PTR can hold one trade or forty. Each row gives you:

- Asset name and (sometimes) ticker
- Transaction type: purchase, sale, or exchange
- Transaction date
- Filing date
- An **amount range**, not an amount — `$1,001 - $15,000`, `$15,001 - $50,000`, and so on up

Two things people get wrong on first contact with this data:

**The 45-day lag is real.** A trade you see filed today may have happened six weeks ago. Any backtest that treats filing date as trade date is measuring something other than what it thinks.

**There are no dollar amounts.** Only buckets. If a tool hands you a clean `amount: 8000`, someone took the midpoint of a range and didn't tell you.

---

## Where It Comes From

Two sources, two completely different problems.

**House** — the Clerk of the House publishes a fresh ZIP every day containing every disclosure filed that year, plus an XML index listing them. You filter the index to `FilingType=P` for periodic transaction reports, then fetch each filing's PDF by document ID. Plain HTTPS, no proxy, no terms gate. Recent PTRs are machine-generated so the text extracts cleanly; older paper filings are scans of a form somebody filled in with a pen.

**Senate** — filings go through the Senate's electronic financial disclosure system, which exposes a paginated search index. You get JSON, with HTML as a fallback when the structured fields come back empty. Better source, completely different code.

So you are not building one scraper. You are building a PDF text extractor with pattern matching for one chamber, an HTML parser for the other, and a normalization layer to make the two agree.

---

## The Free Options

**Roll your own.** Both portals are publicly accessible. Pull the filing index, download each PTR, run `pdftotext` on the House PDFs, parse the Senate HTML, pattern-match the transaction rows.

This works. I know because it's what I do. Budget for the following, all of which I hit:

- Scanned filings with no extractable text. You either OCR them or drop them. Dropping them is honest; silently dropping them is not.
- Ticker formats that vary by filer and by year — some rows have a clean symbol, some have an asset name and nothing else, some have the ticker buried in parentheses inside the name.
- Amended filings. A member files a PTR, then files a correction. Both are in the index. Naive ingestion double-counts the trade.
- Repeated entries inside a single filing that are genuinely two separate trades of the same asset on the same day, which look identical to a deduplicator and aren't.

**Existing open datasets.** There are community-maintained repositories of parsed congressional trades. They're free and they're fine for a hobby project. The catch is staleness and coverage — you inherit whatever gaps the maintainer's parser has, on whatever schedule they run it.

---

## The Paid Options

There are several commercial products in this space — screener-style web apps with an API bolted on, and a handful of raw data pipelines. They differ mainly on three axes:

1. **Do you get raw normalized rows, or their interpretation?** Some products give you sentiment scores and "top congressional buys" leaderboards. That's a product decision, not data.
2. **Do they preserve the amount ranges or flatten them?** Flattening to a midpoint is a guess presented as a number.
3. **Subscription or per-use?** Most are monthly. If you need one historical backfill and then a weekly delta, monthly pricing is a bad fit for the shape of your usage.

I'd rather you evaluate those yourself against your use case than take my ranking of my competitors on faith.

---

## What I Built

I run two pipelines on Apify — one for the House, one for the Senate — that output the same schema.

The design decisions, so you can tell whether they match what you need:

**One schema across both chambers.** The whole point. House PDF rows and Senate HTML rows land in the same shape, so you write one consumer instead of two.

**Amount ranges preserved, not flattened.** You get the bucket the filer actually disclosed. If you want a midpoint, you can take one — but that's your assumption to own, not mine to bake in.

**Deduplication across amended and repeated filings.** Amendments resolve against the original rather than stacking.

**Billed per result, no subscription.** You pay for the rows you pull. House is $2 per 1,000 results, Senate is $3 per 1,000. A one-time historical backfill costs what it costs and then stops costing.

One row per individual transaction. Real output:

```json
{
  "id": "4d6016b44239f646476ffac6798f21ae3e32c8ed75ea6c5b50a0bbdf9e5d3296",
  "politician": "Mark Alford",
  "transaction_date": "2026-03-16",
  "filing_date": "2026-03-31",
  "ticker": "AMZN",
  "asset_name": "Amazon.com, Inc. - Common Stock",
  "asset_type": "Stock",
  "type": "sell",
  "amount_min": 1001,
  "amount_max": 15000,
  "owner": "self"
}
```

`id` is a SHA-256 of `politician|date|asset|amount_min|amount_max` — a stable dedup key, so re-running over an overlapping window doesn't duplicate rows. `ticker` is `null` for bonds, municipals, and structured notes rather than guessed at. `amount_max` is `null` for unbounded "Over $X" disclosures. `owner` distinguishes self, joint, spouse, and child per the STOCK Act categories, which matters more than people expect — a spouse's trade is not the member's trade.

Pulling it:

```bash
curl -X POST "https://api.apify.com/v2/acts/seralifatih~congress-trading-pipeline-1/runs?token=YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "fetchDaysBack": 30 }'

curl "https://api.apify.com/v2/datasets/<dataset-id>/items?token=YOUR_TOKEN&format=json"
```

The parsers are public: [github.com/seralifatih](https://github.com/seralifatih). MIT. If you'd rather read the extraction logic than trust it, you can.

---

## What I Can't Claim

Being straight about the limits, because every data product in this category has them and most don't publish them.

**About 5% of historical House PTRs are unparseable.** Older filings submitted on paper come back as scanned images with no text layer. The parser logs them as unparseable and moves on. It does not silently drop them, but "logged" still means the transactions aren't in your dataset. OCR fallback is on the roadmap and isn't shipped.

**Ticker resolution is imperfect.** When a filing gives an asset name and no symbol, matching it to a ticker is inference. I flag these rather than guessing silently, but "flagged" still means "you have work to do."

**The 45-day lag is not something any vendor can fix.** Anyone selling you "real-time congressional trades" is selling you real-time access to disclosures that are up to 45 days old. That's a limit of the law, not of the pipeline.

**I'm one person.** If a chamber changes its filing format on a Tuesday, my fix lands in days, not hours.

---

## Which One You Want

- **Learning project, no deadline** → roll your own. The parsing problem is genuinely interesting and you'll understand the data better than any API will teach you.
- **You need a dashboard and don't want to build one** → use one of the screener web apps.
- **You need normalized rows in your own pipeline** → that's what mine is for.

If you're building a screener, an alerting system, or a backtest and you want to skip the PDF layer entirely:

**[House Trading Pipeline →](https://apify.com/seralifatih)** · **[Senate Trading Pipeline →](https://apify.com/seralifatih)**

Both output the schema above. Per result, no subscription, no minimum.

If you're parsing this data yourself and hit an edge case I didn't cover, I'm interested — I've probably hit it too.
