# python proxy: Set It Up in requests, Rotate It Without Getting Blocked, and Budget by the Gigabyte

Two different people search this phrase. One is three lines into `requests` and wants to know where the `proxies=` dictionary actually goes. The other already has a working scraper that keeps coming back with 403s and wants to know which proxy to buy before the weekend.

Both questions are answered below, in that order. Code first, because no provider on earth saves a script that quietly keeps using your own IP.

## The five-line version

Here's the smallest thing that works.

python
import requests

proxy = "http://USER:PASS@gateway.example.com:823"
proxies = {"http": proxy, "https": proxy}

r = requests.get("https://httpbin.org/ip", proxies=proxies, timeout=10)
print(r.json())


If the JSON response shows the proxy's IP rather than yours, the wiring is correct. Hitting an IP echo endpoint is the cheapest possible test, and you should run it before you point the script at a real target, because debugging two things at once (proxy + parser) wastes an afternoon.

Three details break people here.

**The scheme in the proxy URL is how you reach the proxy, not what it carries.** An HTTP proxy that forwards HTTPS traffic still gets addressed as `http://user:pass@host:port`. Writing `https://user:pass@host:port` because your target site is HTTPS produces connection errors with messages that don't explain themselves.

**Setting only one key leaks traffic.** If `proxies` has an `https` entry and no `http` entry, plain HTTP requests go out from your own IP. Set both, always, even if you only care about one.

**No timeout means a dead proxy hangs your script.** `timeout=(5, 30)` sets separate connect and read limits, which is what you want when the failure mode is a proxy that accepts the TCP connection and then does nothing.

Environment variables (`HTTP_PROXY`, `HTTPS_PROXY`) work for a single fixed proxy and are genuinely convenient in dev. They're the wrong tool for rotation, since changing the exit IP means restarting the process.

## http, https and socks5: what the dictionary keys actually mean

The `proxies` dict is a routing table, and it accepts more keys than most tutorials show. You can route per scheme, and you can route per host:

python
proxies = {
    "http": PROXY_A,
    "https": PROXY_B,
    "https://example.org": PROXY_C,
}


That last entry sends one specific domain through a different endpoint while everything else follows the general rule. Useful when a single stubborn target needs a different IP pool than the rest of your crawl.

SOCKS5 is a separate story. It needs `pip install "requests[socks]"` (PySocks under the hood), and the URL uses `socks5://user:pass@host:port`. SOCKS5 proxies traffic at a lower level than HTTP, which matters for non-HTTP traffic; for ordinary page scraping the difference is mostly academic. DataImpulse supports HTTP, HTTPS and SOCKS5, so you're not locked into one.

## Rotating versus sticky: the decision that actually determines success

This is where scrapers live or die, and it has nothing to do with code quality.

