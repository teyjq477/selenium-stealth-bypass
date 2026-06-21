# Selenium Avoid Detection: How Does Bot Fingerprinting Work, Why Does Your Scraper Keep Getting Blocked, and What Actually Fixes It? (Complete Anti-Bot Bypass Guide with Proxy, Stealth Mode & API Solutions)

So you built a Selenium scraper. It ran great on your local machine. You were feeling pretty good about yourself. Then you deployed it and within ten minutes it was getting 403s, CAPTCHAs, and the occasional "Access Denied" page that looks like the internet is personally offended by you.

Welcome to the club. This is the wall every web scraper hits eventually.

The good news is that getting blocked by anti-bot systems is not random bad luck — it follows patterns, and once you understand those patterns, you can do something about them. This guide walks through everything: how detection actually works, what techniques help you avoid it, and when it makes more sense to stop fighting the detection arms race and just use a purpose-built tool like 👉 [ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons).

---

## How Websites Detect Selenium (And Why the Standard Driver Is a Red Flag)

Before you can avoid detection, it helps to know what you're being detected *by*.

When Chrome launches under standard Selenium WebDriver, it quietly sets a JavaScript property called `navigator.webdriver` to `true`. This one flag alone is enough for any half-decent anti-bot system to know you're running automation. It's the equivalent of walking into a casino wearing a t-shirt that says "I'm counting cards."

But that's just the start. Modern bot detection systems — Cloudflare, Akamai, PerimeterX, DataDome — look at a whole stack of signals:

- **navigator.webdriver flag**: Is it `true`? Automation detected, goodbye.
- **TLS/JA3 fingerprint**: ChromeDriver-controlled Chrome produces a subtly different TLS handshake than a user-launched Chrome. Advanced systems catch this.
- **Browser fingerprint consistency**: Does your user-agent say you're Chrome 124 on Windows, but your screen resolution is 0x0 and you have zero plugins? Suspicious.
- **Behavioral patterns**: Real users scroll, pause, move their mouse around. Bots tend to hit form fields and click buttons in millisecond-perfect sequences. 
- **IP reputation**: If the same datacenter IP is hammering a site with 500 requests per minute, it's not a human.
- **Headless mode signals**: Running `--headless` leaks specific JavaScript properties. Some systems detect it through rendering differences.
- **CDP monitoring**: Chrome DevTools Protocol commands on port 9222 can be spotted by sites monitoring unusual API call patterns.

The anti-bot industry has gotten genuinely sophisticated. A single technique isn't going to cut it. You need layers.

---

## The Core Techniques to Avoid Detection in Selenium

### 1. Kill the `navigator.webdriver` Flag

The most basic fix. You can disable it with ChromeOptions before launching your driver:

python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)
options.add_argument("--disable-blink-features=AutomationControlled")

driver = webdriver.Chrome(options=options)


This masks the most obvious giveaway. It's table stakes — not a complete solution, but you need it regardless.

---

### 2. Use Undetected ChromeDriver

`undetected-chromedriver` (often abbreviated `uc`) is a patched version of ChromeDriver specifically designed to remove automation signals. It modifies how ChromeDriver injects its automation properties, making it much harder for sites to fingerprint.

Install it:

bash
pip install undetected-chromedriver


Use it like a drop-in replacement for Selenium:

python
import undetected_chromedriver as uc

options = uc.ChromeOptions()
driver = uc.Chrome(options=options)
driver.get("https://your-target-site.com")


It handles a lot of the fingerprint patching automatically. For many sites, this alone can get you past Cloudflare challenges that standard Selenium trips on immediately.

The caveat: sophisticated anti-bot systems have caught up to `undetected-chromedriver` too. It's a moving target. You'll want to combine it with other techniques.

> **Note on headless mode**: If you must run headless, use `--headless=new` (available in Chrome 112+) instead of `--headless`. The newer flag provides a more complete browser environment that's significantly harder to detect than the legacy headless mode.

---

### 3. Add `selenium-stealth` for Deeper Fingerprint Patching

`selenium-stealth` takes fingerprint patching further. It modifies JavaScript properties like `navigator.webdriver`, plugin lists, language settings, and renderer info to match what a real browser session would expose.

bash
pip install selenium-stealth


python
from selenium import webdriver
from selenium_stealth import stealth

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")

driver = webdriver.Chrome(options=options)

