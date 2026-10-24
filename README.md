# linkedin jobs scraper python: Open-Source Libraries, the Login Catch Nobody Mentions, and When an API Beats Self-Hosting

Most people searching for a LinkedIn jobs scraper in Python want the same thing: a script that pulls job title, company, location, and description from LinkedIn without getting blocked after page two. You have two realistic routes. The first is an open-source library running a headless browser on your own machine — free, but it needs a LinkedIn login and breaks whenever LinkedIn changes its markup. The second is a scraping API like ScraperAPI that handles proxies, CAPTCHAs, and rendering for you, starting at $49/month with a 7-day free trial of 5,000 credits, no credit card required.

This article covers both routes: how the popular Python libraries actually work, the maintenance cost they hide, the working pagination trick for LinkedIn's public jobs endpoint, and the full ScraperAPI pricing table with the credit math that matters specifically for LinkedIn.

> **Quick take:** If you're scraping a few hundred job listings occasionally, start with the open-source route — it costs nothing. If you need recurring or large-scale pulls (a jobs database, market research, a recruiting tool), ScraperAPI's Hobby plan at $49/month covers roughly 3,300 LinkedIn requests, and the 7-day trial with 5,000 free credits is enough to verify it works for your target pages before paying anything. 👉 [Start the free 5,000-credit trial](https://www.scraperapi.com/?fp_ref=coupons)

## The open-source Python libraries, honestly assessed

The library most searches land on is `py-linkedin-jobs-scraper` (installable via `pip install linkedin-jobs-scraper`). It's a Selenium-based headless browser tool, and it's genuinely well built: adaptive rate limiting, session auto-recovery, event hooks for data/metrics/errors, and a CLI that works without writing code.

The catch is right in the setup. You need Chrome or Chromium installed, Python 3.10+, and — this is the part most tutorials skip — **you must authenticate with a LinkedIn account once** via `lijs login`. The tool stores your session in a Chrome profile at `~/.lijs/chrome-profile` and reuses it on later runs. The project's own docs are blunt about the failure mode: the session cookie cannot be renewed, LinkedIn expires it after a while, and a run that loses it just stops. Their advice is to expect to replace it by hand.

Fields it extracts are solid for job data: `job_id`, `title`, `company`, `company_employee_count`, `place`, `description`, `description_html`, `salary`, `is_easy_apply`, `applicant_count`, `benefits`, and `reposted`, among others. Filter flags cover time (`--time day/week/month`), salary bands (`--salary 40k` through `200k`), job type, experience level, workplace type, plus toggles like `--easy-apply` and `--under-10-applicants`.

Other options in the same family:

- **jobpare/linkedIn-scraper** — built on Playwright instead of Selenium, no Docker required, exports to CSV, JSON, or SQLite. Same fundamental model: a real browser session hitting LinkedIn's public pages.
- **Scrapy-based scrapers** (for example the Ruy-Araujo project) — require you to generate your own `li_at` cookie and CSRF token by hand before the script runs at all.

All of these share three traits: they need a logged-in LinkedIn session, they need Chrome running somewhere, and they scrape at whatever pace your IP and LinkedIn's rate limiter tolerate. For a personal research project that's fine. For anything running unattended or at volume, it's the reason people end up looking at APIs.

## How to scrape LinkedIn's public jobs endpoint without a login (the pagination trick)

You don't actually need to log in to get LinkedIn job listings. LinkedIn exposes an internal endpoint for its guest job search, and the scraper logic boils down to paginating through it with the right parameter.

Here's what matters, per ScraperAPI's own Python tutorial:

1. The regular jobs search page (`linkedin.com/jobs/search?keywords=...`) loads results through infinite scroll — the URL never changes, so there's no "next page" link to follow.
2. Open DevTools, watch the Network tab, and you'll see the real data endpoint: `https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search?keywords=Product%20Management&location=...&start=0`.
3. The `start` parameter increments by 25 per page. The `pageNum` parameter does nothing — changing it has no effect, which is where most first attempts stall.
4. The endpoint caps out around `start=975`; at 1,000 you get a 404. So any single search yields at most about 1,000 listings.

With plain `requests` + Beautiful Soup, you parse each response for `h3.base-search-card__title` (job title), `h4.base-search-card__subtitle` (company), `span.job-search-card__location` (location), and `a.base-card__full-link` (URL). The problem is reliability: hit this endpoint directly from a datacenter IP or in a loop, and LinkedIn's rate limiting kicks in. That's exactly the failure mode that pushes people toward a proxy layer.

## Where ScraperAPI fits into a Python LinkedIn scraper

ScraperAPI sits between your script and LinkedIn. You wrap any target URL in an API call — `http://api.scraperapi.com?api_key=YOUR_KEY&url=TARGET_URL` — and their infrastructure does the IP rotation (a pool of 150M+ residential, datacenter, and mobile IPs), JavaScript rendering, CAPTCHA solving, and header management. Your Python code stays the same `requests`-based loop; only the URL changes.

For LinkedIn specifically there are two parameters worth knowing:

- `ultra_premium=true` routes the request through their toughest proxy tier for heavily protected sites — that's the recommended setting for LinkedIn when you're getting blocked.
- `country_code` (available for 100+ locations) matters because LinkedIn tailors job listings by geography. If you're building a salary dataset or a talent map for Germany, you want German exit nodes, not whatever IP your server happens to have.
- `output_format=markdown` or `text` converts the page into clean text without parsing HTML yourself — useful if you're feeding job listings into an LLM pipeline.

A minimal request looks like this:

python
payload = {
    'api_key': 'YOUR_SCRAPERAPI_KEY',
    'url': 'https://www.linkedin.com/jobs/search?keywords=developer&location=United%20States&geoId=103644278',
    'country_code': 'us',
    'output_format': 'markdown'
}


Beyond single requests, the platform supports an Async API (queue LinkedIn URLs in bulk, get results via webhook) and a DataPipeline mode for scheduled recurring scrapes — relevant if you're syncing a jobs database daily rather than running one-off scripts.

## ScraperAPI pricing: every plan, with the LinkedIn credit math

Here's the full current plan lineup from ScraperAPI's pricing page. All paid plans include JS rendering, premium proxies, CAPTCHA bypass, auto-retry, and support; annual billing takes 10% off every tier.

| Plan | Monthly price (annual) | Credits/mo | Concurrent threads | Geo-targeting | Get it |
| --- | --- | --- | --- | --- | --- |
| Free plan | $0 | 1,000 | 5 | — | [ Sign up free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 ($44.10) | 100,000 | 20 | US & EU only | [ Start Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 ($134.10) | 1,000,000 | 50 | US & EU only | [ Start Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 ($269.10) | 3,000,000 | 100 | Country-level | [ Start Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 ($427.50) | 5,000,000 | 200 | Country-level | [ Start Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 ($877.50) | 10,500,000 | 300 | Country-level | [ Start Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 ($1,777.50) | 21,500,000 | 500 | Country-level | [ Start Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom quote | 22M+ | 500+ | Country-level | [Contact sales](https://www.scraperapi.com/contact-sales/) |

Now the part almost no comparison post does the math on: **LinkedIn requests cost 30 credits each on ScraperAPI**, not 1. Standard pages cost 1 credit; Amazon costs 5; Google and Bing cost 25; LinkedIn sits at 30. Sites behind Cloudflare, Datadome, or PerimeterX add another 10 credits per request when the bypass fires.

What that means in practice:

- **Hobby ($49/mo):** 100,000 credits ÷ 30 = about **3,333 LinkedIn requests per month**. One job-search page yields up to 25 listings, so that's roughly 80,000 job listings monthly if every request is a clean search page.
- **Startup ($149/mo):** ~33,000 LinkedIn requests, or over 800,000 listings.
- If a request chains through extra protected pages (individual job detail views, for instance), costs climb — use the `max_cost` parameter to cap what any single request can consume, and the Domain Cost Estimator in the dashboard to check a specific URL before you burn credits on it.

Two usage policies worth knowing before you pick a tier. When you run out of credits, scraping pauses by default until renewal. You can opt into **auto-upgrade** (Hobby through Professional) to bump to the next plan automatically, or **pay-as-you-go** (Scaling and above) to keep scraping past your limit at a fixed per-credit rate with a monthly spending cap you set. Concurrency overages just return a 429 error — no charge, no queueing.

My read on the tiers: Hobby covers most individual projects and proofs of concept, and the free plan (1,000 credits, 5 concurrent connections) is enough to test whether your target pages return clean data at all. Startup only makes sense once you're genuinely iterating on job data daily. Business and above are for products with paying users, and the main thing they buy you beyond credits is country-level geo-targeting (Hobby and Startup are locked to US and EU exits) plus unlimited analytics history instead of 30 days.

👉 [Compare all plans and start the 7-day trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI vs. the alternatives, without the marketing

Two comparisons matter for this specific keyword.

**Versus the open-source libraries.** The libs win on price (free) and on field richness — `py-linkedin-jobs-scraper` returns structured data like `applicant_count` and `is_easy_apply` directly, which the guest-endpoint approach makes you parse yourself. They lose on operations: you maintain Chrome, a LinkedIn session that dies on LinkedIn's schedule, and a scraper that breaks when selectors change. If your scraper needs to run on a server unattended, handing the blocking problem to an API is usually cheaper than the hours you'll spend babysitting sessions. If it's a weekend script for your own job hunt, the library is the better deal.

**Versus Bright Data's LinkedIn scraper.** Bright Data offers a purpose-built LinkedIn Scraper API (with an official Python SDK, released September 2026) that returns structured JSON without login or a browser, billing per record — one profile or one job listing equals one credit, with 5,000 free credits monthly for new accounts. Structured-output-per-record is genuinely cheaper if you only need clean fields and volume is high. ScraperAPI's model is different: you get the raw rendered page (or markdown/text) and parse what you need, which is more flexible when you want fields the structured API doesn't expose — and its per-request pricing is more predictable for page-level scraping. Neither approach is strictly better; it depends on whether you want structured records or full-page control.

One honest limitation on both counts: everything here concerns **public** LinkedIn data. Closed-network content behind a login is a different game with real legal exposure, and the open-source projects themselves state their tools are for personal/educational use with the data belonging to LinkedIn.

## FAQ

**Do I need a LinkedIn account to scrape job listings in Python?**
For the public guest endpoint and headless-browser libraries, yes for the latter, no for the former — the guest jobs API (`/jobs-guest/...`) returns listings without authentication, but libraries like `py-linkedin-jobs-scraper` require a one-time LinkedIn login to open a real session, and that session cookie expires and must be replaced by hand. ScraperAPI needs no LinkedIn login at all; it fetches public pages through its own proxy infrastructure.

**How much does it cost to scrape LinkedIn with ScraperAPI?**
LinkedIn requests cost 30 credits each. The free plan (1,000 credits) covers about 33 LinkedIn requests; the $49 Hobby plan covers roughly 3,333 per month. Since each jobs search page returns up to 25 listings, Hobby translates to very roughly 80,000 listings per month for pure search-page scraping. Check any specific URL's exact cost with the Domain Cost Estimator, and set `max_cost` to keep a single request from eating your balance.

**Is scraping LinkedIn job listings legal?**
Public job data is the lower-risk end of scraping, but it's still contested territory. LinkedIn actively blocks scrapers, and the open-source projects carry disclaimers that the data belongs to LinkedIn and the tools are for personal or educational use. If you're building a commercial product, talk to a lawyer — no pricing table changes that.

**Can I try before paying?**
Yes. ScraperAPI gives new accounts a 7-day trial with 5,000 credits, no credit card required, plus a permanent free plan with 1,000 credits and 5 concurrent connections. That's enough to run a real pagination loop against your target search and confirm data quality.

## Which route to take

Run through this once and pick:

- One-off pulls, under a few thousand listings, you're okay re-logging into LinkedIn occasionally → the open-source library route. Free, and the structured fields are nice.
- Recurring or unattended scrapes, or you've already had a scraper die mid-run from rate limiting → the API route, starting with the free trial. Don't buy a paid plan until the free credits have confirmed your target pages return what you need.
- You only want pre-structured JSON records at volume and don't care about full-page flexibility → Bright Data's per-record model will likely cost less; ScraperAPI wins when you want raw page control, markdown output for LLM pipelines, or geo-targeting flexibility.

If you go the API route, one last tip: set `max_cost` on your very first request, not after you've wondered where your credits went. And start on the free tier — 👉 [the 5,000-credit trial signup](https://www.scraperapi.com/?fp_ref=coupons) asks for no card, so the only thing at risk is a few minutes of your time.