- **Rotating** gives you a new exit IP per request (or per the provider's rotation rule). It's the right choice for fetching lots of public pages: search results, catalogues, price feeds, SERPs.
- **Sticky** pins one IP to one session for a window of time. It's the right choice whenever server-side state is tied to an address: logging in, filling a cart, paging through account pages, anything where a cookie jar and an IP need to agree.

The mistake that produces bizarre failures is mixing identities. If you hold a logged-in cookie set for account X and then rotate the IP every request, you've built a user who teleports between countries mid-session. Treat proxy, cookies, user agent and target account as one session record, and keep them together.

With DataImpulse, sticky sessions can be held from roughly 1 to 120 minutes, with 30 minutes as the default average if you don't specify a rotation interval, and the session lives on a port drawn from the 10000–20000 range. Rotating endpoints are the other side of the same product.

## Skipping the proxy list entirely

Most Python proxy guides hand you a `PROXY_POOL` list and a `cycle()` from itertools. That works, and it's also the reason those guides are 4,000 words long: you own health checks, eviction, re-admission, retries and backoff.

There's a shorter path. Providers that run their own pool give you one gateway endpoint and move the targeting into the username, so the rotation happens upstream and your code stays small.

python
USER, PASS = "your_user", "your_pass"
GW = "gateway.example.com:823"

rotating  = f"http://{USER}:{PASS}@{GW}"
sticky    = f"http://{USER}:{PASS}_session-abc123@{GW}"
us_only   = f"http://{USER}:{PASS}_country-us@{GW}"
nyc       = f"http://{USER}:{PASS}_country-us_city-newyork@{GW}"

r = requests.get("https://httpbin.org/ip", proxies={"http": rotating, "https": rotating})


This append-to-the-password convention is what DataImpulse documents in its integration files (rotating on the plain endpoint, `_session-*` for sticky, `_country-*` and `_city-*` for geo). Copy the exact gateway host and port from your dashboard rather than trusting a blog post, since vendors change endpoints.

One cost detail worth internalizing before you hard-code city targeting: country-level targeting is included in the base rate, but city, ZIP and ASN filters on residential traffic bill at **double** the standard per-GB rate. That's straight from DataImpulse's own pricing guide and confirmed by third-party writeups. A scraper that silently pins every request to a ZIP code can burn through a balance at twice the speed you budgeted for.

👉 [Test the geo-parameter pattern on a $5 pack before you scale anything](https://bit.ly/dataimPulse)

## What actually breaks in production

The proxy is rarely the problem. The error handling is.

python
import time, requests
from requests.exceptions import ProxyError, ReadTimeout, ConnectTimeout, HTTPError

def fetch(url, proxies, attempts=4):
    for i in range(attempts):
        try:
            r = requests.get(url, proxies=proxies, timeout=(5, 30))
            if r.status_code in (403, 429):
                raise HTTPError(f"pushed back: {r.status_code}")
            r.raise_for_status()
            if not r.content:
                raise ValueError("empty body")
            return r.text
        except (ProxyError, ReadTimeout, ConnectTimeout, HTTPError, ValueError):
            time.sleep(2 ** i)
    return None


Four things in there matter more than they look.

`ProxyError`, `ReadTimeout` and `ConnectTimeout` are three distinct failures, and catching only one leaves the other two crashing the run. Explicit `2 ** i` backoff keeps you from hammering a target that just told you to slow down. And the empty-body check catches the sneakiest case: a 200 response that's actually a consent wall or a challenge page. HTTP success means a document arrived, not that it's the document you asked for.

There's a billing angle too. Blocked and retried requests still move bytes, and that traffic counts. Fifty percent of your retries hitting 429s is a 50% surcharge on your cost per usable record. Which is why the sane onboarding step is a small pack, a handful of your real target URLs, and a measurement of cost per *successful* request rather than per gigabyte.

## Beyond requests

**aiohttp and httpx.** Concurrency changes the failure profile: one wedged connection no longer stalls the run, but you now need per-request timeouts and a semaphore to avoid opening 500 sockets. The proxy wiring is the same idea, passed per request.

**Scrapy.** Two options. Either `scrapy-rotating-proxies` with your own list, which adds a middleware for proxy assignment plus ban detection and per-proxy health tracking, or point `DOWNLOADER_MIDDLEWARES` at nothing at all and set a single rotating gateway as the proxy, letting the provider handle rotation. The second option removes an entire class of maintenance work; the tradeoff is less control over which IP lands on which request.

**Playwright and Selenium.** Browsers take the proxy at launch, and authenticated proxies are painful because Chrome handles proxy auth through a dialog rather than request headers. The practical workaround is IP whitelisting instead of username/password, so no credentials need to reach the browser at all. DataImpulse supports both authentication methods, which is the kind of detail that only matters once you've spent an evening fighting `--proxy-server` flags.

## Free lists versus paying per gigabyte

Free proxy lists are not free. They're paid for in debugging time, dead endpoints, and IPs with other people's abuse history attached, which is precisely the fingerprint that anti-bot systems score against.

The per-GB model is easier to reason about. A typical HTML page runs somewhere between 100 KB and 500 KB, so a gigabyte is roughly 2,000 to 10,000 page fetches before retries. At $1/GB, that's in the neighborhood of $0.0001–$0.0005 per fetch. Whether that's cheap depends entirely on your success rate and your target, which is why you measure on a small pack instead of assuming.

## DataImpulse pricing, all four product lines

DataImpulse sells four proxy types on a pay-as-you-go basis with no subscription, a $5 minimum first top-up, and balances that don't expire. Here's the published ladder.

| Proxy type | Plan | Traffic | Price | Per GB | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5 | $1.00 | [ Grab the 5 GB starter pack](https://bit.ly/dataimPulse) |
| Residential | Basic | 50 GB | $50 | $1.00 | [ Buy 50 GB at the flat rate](https://bit.ly/dataimPulse) |
| Residential | Advanced | 1 TB | $800 | $0.80 (20% off) | [ Check the 1 TB residential price](https://bit.ly/dataimPulse) |
| Datacenter | Intro | 10 GB | $5 | $0.50 | [ Start with 10 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Datacenter | Basic | 100 GB | $50 | $0.50 | [ Buy 100 GB for high-volume crawling](https://bit.ly/dataimPulse) |
| Datacenter | Advanced | 1 TB | $450 | $0.45 | [ See the 1 TB datacenter rate](https://bit.ly/dataimPulse) |
| Datacenter | Custom | 5 TB+ | from $2,250 | negotiated | [ Request a datacenter quote](https://bit.ly/dataimPulse) |
| Mobile | Intro | 2.5 GB | $5 | $2.00 | [ Test mobile IPs from $5](https://bit.ly/dataimPulse) |
| Mobile | Basic | 25 GB | $50 | $2.00 | [ Buy 25 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Mobile | Advanced | 1 TB | $1,600 | $1.60 (20% off) | [ Check the 1 TB mobile rate](https://bit.ly/dataimPulse) |
| Mobile | Custom | 5 TB+ | from $8,000 | negotiated | [ Ask about mobile volume pricing](https://bit.ly/dataimPulse) |
| Premium residential | Intro | 1 GB | $5 | $5.00 | [ Try premium residential for $5](https://bit.ly/dataimPulse) |
| Premium residential | Basic | 10 GB | $50 | $5.00 | [ Buy 10 GB of premium residential](https://bit.ly/dataimPulse) |
| Premium residential | Custom | 5 TB+ | from $20,000 | negotiated | [ Request a premium residential quote](https://bit.ly/dataimPulse) |

Two notes that affect what you actually pay. Traffic routed through city, ZIP or ASN filters on the standard residential line bills at 2× the listed per-GB rate; country targeting is included. And the 20% volume discount on residential and mobile only kicks in at the 1 TB tier, which is a long way from a $5 test pack.

## Which plan fits which Python workload

**Learning the API and validating success rates:** the residential $5 / 5 GB pack. It's enough for a few thousand real requests against your actual targets, which is the only number that predicts anything.

**Public pages, no serious anti-bot:** datacenter at $0.50/GB. Half the residential price and lower latency. If your targets are documentation sites, news archives, or your own projects, paying for residential IPs is paying for capabilities you aren't using.

**Retail, classifieds, SERPs, anything behind Cloudflare or Akamai:** standard residential. The whole point of the $1/GB line is that requests look like ordinary home connections, which is why it's the default for scraping work.

**Mobile app data and the most defensive platforms:** mobile at $2/GB. It's the highest per-GB cost of the four, so it's the line item most likely to surprise you at month end. Budget it deliberately.

**Long-lived browser profiles and account work:** premium residential at $5/GB, or consider whether static ISP proxies are the better tool. A TechRadar review of the service makes this same split, and also flags the honest downside: DataImpulse is developer-first, so if you want a fully managed, hands-off extraction product with the proxy underneath it, this isn't that. You get infrastructure and a dashboard, not a scraping team.

## Gotchas before you pay

There's no free trial. Every route in starts with a $5 minimum purchase, which is why the intro pack doubles as the trial.

The 7-day money-back guarantee covers Intro plans paid by card, as long as less than 80% of the traffic is consumed. Crypto purchases on those plans aren't refundable. Read that before topping up with USDT.

There's also no promo code worth hunting. Sources that track DataImpulse coupons keep landing on the same conclusion: the flat $1/GB is the offer, and the $5 starter pack is the entry point. Discount codes that circulate elsewhere are usually recycled or expired.

Finally, the reason the pay-as-you-go model suits Python work specifically: crawlers are bursty. A scraper that runs hard for a week and then sits idle for two months is a terrible fit for a monthly subscription and a perfectly fine fit for a balance that doesn't expire. Your unused gigabytes are still there when the next project starts.

👉 [Start with 5 GB for $5 and find your real cost per successful request](https://bit.ly/dataimPulse)
