# selenium rotating proxy: Getting Selenium to rotate IPs without breaking logins, headless servers, or your budget

Selenium is the wrong tool for most scraping jobs, and that's exactly why proxy rotation in Selenium is harder than in `requests`. A single `driver.get()` doesn't make one request. It fetches the document, then every stylesheet, script, font, and XHR the page asks for, over one or two connections, sometimes with HTTP/2 multiplexing that your proxy has to keep alive. Add a browser fingerprint that screams "automation," and the IP you burned through in a `requests` run gets flagged in Selenium after a fraction of the pages.

So the question isn't really "how do I rotate proxies in Selenium." It's how do you rotate them in a way that survives authentication, preserves the sessions your script depends on, and doesn't cost more than the data you're collecting. Those three constraints are where most tutorials stop short.

## Why a Selenium run eats IPs faster than anything else you've written

A few concrete differences matter:

**Page weight.** A browser loads assets an HTTP client skips. If you're on a per-gigabyte plan, that's the budget difference between a few thousand pages and a few hundred thousand.

**Connection reuse.** Chrome keeps a connection pool alive. If your rotation logic swaps the proxy mid-session, you're not just changing an IP, you're tearing down live connections, and pages will fail in ways that look like network errors rather than blocks.

**Fingerprint + IP pairing.** Rotating the IP every request while the browser fingerprint stays identical is a strong signal in itself. Sites that do bot scoring correlate the two; a residential IP that keeps changing while `navigator.webdriver` is `true` doesn't look more human, it looks weirder.

The practical upshot: for browser automation, **sticky sessions usually beat per-request rotation**, and unlimited-bandwidth plans usually beat per-gigabyte plans. Keep that in mind, because it determines which proxy package you should actually buy.

## The credential problem nobody warns you about first

Here's the wall everyone hits in their first hour.

python
options.add_argument("--proxy-server=http://user:pass@host:port")


Chrome ignores the credentials and keeps the host:port. It will establish the connection, get a 407, and either return a blank page or an auth prompt that Selenium can't answer. This is not a proxy provider bug; Chromium simply doesn't parse userinfo out of `--proxy-server` for HTTP proxies.

Three ways around it:

1. **IP whitelisting.** Add your server's egress IP in the provider's dashboard and connect without credentials at all.
2. **A proxy auth extension.** Generate a small Manifest V3 extension that injects the credentials in `chrome.webRequest.onAuthRequired`.
3. **selenium-wire.** A wrapper that handles `Proxy-Authorization` for you and lets you swap proxies on a live driver.

Rotating endpoint strings also matter here. Some providers accept `username:password` inline in the proxy URL in some languages and not others, so the same credentials that work in `curl` will silently fail in Chrome. Whichever method you pick, verify with a single request to an IP echo endpoint before you build rotation on top of it:

python
driver.get("https://ipinfo.io/ip")
print(driver.find_element(By.TAG_NAME, "body").text)


If that returns your own IP, the proxy isn't in use. If it returns a blank page, you've hit the auth wall.

## Four rotation patterns, ranked by how long they last in production

### 1. Random pick from a list

python
import random
proxy = random.choice(proxy_list)
options.add_argument(f"--proxy-server={proxy}")


Fine for a demo, useless at scale. You're maintaining a list of IPs by hand, you have no health checks, and dead entries fail silently 40% of the time. Free proxy lists are worse: slow, already blacklisted, and often running inspection you don't want in the middle of an authenticated session.

### 2. selenium-wire, swapping a live driver

python
from seleniumwire import webdriver

driver = webdriver.Chrome(seleniumwire_options={"proxy": {"http": ..., "https": ...}})
driver.get("https://ipinfo.io/ip")
driver.proxy = next_proxy  # swap without relaunching Chrome


Genuinely convenient, and it's the method most 2025-era tutorials recommend. Two caveats. It's a third-party wrapper, so pin your Selenium version when you upgrade. And swapping the proxy on a live driver still drops in-flight requests, so expect a retry layer regardless.

### 3. Auth extension plus a fresh driver per IP