stealth(driver,
    languages=["en-US", "en"],
    vendor="Google Inc.",
    platform="Win32",
    webgl_vendor="Intel Inc.",
    renderer="Intel Iris OpenGL Engine",
    fix_hairline=True,
)

driver.get("https://your-target-site.com")


The key is consistency. Your language, timezone, screen resolution, and platform should all tell the same story. A mismatch anywhere is a fingerprint leak.

---

### 4. Rotate User Agents

Sending the same user agent string on every request is a pattern. Real users on Chrome update their browser; they use different devices. Your scraper should reflect that.

python
import random

user_agents = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
]

options = webdriver.ChromeOptions()
options.add_argument(f"--user-agent={random.choice(user_agents)}")


Keep your user agents current. A list full of Chrome 89 strings in 2025 raises flags — those browser versions are long past their usage peak in the wild.

---

### 5. Simulate Human-Like Behavior

This is the one people underestimate. Anti-bot behavioral analysis doesn't just look at what you're doing — it looks at *how* you're doing it.

Real users:
- Move the mouse before clicking
- Scroll through pages before extracting data
- Don't submit forms in 12 milliseconds
- Pause between page loads

python
import time
import random
from selenium.webdriver.common.action_chains import ActionChains

# Random delay between actions
def human_delay(min_sec=0.8, max_sec=3.0):
    time.sleep(random.uniform(min_sec, max_sec))

# Scroll the page before interacting
driver.execute_script("window.scrollTo(0, document.body.scrollHeight / 2);")
human_delay()

# Move mouse to element before clicking
element = driver.find_element("css selector", "#target")
ActionChains(driver).move_to_element(element).pause(random.uniform(0.3, 0.8)).click().perform()
human_delay()


None of this is glamorous. But it's often the difference between a scraper that works and one that gets killed.

---

### 6. Rotate Proxies

IP reputation is a major detection vector. A datacenter IP sending hundreds of requests to the same domain is an obvious signal. Even if every other technique is perfect, a flagged IP gets you nowhere.

You need proxy rotation. The options, roughly in order of quality:

- **Free proxies**: Avoid. They're burned, shared by thousands of users, and typically detected instantly.
- **Datacenter proxies**: Better, but still fairly easy for advanced systems to identify. Fine for sites with basic protection.
- **Residential proxies**: IPs from real ISPs and real users. Much harder to flag. More expensive.
- **Mobile proxies**: The gold standard for avoiding detection. Actual mobile carrier IPs. Very rarely blocked.

The problem with managing proxies yourself is the overhead: buying, validating, rotating, and replacing them as they get burned. For serious projects, this is where managed services earn their keep.

---

## When DIY Anti-Detection Becomes a Full-Time Job

Here's the honest reality: the anti-bot arms race never stops. Cloudflare releases a new detection layer, undetected-chromedriver gets updated, you patch your setup, repeat. If your core job is collecting data, spending a quarter of your engineering time playing cat-and-mouse with bot detection is a poor trade.

This is exactly the problem 👉 [ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) was built to solve.

ScraperAPI handles proxy rotation, CAPTCHA solving, browser fingerprinting, and anti-bot bypass as a managed service. Instead of maintaining a fragile stack of undetected-chromedriver + selenium-stealth + rotating proxies + CAPTCHA services, you send a single API call and get back the rendered HTML.

python
import requests

payload = {
    'api_key': 'YOUR_API_KEY',
    'url': 'https://your-target-site.com',
    'render': 'true'  # enables JS rendering
}

response = requests.get('https://api.scraperapi.com/', params=payload)
print(response.text)


That's it. ScraperAPI handles everything else: rotating through its pool of 40M+ residential proxies, managing browser fingerprints, dealing with Cloudflare and similar systems, and automatically retrying failed requests.

For Selenium users specifically, ScraperAPI offers a proxy mode that lets you route your existing Selenium sessions through ScraperAPI's infrastructure — so you keep your existing automation logic while offloading the detection problem.

---

## ScraperAPI Plans: Pricing Breakdown

ScraperAPI offers a 7-day trial with 5,000 API credits and no credit card required. Here's the full breakdown of available plans:

