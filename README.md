<p align="center"><a href="https://apify.com/DevilScrapes/medium-user-articles-scraper"><img src="assets/banner.svg" alt="Medium Author RSS Scraper — Python example" width="100%"></a></p>

# Medium User Articles Scraper in Python: Medium author and publication feeds to JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/medium-user-articles-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/medium-user-articles-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/medium-user-articles-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository is a compact, tested Python client for the **[Medium Author RSS Scraper](https://apify.com/DevilScrapes/medium-user-articles-scraper)** on Apify. Give it a list of Medium handles or publication feeds, for instance a company's engineering blog, and it returns their most recent articles as a table: headline, URL, byline, tags, publication time and, if you want it, the HTML body. Medium's feeds are fetched and parsed on Apify, so there is no XML handling in your code and no request pacing to tune. Locally you get `results.json` and `results.csv`, which is what a pandas DataFrame, a Jupyter notebook or a nightly cron archive wants to start from.

> **Actor:** [Medium Author RSS Scraper](https://apify.com/DevilScrapes/medium-user-articles-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [medium-user-articles-scraper-javascript](https://github.com/devilscrapes/medium-user-articles-scraper-javascript)

## 🔥 What you get

- **Accounts and publications in one list**: a handle such as `@netflixtechblog` (the `@` is optional) is turned into its Medium feed address, and a full feed URL like `https://medium.com/feed/netflix-techblog` is fetched as given.
- **Rows that carry the essentials**: headline, article URL, the feed's guid as `article_id`, the byline, the tags the writer picked and the publication time converted to ISO-8601 in UTC.
- **Article HTML on request**: with `includeContent` on, `content_html` holds the body from the feed, or its summary when that is all the feed provides; switch it off and the column is `null`, which keeps rows small.
- **A per-account cap**: `maxArticlesPerUser` (1 to 50) trims each feed, although Medium itself rarely lists more than the ten newest posts.
- **One dead handle does not stop the batch**: a feed that answers with an error is logged and skipped, and the remaining accounts are still delivered.
- **Billed per article written**: a start event plus a fee per row, so a handle whose feed is empty or unreachable costs nothing extra.

## 🎯 Use cases

### Benchmark engineering blogs

Track a dozen company tech blogs and load the rows into a DataFrame. Monthly article counts per account and the most frequent tags show who publishes steadily and which topics, from machine learning to infrastructure, each team writes about.

### Build a reading corpus for analysis

Switch `includeContent` on and the HTML bodies give you text for keyword extraction, topic modelling or summarisation experiments in a notebook. Strip the markup with your parser of choice and keep `link` as the citation.

### Archive a publication over time

Medium's feeds only expose recent posts, so history has to be collected as it happens. Run the script daily from cron, append each `results.csv` to a growing file and deduplicate on the article URL to build the long archive the feed alone never shows.

### Monitor competitors' content calendars

Product marketers can follow rival company accounts and see within a day when a launch post, a case study or a hiring article appears, along with the tags it was filed under.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/medium-user-articles-scraper-python.git
cd medium-user-articles-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Medium Author RSS Scraper](https://apify.com/DevilScrapes/medium-user-articles-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Most of `main.py` deals with arguments, output files and failure reporting. Underneath, the integration is just five lines: `apify-client` launches the Actor, waits for the run to end and iterates over every dataset item.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/medium-user-articles-scraper").call(run_input={'usernames': ['@netflixtechblog', '@AirbnbEng'], 'maxArticlesPerUser': 5, 'includeContent': False, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`input.json` goes to the Actor as is. The sample reads two official engineering accounts, Netflix Technology Blog and Airbnb Engineering, keeps five posts from each and leaves the HTML out, so the test run writes at most ten small rows. Add handles or publication feed URLs to `usernames` to watch more sources.

```json
{
  "usernames": [
    "@netflixtechblog",
    "@AirbnbEng"
  ],
  "maxArticlesPerUser": 5,
  "includeContent": false,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `usernames` | array | yes | `["@medium"]` | — |
| `maxArticlesPerUser` | integer | no | `10` | — |
| `includeContent` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Every row is one article. `username` repeats the handle you passed, or the publication slug when you gave a feed URL. Article links come straight from the feed and include a `?source=rss` tracking query, so cut everything after `?` before using them as keys. `published` is `null` if a feed entry carries no readable date. On publication feeds, `author` is the individual writer of that piece rather than the publication.

| Field | Type | Always present |
|---|---|---|
| `username` | string | yes |
| `article_id` | string or null | no |
| `title` | string | yes |
| `link` | string | yes |
| `author` | string | no |
| `content_html` | string or null | no |
| `categories` | array | no |
| `published` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/medium-user-articles-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The pandas recipe below (`pip install pandas`) cleans the links, counts articles per account per month, lists each account's five most used tags in `top_tags.csv` and writes a tidy `articles.csv` sorted newest first.

```python
import json

import pandas as pd

posts = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
posts["published"] = pd.to_datetime(posts["published"], utc=True, errors="coerce")
posts["url"] = posts["link"].str.split("?").str[0]  # drop the feed's tracking query
posts = posts.drop_duplicates("url")

cadence = (
    posts.dropna(subset=["published"])
    .assign(month=lambda df: df["published"].dt.strftime("%Y-%m"))
    .groupby(["username", "month"])
    .size()
    .unstack(fill_value=0)
)
print(cadence.to_string())

tags = posts.explode("categories").dropna(subset=["categories"])
top_tags = tags.groupby("username")["categories"].value_counts().groupby(level=0).head(5)
top_tags.rename("articles").to_csv("top_tags.csv")
print(top_tags.to_string())

latest = posts.sort_values("published", ascending=False)
latest[["username", "published", "title", "url"]].to_csv("articles.csv", index=False)
```

## ❓ FAQ

### Do I need a Medium account, an integration token or a paid membership?

No. Only an Apify API token. The Actor reads the RSS feeds Medium publishes for every public account and publication, which do not require signing in.

### Why do I get only about ten articles per account?

Each Medium feed lists the newest posts only, typically ten. Raising `maxArticlesPerUser` cannot go past what the feed contains. To keep older items, collect on a schedule and store every run's output, as the archive use case above describes.

### How do I add a publication rather than a person's account?

Pass its feed URL, `https://medium.com/feed/<publication-slug>`. A bare handle is always expanded to `medium.com/feed/@handle`, which is the format for accounts. Profile page URLs are fetched as they are and are not feeds, so use a handle or a feed address instead.

### Does it return claps, responses or paywalled text?

No. Those numbers are not part of the feed, and member-only stories are delivered only as far as the feed itself includes them. The Actor reads the feed and nothing else.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/medium-user-articles-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Medium Author RSS Scraper on Apify Store](https://apify.com/DevilScrapes/medium-user-articles-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/medium-user-articles-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [DEV.to Scraper — Articles by Tag & Author](https://apify.com/DevilScrapes/dev-to-articles-scraper) — the same kind of article feed from DEV.to, by tag or author.
- [Substack Newsletter Scraper](https://apify.com/DevilScrapes/substack-newsletter-scraper) — full Substack archives with reaction and word counts.
- [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper) — see which of those posts reached Hacker News and how they scored.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/medium-user-articles-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Medium or A Medium Corporation.