python
def build_auth_extension(host, port, user, password, path):
    manifest = json.dumps({
        "version": "1.0.0",
        "manifest_version": 3,
        "name": "proxy auth",
        "permissions": ["proxy", "storage", "webRequest", "webRequestAuthProvider"],
        "host_permissions": ["<all_urls>"],
        "background": {"service_worker": "background.js"},
    })
    background = f"""
    chrome.runtime.onInstalled.addListener(() => {{
      chrome.proxy.settings.set({{ value: {{
        mode: "fixed_servers",
        rules: {{ singleProxy: {{ scheme: "http", host: "{host}", port: {port} }},
                  bypassList: ["localhost"] }}
      }}, scope: "regular" }});
    }});
    chrome.webRequest.onAuthRequired.addListener(
      (d, cb) => cb({{ authCredentials: {{ username: "{user}", password: "{password}" }} }}),
      {{ urls: ["<all_urls>"] }}, ["blocking"]
    );
    """
    with zipfile.ZipFile(path, "w") as z:
        z.writestr("manifest.json", manifest)
        z.writestr("background.js", background)
    return path


Reliable, and it works with any HTTP proxy. The cost is a Chrome launch per IP, which is two to three seconds of overhead plus the memory. If you're rotating every page, this is a losing trade. It's the right pattern when you need a handful of long-lived browser profiles, each pinned to one region.

### 4. A rotating gateway endpoint

Instead of collecting proxies, you point Selenium at one host:port and express the session policy in the credentials string. Rotation, geography, and stickiness all get decided server-side.

This is the pattern that scales, and it's the one worth building on. Which brings us to what a provider actually has to support for it to work in Selenium specifically.

## Wiring 9Proxy into a Selenium script

9Proxy sells residential IPs in two different billing shapes, and the difference matters for automation more than the price does.

- **Residential by IP.** You buy a pool of IP credits with unlimited bandwidth. Each IP you activate lives a few hours up to roughly 24 hours. Authentication runs through their desktop app via local port forwarding, with optional proxy authentication and a built-in auto-rotation proxy that can cycle at intervals you set on chosen ports.
- **Residential by GB.** You buy traffic, measured in gigabytes, and generate unlimited endpoints from the dashboard. No app required. Authenticate with a sub-user username/password or by whitelisting your server IP. Traffic is valid for 180 days, or without expiry on Enterprise packages.

For a Selenium script running on a Linux box, the second one is the cleaner fit. No desktop app, no port forwarding, credentials that work over `Proxy-Authorization`, and a fixed hostname you can drop into your config. The pool is advertised at 20M+ residential IPs across 90+ countries.

### Path A: IP whitelisting (best for headless servers)

Whitelist the machine's egress IP in the dashboard, then skip credentials entirely:

python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--proxy-server=http://your_9proxy_host:your_port")
options.add_argument("--headless=new")
driver = webdriver.Chrome(options=options)
driver.get("https://ipinfo.io/ip")


Nothing to leak into a shell history, nothing to rotate in code. On a fixed-IP cloud instance this is the least fragile setup.

### Path B: Sub-user credentials, with or without an extension

Create a sub-user, assign it traffic, and build a username that encodes exactly which IP you want. This is where the design gets useful for Selenium.

The username format documented by 9Proxy is:


<subuser>-country-<cc>-st-<state>-city-<city>-isp-<isp_code>-sst-<minutes>-ssid-<session_id>


Every parameter after the sub-user name is optional, and the ones you include define your session behavior.

**Per-request rotation** (fresh IP on every request) is just the sub-user plus optional geo targeting:


myuser-country-us
myuser-country-us-city-newyork


**Sticky sessions** add `sst`, in minutes:


myuser-country-us-sst-10


**Parallel sticky sessions** add a distinct `ssid` per browser instance, so ten Selenium workers can each hold their own IP from the same configuration:


myuser-country-us-sst-10-ssid-worker01
myuser-country-us-sst-10-ssid-worker02


That `ssid` parameter is the piece most rotation guides don't cover, and it's the difference between running ten parallel Chrome instances with ten distinct sticky IPs and running ten instances that randomly share and thrash the same ones. If you're mirroring browser profiles or running parallel account workflows, this is what you want.

