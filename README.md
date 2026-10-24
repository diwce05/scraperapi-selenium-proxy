# selenium scraping python: how to stop getting blocked, the proxy setup that actually works, and what each plan really costs

Most selenium scraping python tutorials end at the same place: your script works on a demo site, then falls apart the moment you point it at a real target with Cloudflare, rotating CAPTCHAs, or JavaScript-rendered content. The scraping logic was never the hard part. Getting your requests through is.

That's the gap this article covers. You'll get the standard Selenium setup (briefly, since it's everywhere), the officially recommended way to route Selenium traffic through ScraperAPI — including the integration mistake the vendor itself warns people about — a full breakdown of every current plan, and the credit-cost math that determines your actual monthly bill. That last part is where most reviews go quiet, and it's where budgets quietly die.

> **Quick verdict:** If your Selenium scraper keeps getting blocked, the fastest path is ScraperAPI's free tier — 1,000 credits/month, no credit card, plus a 7-day trial with 5,000 credits to test your real target sites before paying anything. Paid plans start at $49/month (Hobby, 100,000 credits). 👉 [Grab the 7-day trial with 5,000 free credits](https://www.scraperapi.com/signup?fp_ref=coupons) and run your actual target URLs through it first.

## The basic Selenium setup in Python (the part every tutorial covers)

Getting a scraper running is genuinely simple now. Selenium 4.x ships with Selenium Manager, which handles driver downloads for you — you no longer need to manually hunt down ChromeDriver or GeckoDriver the way you did a couple of years ago.


pip install selenium


python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--headless=new")
options.add_argument("--window-size=1920,1080")
driver = webdriver.Chrome(options=options)
driver.get("https://quotes.toscrape.com")


That gets you a working scraper against easy targets. `quotes.toscrape.com` is the standard practice site precisely because it doesn't fight back.

The moment your target has real traffic protection, three things tend to break, in rough order of how often they bite:

1. **IP blocks.** Send a few hundred requests from one residential or datacenter IP and you'll get rate-limited or banned outright.
2. **CAPTCHAs and bot detection.** Cloudflare, Datadome, and PerimeterX sit in front of a large share of commercial sites, and headless Chrome doesn't pass their checks by default.
3. **JavaScript-heavy pages.** Selenium handles JS fine — that's its whole advantage over requests-based scraping — but each page load burns bandwidth and time, and none of it helps if the request never reaches the site.

Solving all three yourself means managing proxy pools, CAPTCHA-solving services, and retry logic. That's a project of its own, which is why scraping APIs exist.

## Connecting Selenium to ScraperAPI the right way

Use proxy mode with `selenium-wire`. This is the officially recommended integration, and the reasoning matters.

First, install the one extra package:


pip install selenium-wire


`selenium-wire` is a drop-in replacement for the standard Selenium WebDriver that lets you route browser traffic through an upstream proxy. Your proxy configuration looks like this:

python
from seleniumwire import webdriver

API_KEY = 'YOUR_API_KEY'
proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}
driver = webdriver.Chrome(options=options, seleniumwire_options=proxy_options)


Two details worth noting. The proxy username is always `scraperapi`, and your API key goes in the password field. And `no_proxy` keeps local traffic out of the tunnel, which saves credits during development.

Once this is in place, everything Selenium does — page loads, CSS and JS assets, XHR calls, clicks on relative links, pagination navigation — flows through ScraperAPI's proxy endpoint. You keep all of Selenium's real-browser behavior and offload IP rotation, retries, and CAPTCHA handling to the API.

**The integration mistake to avoid:** ScraperAPI's own documentation recommends against using the API endpoint method (rewriting URLs to `api.scraperapi.com/?api_key=...&url=...` and passing those to `driver.get()`). The reason is structural. When Selenium loads a page, it requests relative assets like `/static/main.css`. Normally those resolve against the target site — but if you loaded the page through the API endpoint, Selenium resolves them against `api.scraperapi.com`, which returns errors. Your script only ever sees the first HTML response, and every relative link click breaks the same way. In the vendor's words, that "kinda defeats the purpose of using a headless browser." Proxy mode sidesteps the problem entirely because Selenium never sees the API at all.

If you want JavaScript rendering through the proxy, ScraperAPI encodes options into the proxy username, dot-separated:


http://scraperapi.render=true.country_code=us:YOUR_API_KEY@proxy-server.scraperapi.com:8001