| Plan | Monthly Price | Annual Price | API Credits | Concurrent Threads | Geotargeting | Key Features |
|---|---|---|---|---|---|---|
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only | JS rendering, premium proxies, CAPTCHA handling |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only | All Hobby features |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global | Unlimited analytics history |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** | $475/mo | $427.50/mo | 5,000,000 | 200 | Global | Pay-as-you-go, unlimited analytics |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** | $975/mo | $877.50/mo | 10,500,000 | 300 | Global | Priority support, PAYG |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global | Priority routing, PAYG |  [Start Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22M+ | 500+ | Global | Dedicated support, Slack channel, custom pricing |  [Contact Sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

All plans include: JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom header support, CAPTCHA & anti-bot detection, custom session support, desktop & mobile user agents, automatic retries, unlimited bandwidth, and 99.9% uptime guarantee.

Annual billing saves 10% across all paid plans. Plans from Scaling upward include pay-as-you-go so you don't get cut off if you exceed your credit limit mid-project.

---

## Comparing Your Options: DIY vs. Managed API

| Approach | Setup Time | Maintenance | Cost | Success Rate on Hard Sites |
|---|---|---|---|---|
| Basic Selenium + ChromeOptions | 1 hour | High | Low | Low |
| undetected-chromedriver | 2–3 hours | Medium-High | Low-Medium | Medium |
| selenium-stealth + proxies | 1–2 days | High | Medium | Medium-High |
| Full DIY stack (all techniques) | 1 week+ | Very High | High | High (but fragile) |
| ScraperAPI managed service | 30 minutes | Very Low | Starts $49/mo | Very High |

The DIY stack isn't wrong — for simple targets, some combination of the techniques above will work fine. But as you move to more protected targets (Cloudflare Enterprise, DataDome, PerimeterX), the DIY path gets exponentially harder and more expensive in engineering time.

---

## Practical Tips That Don't Fit Neatly Into Categories

A few things worth knowing that often get left out of guides:

**Manage cookies across sessions.** By default, each new Selenium session starts fresh — no cookies, no history. Real users have both. Persisting cookies from a real browsing session (or simulating a login flow before scraping) makes your sessions look far more legitimate.

**Follow the page flow.** Don't jump directly to the internal data page. Load the homepage, navigate through categories, then reach your target. Sites track navigation paths and direct hits to deep pages are a signal.

**Check your TLS fingerprint.** If you're still getting blocked after everything else, visit `https://tls.peet.ws` through your scraping setup. The TLS handshake your browser produces when controlled by ChromeDriver is subtly different from a normal Chrome session. This is a hard problem to fix without using a managed proxy service.

**Watch your concurrency.** Even with perfect fingerprints, hammering a site with 50 concurrent requests from the same proxy looks inhuman. Spread your load. Respect rate limits.

**Test against a target before scaling.** What works on one site may be instantly detected on another. Build in detection testing before you scale a new scraper.

---

## When to Give Up on Selenium for Detection Avoidance

Some sites are simply hardened to the point where Selenium-based approaches — even fully fortified ones — aren't practical. If you're hitting:

- Cloudflare Enterprise with turnstile challenges
- DataDome on major e-commerce platforms  
- PerimeterX on well-funded retail sites

...the engineering cost to maintain a working DIY setup often exceeds the cost of a managed API.

For those situations, routing through 👉 [ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) gives you access to infrastructure that's specifically tuned against these systems, handles the cat-and-mouse updates for you, and has a 99.9% uptime guarantee backing it.

The free trial (5,000 API credits, no credit card) is worth testing against your hardest targets just to get a baseline on what a managed solution can actually do.

---

## Summary: What to Try First, What to Try Next

If you're hitting detection issues with Selenium, work through this in order:

1. **Disable the automation flag** with `excludeSwitches` and `--disable-blink-features=AutomationControlled`
2. **Switch to `undetected-chromedriver`** for automatic fingerprint patching
3. **Add `selenium-stealth`** for deeper JavaScript property masking
4. **Rotate your user agents** and keep them current
5. **Add human-like delays and behavior** — scrolling, mouse movement, random pauses
6. **Use rotating proxies** — residential if you're hitting sites with serious bot protection
7. **Persist cookies** and follow realistic page navigation flows
8. **If you're still blocked at scale**, consider routing through ScraperAPI to offload the entire detection problem

Each layer adds to your stealth profile. Most scrapers will work fine with steps 1–4 on moderately protected sites. For the hard targets, you'll need all of them — or a managed API that has already solved this problem at scale.

---

*All ScraperAPI pricing information is current as of the publication date. ScraperAPI's free trial includes 5,000 API credits with no credit card required.*