A quick sanity check before you wire it into your scraper:

python
from seleniumwire import webdriver

user = "myuser-country-us-city-newyork-sst-10-ssid-worker01"
driver = webdriver.Chrome(seleniumwire_options={
    "proxy": {"http": f"http://{user}:{password}@host:port",
              "https": f"http://{user}:{password}@host:port"}
})
driver.get("https://ipinfo.io/json")


Run it twice with the same `sst` and `ssid` and you should see the same IP for the whole window. Change the `ssid` and you should see a different one.

### Geo-targeting without a second vendor

Because targeting lives in the username, you can switch countries per worker without changing infrastructure:

python
regions = {"us": "myuser-country-us-sst-15-ssid-a1",
           "de": "myuser-country-de-sst-15-ssid-a2",
           "vn": "myuser-country-vn-city-hanoi-sst-15-ssid-a3"}


One caveat worth internalising: narrowing by state, city, and ISP together shrinks the available pool and slows down IP assignment. Target country first; only add city or ISP if the task actually needs it.

> Sticky sessions are what browser automation usually wants. Per-request rotation looks impressive in a tutorial and breaks the moment your script has to log in, keep a cart, or stay in the same funnel for more than one page load.

## Headless servers: the four things that actually break

**Auth extensions and `--headless=new`.** Chrome's modern headless mode handles extensions, but service workers can fail quietly, and you'll get an empty page with no error. If you must use credential-based auth via extension, run under `Xvfb` instead of headless:

bash
xvfb-run --auto-servernum python scraper.py


Since 9Proxy's GB-based product supports IP whitelisting and plain username/password, you can sidestep this class of problem entirely on servers.

**Zombie Chrome processes.** Missing `driver.quit()` on an exception path leaves Chromes running. One overnight run can leave 200 of them holding your RAM. Wrap in `try/finally`, or register `atexit`, and hit both.

**Page load timeout is not proxy timeout.** `driver.set_page_load_timeout(30)` measures the whole page. If the proxy is dead, you wait the full 30 seconds to learn that. Probe the proxy with a cheap `requests.get` against an IP echo endpoint first, five-second timeout, and only launch Chrome if it answers.

**Detection signals.** `navigator.webdriver`, a missing `window.chrome.runtime`, a headless user agent, WebGL renderer strings. A residential IP won't save you if the browser announces itself. Clean your flags before blaming the proxy.

## Which 9Proxy package fits a Selenium workload

Two facts drive the choice.

First, **Selenium is bandwidth-hungry.** Full-page loads with images and JS burn gigabytes quickly. On a per-GB plan, 100 GB at $150 isn't much if you're loading real pages rather than hitting JSON endpoints.

Second, **browser workflows want stable IPs.** Unless you're scraping static pages with no session state, per-request rotation is usually the wrong default.

That points most Selenium users at either the IP-based plans, where bandwidth is unlimited while an IP is active, or sticky GB sessions. Per-request rotation only makes sense for stateless, lightweight automation.

9Proxy raised prices on IP-based and Bundle packages on June 1, 2026, its first adjustment since launch. GB-based pricing didn't change. If you've read a review quoting $20 for 100 IPs, that's a pre-adjustment number.

