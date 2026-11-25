# Redfin API: Is There an Official One, What Are the Working Alternatives, and Which Should You Use? (Free Python Wrapper, Hosted APIs, MCP, and a General Scraping API)

There is no public, documented Redfin developer API to sign up for. Every package and service reviewed for this guide that returns Redfin data describes itself as unofficial or third-party, and Redfin's own developer documentation never appears in the sources. So the real question is not "where is the API key" but "which of four working routes fits my job."

The routes differ far more than their landing pages suggest. A free Python package gets you an estimate and a walk score in about 20 lines. Hosted APIs get you 40+ filters, 16 detail endpoints and a published latency figure. An MCP server gets Redfin data into an AI client. A general scraping API gets you Redfin plus every other listing site on one bill. Below: what each one actually returns, what it costs, and the two cost traps most write-ups skip.

## Quick picks before the details

- **A few dozen addresses, one-off research or a prototype:** `pip install redfin`. Free, MIT licensed, no key.
- **Production features with filters and a service-level figure:** a hosted Redfin API (RapidAPI listing or RealtyAPI). Budget by credits, and watch the address-lookup surcharge.
- **Tool calls inside an LLM or agent:** the HasData Redfin MCP server, 1,000 free credits a month.
- **Several listing sites, steady volume, one invoice:** ScraperAPI, which has a free tier of 1,000 credits/month at 5 concurrent connections and a 7-day refund policy. 👉 [Check the current ScraperAPI plans and free-tier limits](https://www.scraperapi.com/?fp_ref=coupons)

## Route 1: the free Python wrapper (`pip install redfin`)

The `reteps/redfin` package describes itself as "a Python wrapper around the unofficial Redfin API." It is MIT licensed, installs from PyPI, and the PyPI project page carries a release date of 2026-04-06, so it is still being published rather than abandoned.

The minimum working flow in the project README is short enough to paste:

python
from redfin import Redfin

client = Redfin(user_agent="MyApp/1.0 (you@example.com)", request_delay=1.0)

response = client.search('1600 Amphitheatre Pkwy, Mountain View, CA')
url = response['payload']['exactMatch']['url']

initial_info = client.initial_info(url)
property_id = initial_info['payload']['propertyId']

avm = client.avm_details(property_id, "")
print(avm['payload']['predictedValue'])          # Redfin Estimate

stats = client.neighborhood_stats(property_id)
print(stats['payload']['walkScoreInfo']['walkScoreData']['walkScore']['value'])


Two CLI commands cover the same ground without writing Python: `redfin estimate <property_id>` prints the estimate, beds/baths and last sold price, and `redfin neighborhood <property_id>` prints walk, bike and transit scores. Add `--json` to `estimate` for raw output, `--delay 1.5` to slow the request rate, `--user-agent` to identify your app.

Beyond those, the README lists roughly two dozen property-ID methods: `avm_historical`, `owner_estimate`, `cost_of_home_ownership`, `similar_listings`, `similar_sold`, `nearby_homes`, `property_comments`, `tour_insights`, `floor_plans` and more. That is a wider surface than most people expect from a free package.

**Where it breaks at scale.** The client reads the `Retry-After` header on a 429 and retries **once**. Not three times, not with exponential backoff. If the second attempt also fails, the exception lands in your code. `request_delay=1.0` is a fixed pause between requests ("1 second between requests" in the README), which you set yourself; the package does not throttle adaptively. Anyone pulling thousands of records needs their own queue and backoff layer around it.

The second cost is structural. Values come out of deep response paths like `payload.exactMatch.url` and `payload.walkScoreInfo.walkScoreData.walkScore.value`. There is no versioned field contract. When the underlying pages change shape, your code changes with them.

## Route 2: hosted APIs on RapidAPI

A "Redfin API" listed on RapidAPI is a separate product from the package above. Its page displays popularity 9.6, service level 100%, latency 619ms and test pass rate 100%, and notes that these are page-displayed figures that can move.

The Quick Start is three calls:

1. `GET /autocomplete?location=Austin TX` returns `region_id` and `region_type`.
2. `GET /search?region_id=30818&region_type=6&search_type=ForSale&min_price=400000&min_bedroom=3` returns up to **350 properties per page** with price, beds, baths, sqft, address and the Redfin URL.
3. `GET /property/details?property_id=31897110` returns description, photos, features and price history.

The endpoint list is the selling point: 16 property-detail endpoints, including `/property/estimate` (with a confidence range), `/property/walk-score`, `/property/flood-info`, `/property/parcel-bounds`, `/property/price-drop-info` and `/property/mortgage-calculator`, plus two market-data endpoints. Coverage claims all 50 states, 30,000+ cities and every US ZIP code. Filters run past 40, with boolean flags for pool, waterfront, basement, fireplace, fixer-upper and similar.

One practical warning: the parameter names here are specific. This API uses `search_type=ForSale` and `min_bedroom`; RealtyAPI (below) uses `searchType=For_Sale` and `minBeds=Three`. Copying a query string from one service to the other returns an error, not a result set.

Pricing on the listing: the page has a "Subscription plans" heading but no plan names or figures in the body, so there is no number to quote here.

## Route 3: RealtyAPI's Redfin endpoints (OpenAPI 3.1.0)

This one publishes a machine-readable spec: OpenAPI 3.1.0, version 1.0.0, server `redfin.realtyapi.io`, API key sent in the `x-realtyapi-key` header. Every response carries `x-credits-limit` and `x-credits-remaining` headers, so you can meter a run without guessing.

**The surcharge worth knowing about.** The `/detailsbyaddress` endpoint is documented as costing **2 requests per call** and running **2x slower than `/detailsByID`**. Same house, double the credits, half the throughput, purely because you started from a street address. For a batch job, resolve `property_id` once through search and then fetch details by ID. On 5,000 lookups that difference is 10,000 billable requests instead of 5,000, and twice the wall-clock time.

Other details that matter for design:

- `/search/bycoordinates` takes latitude, longitude and a `radius` in miles, which the spec describes as a square area rather than a circle.
- `resultCount` defaults to **100** and maxes out at **1,000**; `page` returns a `nextPage` boolean you loop until false.
- `soldWithin` defaults to `Last_7_Days`, so a sold search without that parameter silently returns a week of data.
- Bedroom filters are enums (`Studio`, `One` … `SixPlus`), not integers.
- Mortgage payment filters assume a **6.5% 30-year fixed** loan including HOA, taxes and insurance.
- `/housingMarketTrends` returns median sale price, homes sold, days on market with sale-to-list, above-list share and price drops; `location` accepts a ZIP, `Cary, NC`, a county or neighborhood, or a region ID in the form `4_10960`.

**Pagination arithmetic.** Pulling 10,000 listings costs about 29 requests at 350 per page on the RapidAPI product, 10 requests at RealtyAPI's 1,000 maximum, or 100 requests at its default of 100. Same data, tenfold difference in request count depending on which service and which `resultCount` you use. Credits and rate limits both scale with that number.

## Route 4: an MCP server for AI clients

`HasData/redfin-mcp` is an MCP server aimed at AI clients rather than application backends. The project description covers search across for-sale, for-rent and sold listings with a full filter set, plus reading a single listing page's full text, and states MIT licensing, **1,000 free credits per month**, and no MLS licence. If your product is a chat assistant that answers "what's for sale near me", this removes the need to write a tool wrapper yourself. If you need to store and version structured records, a REST API is still the more direct route.

## Route 5: general-purpose scraping when Redfin is not the only site

The four routes above are Redfin-shaped. The moment your pipeline also needs Zillow, Realtor.com, county records or a broker's own site, you are maintaining N scrapers instead of one.

ScraperAPI is a general scraping API rather than a Redfin endpoint catalogue: you send a target URL plus your key, and it handles proxy rotation and bot-protection bypass. The company publishes a [Redfin scraping walkthrough](https://www.scraperapi.com/web-scraping/redfin?fp_ref=coupons) covering sale and rental listing pages, `country_code=us` geo-targeting, structured data endpoints, and an async scraper for up to 10,000 URLs. Parsing stays your job, or you use their structured output.

**How the credit model works, because it decides your budget.** A standard page costs 1 credit. Amazon costs 5, Google and Bing 25, LinkedIn 30, and any site behind Cloudflare, DataDome or PerimeterX adds 10 credits per request when the bypass is used. There is a Domain Cost Estimator in the dashboard, and you can set `max_cost` per request.

If 10,000 Redfin listing pages cost roughly 10,000 credits at 1 credit each, the free tier of **1,000 credits/month with 5 concurrent connections** covers about a thousand pages monthly, and the first 7 days after signup include **5,000 credits** for a larger test.

Two behaviours to plan for. Exceeding your concurrency limit returns a 429 and the request is turned away rather than queued; you are not charged extra for the attempt. Running out of credits pauses scraping until renewal unless you enable auto-upgrade (available on Hobby, Startup, Business, Scaling and Professional) or pay-as-you-go (Scaling, Professional, Advanced and Enterprise, with a monthly spending cap you set). Annual plans hand over the full year's credits upfront and can be renewed early for a fresh batch. Cancellation is free at any time, and there is a **7-day no-questions-asked refund**.

### ScraperAPI plans currently listed

Per-plan credit allowances and prices sit on the pricing page and change; the table below records what the documentation confirms about each tier's mechanics.

| Plan | Confirmed mechanics | Billing | Start |
| --- | --- | --- | --- |
| Free | 1,000 credits/month, 5 concurrent connections, 5,000 credits in the first 7 days | Monthly | [Start on the free plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | Entry paid tier; optional auto-upgrade to the next plan when credits run out | Monthly or annual | [Check the Hobby price](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | Auto-upgrade available | Monthly or annual | [Check the Startup price](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | Auto-upgrade available | Monthly or annual | [Check the Business price](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | Auto-upgrade plus pay-as-you-go | Monthly or annual | [Check the Scaling price](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | Auto-upgrade plus pay-as-you-go | Monthly or annual | [Check the Professional price](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | Pay-as-you-go with a monthly spending cap | Monthly or annual | [Check the Advanced price](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Pay-as-you-go; limits and plan changes handled with sales | Custom | [Talk to ScraperAPI about Enterprise](https://www.scraperapi.com/?fp_ref=coupons) |

The gap between Hobby/Startup/Business (auto-upgrade, no pay-as-you-go) and Scaling upward (both) is the real dividing line in that table. If your monthly volume swings, you want a tier with pay-as-you-go and a cap, not one that silently moves you up a plan.

## Which route fits your job

| Your situation | Use | Who does not need it |
| --- | --- | --- |
| 50 addresses, a spreadsheet, done by Friday | Free `redfin` package | Anyone who needs an SLA or a support contact |
| An app with filters, 16 detail endpoints and published uptime | A hosted Redfin API | Anyone under ~1,000 lookups a month, where the free package is enough |
| Redfin data inside ChatGPT-style tool calls | HasData MCP server | Teams storing records in their own database |
| Redfin plus three other sites, steady volume, one invoice | ScraperAPI | Single-site jobs with no parsing budget |

The recurring decision is between paying for structure and paying for coverage. Hosted Redfin APIs sell structure: named endpoints, typed filters, published latency. A scraping API sells coverage: the same request pattern works on any site, and you own the extraction logic. If you already have a parser you trust, the scraping route usually wins on flexibility. If you do not, hosted endpoints save weeks.

## Limits and risks worth pricing in

**Unofficial interfaces move.** None of these routes carries a versioned contract from Redfin. Field paths change, endpoints get renamed, and the free package's nested dictionaries are the most exposed of the lot.

**Rate limits are real.** The free package ships a fixed `request_delay` and a single 429 retry, which is a signal about how the underlying site behaves under load rather than a guarantee of tolerance.

**Credit maths has traps.** RealtyAPI's address lookup doubling is the clearest one. On scraping APIs, per-domain multipliers (5 credits for Amazon, 25 for Google, +10 for bot-protected sites) mean your effective budget depends on the target list, not the headline credit count.

**Terms are your call.** The material reviewed here does not cover Redfin's terms of service or its robots policy, so confirm what the target site allows before you order a paid plan and point it at that site.

## FAQ

**Does Redfin have an official API?**
No public, documented Redfin developer API appears in the sources reviewed. Every package and service that returns Redfin data calls itself unofficial or third-party.

**How much data can I pull with the free Python package?**
There is no published quota; throughput depends on the delays you configure and how the site responds. With `request_delay=1.0` you are issuing about one request per second, and a 429 that survives one retry stops the run until you handle it.

**Can I get the Redfin Estimate and walk score from an API?**
Yes on both. In the free package, the estimate is `avm_details(property_id, "")` at `payload.predictedValue`, and the walk score is `neighborhood_stats(property_id)` at `payload.walkScoreInfo.walkScoreData.walkScore.value`. The RapidAPI product exposes `/property/estimate` and `/property/walk-score`; RealtyAPI returns trends and details through its own endpoints.

**Why does an address lookup cost more than a property ID lookup?**
RealtyAPI documents `/detailsbyaddress` at 2 requests per call and 2x slower than the ID-based endpoint, because the address has to be resolved first. Search once, store the property ID, then fetch details by ID for the rest of the run.

**Which option is best for a one-off list of 50 addresses?**
The free `redfin` package. A loop with `request_delay` set, a `try/except` around each call and the property IDs cached to a file will finish faster than signing up for anything.

**Can I use these inside an AI agent?**
The HasData MCP server is built for that, with 1,000 free credits a month and MIT licensing. Otherwise wrap any REST endpoint as a tool yourself.

## Picking one

If the job is a one-off list, install `redfin`, set a delay, cache property IDs, and stop there. If you are shipping a product that needs filters, detail endpoints and someone to call when latency spikes, a hosted Redfin API is the honest expense. If the agent is the product, start with the MCP server's free credits. If Redfin is one of several sites you scrape and you want one bill, one concurrency limit and one refund policy, 👉 [open the ScraperAPI pricing page and size the plan against your monthly page count](https://www.scraperapi.com/?fp_ref=coupons) — the 1,000-credit free tier plus the first-week 5,000 credits is enough to measure your real per-page cost before you pay anything.