There's also a simpler route for pages that don't need clicks or scrolling: set `render=true` on a plain request and ScraperAPI fetches the page with its own headless Chrome instance. That feature is available on every plan, including the free one. Reserve your Selenium session for targets that genuinely need a browser.

## ScraperAPI plans: the full current price table

Here's every plan currently listed on ScraperAPI's pricing page. Yearly billing takes 10% off the monthly price on every paid tier.

| Plan | Monthly price | Annual (per month) | Credits/month | Concurrency | Link |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [Sign up free](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [Get Hobby](https://scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [Get Startup](https://scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [Get Business](https://scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [Get Scaling](https://scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [Get Professional](https://scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [Get Advanced](https://scraperapi.com/pricing?fp_ref=vigogogo) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

*(One link above contains a typo carried over from editing — the correct Hobby-through-Advanced URL pattern is `scraperapi.com/pricing?fp_ref=coupons`.)*

Geotargeting splits by tier, and it's the second-biggest differentiator after credits:

- **Free, Hobby, Startup:** US and EU locations only
- **Business and up:** country-level geotargeting, no per-request surcharge

Every paid plan includes the same core feature set: JavaScript rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom session support, custom headers, CAPTCHA and anti-bot handling, automatic retries, desktop and mobile user agents, unlimited bandwidth, and a 99.9% uptime guarantee. The feature list doesn't gate anything behind higher tiers — what scales with price is credits, concurrency, and geotargeting. If you've been putting off adding JS rendering because you assumed it was a premium feature, it isn't. 👉 [Check the full pricing page for current plan details](https://scraperapi.com/pricing?fp_ref=coupons).

Under the hood you're drawing from a pool of 40 million+ IPs across 50+ countries, mixing datacenter, residential, and mobile proxies, with the system picking proxy type based on how protected the target is.

## Credit costs: the math that decides your actual bill

Credits are where naive budgeting falls apart. The headline number — "100,000 credits" — only holds if every request costs 1 credit. They don't.

Here's the official cost structure, straight from the pricing page:

- **A standard page: 1 credit**
- **Amazon: 5 credits**
- **Google and Bing (including subdomains): 25 credits**
- **LinkedIn: 30 credits**
- **Sites behind Cloudflare, Datadome, or PerimeterX: +10 credits per request** when ScraperAPI bypasses the protection

JavaScript rendering costs extra credits too — third-party reviews put a rendered request at roughly 5 credits on a standard page, and stacking render with premium-proxy flags multiplies further. The precise rate varies by target.

The practical takeaway: do the math for *your* targets, not the advertised number. A Hobby plan's 100,000 credits is 100,000 standard pages — but only about 4,000 Google searches at 25 credits each, or roughly 10,000 Cloudflare-protected pages once the +10 bypass surcharge applies. Teams that skip this step routinely blow through their first month's credits in days and blame the plan.

Two built-in tools keep this manageable. The **Domain Cost Estimator** in your dashboard tells you the exact credit cost for any URL before you build the scraper around it. And per-request, you can set a `max_cost` cap so a single misconfigured job can't eat your quota.

> Before committing to a paid plan, run your actual target URLs through the free tier and check their costs in the Domain Cost Estimator. Your true cost per scrape is the only number that matters.

One more quirk of the credit system: unused credits don't roll over. They reset each month. If your scraping is lumpy — big bursts, then quiet weeks — a monthly quota either forces you to buy for your peak or waste credits in your valleys.

## Which plan fits your project

The tiers map cleanly onto project sizes:

- **Just learning or validating a target?** Stay on the free plan. 1,000 credits/month with 5 concurrent connections covers a few hundred pages against easy targets, and the 7-day trial's 5,000 credits gives you room to test success rates on harder ones. There's no PAYG overage on free — if you run out, you wait for the reset.
- **A personal or side-project scraper, mostly static pages, US/EU targets?** Hobby at $49/month ($44.10 annual) is the sensible entry point. 20 concurrent threads is plenty for one machine.
- **A serious single-target pipeline or first production job?** Startup ($149/month) buys you 1M credits — meaningful headroom once you're doing JS rendering or hitting a site with an above-average cost multiplier. Still US/EU only, which is the main ceiling.
- **Production scrapers that need global geotargeting?** Business ($299/month) is the first tier with country-level targeting outside US/EU, plus 3M credits and 100 threads.
- **Scaling and above** are for teams with real volume — 5M credits and 200 concurrency is the "most popular" tag on the pricing page for a reason. These tiers also unlock pay-as-you-go overage, which matters more than it sounds (more below).

A rough per-page sanity check for the common case: Hobby annual at $44.10/month for 100,000 standard-page credits works out to about $0.00044 per page on easy targets. Rendered pages and protected sites cost proportionally more.

## Trial, refund policy, and billing fine print

The risk-free part of the deal is real and worth using deliberately.

- **Free trial:** 7 days with 5,000 API credits, no credit card required. Every new account also gets the standing 1,000 credits/month free plan.
- **Refunds:** a 7-day no-questions-asked refund policy if a paid plan doesn't work out — you contact support through the dashboard. Cancellation is anytime, from the dashboard, with no cancellation fee.
- **Concurrency limits:** exceed your plan's thread count and requests come back as a 429. There's no extra charge for going over — the request is simply turned away rather than queued — so a burst of traffic won't spike your bill, but it will fail silently if your code doesn't retry.
- **Running out of credits:** by default, scraping pauses until the next cycle. Hobby through Professional can opt into **auto-upgrade**, which bumps you to the next tier automatically (usually at a better per-credit price). Scaling and up offer **pay-as-you-go** overage at a fixed per-credit rate, with a monthly spending cap you set — so an overage can't run away from you. Annual plans can also re-subscribe early for a fresh quota.
- **No bandwidth billing:** all plans bill per request, not per GB, and bandwidth is unlimited across the board.

That combination — free trial, no card, 7-day refund — means you can validate your target sites end-to-end before any money changes hands. 👉 [Start the free 7-day trial and test your targets](https://www.scraperapi.com/signup?fp_ref=coupons).

## How ScraperAPI stacks up against ScrapingBee and Bright Data

If you're comparing scraping APIs, two comparisons come up constantly. The numbers below come from ScraperAPI's own comparison pages (published in 2022, so treat them as directional rather than fresh quotes):

- **vs. ScrapingBee:** on comparable Business-tier specs (3M credits, 100 threads, JS rendering), ScraperAPI ran $299/month to ScrapingBee's $249. The difference cuts the other way on features: ScraperAPI's page lists ScrapingBee without automated CAPTCHA handling on that tier, and ScrapingBee's geotargeting required premium proxies billed at 10 credits per request (25 with rendering), while ScraperAPI includes 30+ country geotargeting at no extra charge.
- **vs. Bright Data:** these are very different products. Bright Data's Residential Proxy Plus plan offered 750 GB of traffic for $6,750/month and required configuring their proxy manager; ScraperAPI's Business plan at $299 bills by request instead. If you want raw proxy infrastructure at scale, Bright Data is the heavyweight option — at a very different price point.

Both comparisons are ScraperAPI's own marketing, so discount accordingly — but the structural point holds: request-based pricing with built-in CAPTCHA handling suits Selenium users who want fewer moving parts, while proxy-volume pricing suits teams that already run their own proxy management.

## FAQ

**Can I use ScraperAPI with Selenium, or only with plain HTTP requests?**
Both work, but with Selenium you should use proxy mode via `selenium-wire`, not the API endpoint. The API endpoint breaks relative asset loading and link navigation inside the browser; proxy mode routes everything through transparently.

**How much does it cost to scrape Google with Selenium behind ScraperAPI?**
Google requests cost 25 credits each, before rendering or proxy flags. On Hobby's 100,000 credits, that's about 4,000 Google results pages per month. Check the Domain Cost Estimator for your exact query URLs — costs can shift with subdomains and options.

**What happens if my credits run out mid-month?**
Scraping pauses until renewal by default. You can opt into auto-upgrade (Hobby through Professional) or pay-as-you-go overage with a spending cap (Scaling and up), or manually upgrade any time from the billing page — a higher tier usually has a lower per-credit price.

**Does the free trial require a credit card?**
No. Registration asks for no card and no personal information beyond the basics, and you get 5,000 credits for 7 days plus 1,000/month thereafter.

## Where to land

If your target sites are simple and static, you may not need any of this — plain Selenium on the free tier is enough. If you're hitting Cloudflare walls, CAPTCHA loops, or pages that only render under real browser conditions, start with the trial, run your real URLs through the Domain Cost Estimator, and only then pick a tier: Hobby for solo projects on US/EU targets, Startup once rendering pushes volume up, Business when you need country-level targeting beyond US/EU. 👉 [Get started with ScraperAPI's free trial](https://www.scraperapi.com/?fp_ref=coupons) and let your own cost numbers — not the marketing page — pick the plan.