**Residential by IP (one-off balance purchase, bandwidth unlimited, unused IP credits don't expire)**

| Package | Price (USD) | Effective per IP | Buy |
| --- | --- | --- | --- |
| 100 IPs | $24 | $0.24 | [ Start with 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | $72 | $0.144 | [ Get the 500 IP package](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | $126 | $0.084 | [ Claim 1,500 IPs](https://bit.ly/9-Proxy) |
| 2,500 IPs | $210 | $0.084 | [ Check the 2,500 IP price](https://bit.ly/9-Proxy) |
| 5,000 IPs | $360 | $0.072 | [ View the 5,000 IP plan](https://bit.ly/9-Proxy) |
| 15,000 IPs | $720 | $0.048 | [ See the 15,000 IP tier](https://bit.ly/9-Proxy) |
| 25,000 IPs | $863 | $0.035 | [ Check the 25,000 IP tier](https://bit.ly/9-Proxy) |
| 50,000 IPs | $1,438 | $0.029 | [ View the 50,000 IP plan](https://bit.ly/9-Proxy) |
| 100,000 IPs (Business) | $2,300 | $0.023 | [ See Business pricing](https://bit.ly/9-Proxy) |
| 200,000 IPs (Business) | $4,140 | $0.021 | [ Check Business volume rates](https://bit.ly/9-Proxy) |
| 500,000 IPs (Business) | $8,625 | $0.018 | [ View large-volume pricing](https://bit.ly/9-Proxy) |

**Residential by GB (traffic-based, 180-day validity unless noted)**

| Package | Price (USD) | Effective per GB | Buy |
| --- | --- | --- | --- |
| 5 GB | $15 | $3.00 | [ Try the 5 GB package](https://bit.ly/9-Proxy) |
| 50 GB + 5 bonus | $105 | $2.10 | [ Grab the popular 55 GB pack](https://bit.ly/9-Proxy) |
| 100 GB | $150 | $1.50 | [ Check the 100 GB price](https://bit.ly/9-Proxy) |
| 200 GB | $200 | $1.00 | [ View the 200 GB plan](https://bit.ly/9-Proxy) |
| 1,000 GB | $800 | $0.80 | [ See the 1 TB package](https://bit.ly/9-Proxy) |
| 2,000 GB | $1,500 | $0.75 | [ Check the 2 TB tier](https://bit.ly/9-Proxy) |
| 3,000 GB (Enterprise, no expiry) | $2,160 | $0.72 | [ View Enterprise GB pricing](https://bit.ly/9-Proxy) |
| 6,000 GB (Enterprise, no expiry) | $4,200 | $0.70 | [ See Enterprise 6 TB](https://bit.ly/9-Proxy) |
| 10,000 GB (Enterprise, no expiry) | $6,800 | $0.68 | [ Check Enterprise 10 TB](https://bit.ly/9-Proxy) |

**Bundle packages (IPs plus bandwidth, 180-day traffic validity)**

| Bundle | Contents | Price (USD) | Buy |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [ Start with the Starter bundle](https://bit.ly/9-Proxy) |
| Growth / Popular | 1,500 IPs + 50 GB | $180 | [ Get the Growth bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 (listed at $860, 16.28% off) | [ View the Pro bundle](https://bit.ly/9-Proxy) |

A sanity check on the arithmetic: if you're loading heavy pages, 5 GB disappears fast. If your Selenium jobs are mostly DOM scraping on lightweight pages, the Starter bundle is a legitimate way to test the setup for $30. If you're running sticky sessions across a handful of parallel Chrome instances, 100 IPs and unlimited bandwidth goes further than any GB pack at a similar price.

One more thing worth knowing: 9Proxy's affiliate terms state that users who sign up through a referral link get 5% off, and their own forum posts describe limited trials for new users depending on availability. The link in this article carries an invite code, so a new account should pick up that discount.

## Quick answers

**Does Selenium support proxy authentication directly?** No, not via `--proxy-server`. Use IP whitelisting, an auth extension, or selenium-wire.

**Should I rotate per request or use sticky sessions?** For browser automation, sticky. Per-request rotation fits stateless HTTP clients and lightweight page fetches.

**Why do I get the same IP as my own machine?** The auth is failing silently. Check the IP echo endpoint and confirm the extension or credentials are actually applied.

**Can I run parallel Selenium workers on separate IPs?** Yes, with a distinct `ssid` per worker on sticky sessions, or with separate IP-based ports.

**Is per-GB or per-IP cheaper for Selenium?** It depends entirely on page weight and how long each session needs to live. Heavy pages and long sessions favour per-IP; light pages and short bursts favour per-GB.

Third-party reviews put the network around 99.5% success with roughly 0.6-second average response in one hands-on test, which is in line with what you'd expect from a mid-market residential pool. Ratings aggregators place it around 3.9 stars. Judge it on your own targets after the first load test, not on a review score.
