# sticky proxies: how long the IP actually holds, when sticky beats rotating, and how to set one up in minutes

People who search for "sticky proxies" are usually halfway through a problem, not shopping for a category. The script logged in fine and got bounced on the second page. The marketplace account got flagged after forty requests. The checkout test failed because the cart cookie arrived from one IP and the payment step left from another. None of that is a bandwidth problem. It's session continuity, and a sticky session is the usual fix — when it's configured correctly.

Below: what a sticky session really does, how long the IP holds, how the parameters are built on a pay-as-you-go provider, and the places where a perfectly configured sticky setup still falls over.

## What "sticky proxy" actually means

A sticky proxy holds one exit IP for a defined window. Every request inside that window leaves from the same address, so the site sees one consistent visitor instead of a hundred unrelated hits. When the window expires, the address goes back into the pool and you get a different one.

Rotating is the opposite mode: a fresh exit on every request, or on a short timer. Both come out of the same pool, and most providers sell them as two endpoints rather than two products.

Worth clearing up early, because it causes real money to be spent badly: **sticky is not the same as static.** A sticky session rents an address from a rotating residential pool for a few minutes. A static (ISP) proxy is an address you keep for months. If an account needs one IP in June and the same IP in September, sticky sessions won't get you there — and DataImpulse doesn't sell ISP proxies at all, only residential, mobile, datacenter, and premium residential pools. Choose the right tool before you argue about configuration.

## Why sites notice when the exit IP moves mid-visit

The reason has little to do with IP reputation scores and everything to do with state the site handed you on the way in.

A page load isn't one request. It's a document, then dozens of subresources, XHR calls, and often a WebSocket — all from the same tab, all tied together by three things:

- **Cookies.** The session cookie set on the first response gets presented on every request after it. The site issued that cookie to whoever arrived on the first IP.
- **Session tokens.** CSRF tokens, auth tokens, server-side session IDs — all bound to the context they were minted in.
- **The Referer chain.** Each in-site navigation carries the previous URL, so the requests form an ordered path rather than a set of unrelated hits.

Rotate the exit in the middle of that sequence and you hand the site a visitor whose IP contradicts their own cookies. Nothing about the browser changed, but the story the requests tell stopped making sense — and that contradiction is cheaper for a detection system to spot than any fingerprinting trick.

There's a second layer most people miss: the *header profile*. User-Agent, Accept-Language, sec-ch-ua-* need to stay constant for the whole session. A session that arrives as Chrome 108 on one request and Chrome 124 on the next is suspicious even before anyone knows it's automated.

## How long do sticky sessions last?

Providers quote numbers from ten minutes to twenty-four hours, so "sticky" describes a mechanism, not a duration. DataImpulse's documentation puts its sticky connections at **1 to 120 minutes, with 30 minutes as the default**, while its standard rotating endpoints sit on ports 823 (HTTPS) and 824 (SOCKS5). Purpose-built sticky connections use ports in the 10000–20000 range.

There's also a hard limit that isn't a policy: residential IPs come from real devices. If the device behind your session goes offline, the provider swaps in another available address. That's why nobody honest guarantees a specific sticky IP.

Practical rule: pick a session length that covers your longest realistic user flow, not the maximum the provider allows. A 120-minute session on a task that finishes in 90 seconds burns nothing but exposure.

## Sticky vs rotating, by task

| What you're doing | Session type | Why |
| --- | --- | --- |
| Login flows, account checks, checkout tests | Sticky | Session cookies and tokens must stay tied to one exit |
| Multi-account management | Sticky, one tag per account | A mid-session IP change is the classic flag |
| Price monitoring on one product page | Rotating | Each fetch is independent; volume matters more |
| SERP scraping at scale | Rotating | No state carries between requests |
| Ad verification from a specific city | Sticky | The creative needs a consistent local identity |
| Bulk data collection behind soft rate limits | Rotating with sane retry logic | Spread volume across many exits |

If you can't say which of those your job resembles, watch one manual run in your browser's network tab and see whether a cookie or token gets re-sent. That answers it in two minutes.

## Setting up a sticky session: the actual parameters

DataImpulse keeps everything on one endpoint and encodes targeting and session behaviour in the proxy username. The shape is the same for residential, mobile, and datacenter pools:


host:     gw.dataimpulse.com
port:     823 (HTTPS) or 824 (SOCKS5)
username: YOUR_USERNAME__cr.us;sessid.my-session
password: YOUR_PASSWORD


Two documented ways to pin an IP:

- **`sessid`** — tag the session. Reuse the same value and you're routed to the same address for roughly 30 minutes. The value can be any string or number; the IP behind a given tag stays consistent for that window. It's an alternative to a dedicated sticky port when you just want repeat requests on one address.
- **`sessttl`** — set the rotation interval in minutes on a sticky connection. `sessttl.60` rotates hourly, `sessttl.1` rotates every minute. Leave it off and the default 30-minute window applies.

Country targeting rides in the same string as `cr.` plus the two-letter code:

bash
curl -x "http://USERNAME__cr.us;sessid.my-session:PASSWORD@gw.dataimpulse.com:823" https://api.ipify.org/


And in Python, without changing anything else in your client:

python
import requests

proxy = "http://USERNAME__cr.us;sessid.my-session:PASSWORD@gw.dataimpulse.com:823"
r = requests.get("https://example.com/account", proxies={"http": proxy, "https": proxy}, timeout=30)
print(r.status_code)


Two habits that save time. First, drop the session tag to get rotating behaviour — you don't need a second endpoint or a second account. Second, the exact parameter names your account accepts are listed in the DataImpulse dashboard; copy them from there rather than from a blog post, including this one.

If you want to test this without committing budget, 👉 [grab 5 GB of residential traffic for $5](https://bit.ly/dataimPulse) — the balance doesn't expire, so a short experiment doesn't turn into a subscription you forget about.

## Where sticky setups still fall apart

Getting the IP to hold is step one. These five mistakes account for most "sticky proxies don't work" threads:

1. **Rotating mid-login.** The usual cause is a session tag generated per request instead of per job. Derive the tag from something stable — job ID plus target domain — not from a random number each time.
2. **Changing headers mid-session.** IP stays put, User-Agent jumps from desktop Chrome to a Python default. That mismatch is enough to block you. One session, one header profile.
3. **Retrying instantly.** A failed request followed within milliseconds by another from the same origin reads as a bot. Change one variable, wait, then retry.
4. **Jumping countries between sessions.** Going from Tokyo to Toronto on consecutive sessions looks like a VPN switch. If you rotate, rotate within the same geo or ASN cluster.
5. **Ignoring targeting costs.** On standard residential, city, state, ZIP, and ASN filters are billed at a higher rate — reported as double the base per-GB price. Country targeting is included. A city-level sticky workflow can cost twice what you budgeted.

## What sticky sessions cost on a pay-as-you-go provider

This is where DataImpulse's model is genuinely different from the subscription crowd. It charges by traffic with no monthly minimum, and unused GB never expire — so a project that runs for two weeks and then goes quiet for three months isn't paying for idle access.

Entry pricing starts at **$5**, which buys 5 GB of residential traffic, 10 GB of datacenter, 2.5 GB of mobile, or 1 GB of premium residential. Country targeting is included in the base rate. There's a 7-day money-back guarantee on Intro plans for card payments, provided less than 80% of the traffic has been consumed; crypto purchases on Intro plans are non-refundable. DataImpulse's own pricing write-up cites a 99.51% published success rate, and the site advertises a 90M+ ethically sourced IP pool across 195 countries.

## All current DataImpulse plans

| Proxy type | Smallest top-up | Price per GB | Volume tiers | Billing model | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 / 5 GB | $1/GB | 1 TB at $800 ($0.80/GB) | Pay-as-you-go, no subscription | [Start with 5 GB for $5](https://bit.ly/dataimPulse) |
| Datacenter | $5 / 10 GB | $0.50/GB | $50 / 100 GB, $450 / 1 TB ($0.45/GB), custom from $2,250 for 5 TB+ | Pay-as-you-go, traffic doesn't expire | [Compare the datacenter plans](https://bit.ly/dataimPulse) |
| Mobile (5G/4G/LTE) | $5 / 2.5 GB | $2/GB | $50 / 25 GB, $1,600 / 1 TB ($1.60/GB), custom from $8,000 for 5 TB+ | Pay-as-you-go, traffic doesn't expire | [Check the mobile proxy pricing](https://bit.ly/dataimPulse) |
| Premium residential | $5 / 1 GB | $5/GB | $50 / 10 GB, custom from $20,000 for 5 TB+ | Pay-as-you-go, dedicated account manager | [Look at the premium residential pool](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

A few notes on reading that table. The 20% volume discount on residential and mobile kicks in at the 1 TB tier, not before — from roughly 5 GB up to a few hundred GB the residential rate stays flat at $1/GB. Datacenter is the cheapest per gigabyte but its IPs carry weaker trust signals, so it's a poor choice for login flows even though it works fine for speed-focused public-page tasks. Premium residential adds faster connection speeds, tighter geo-targeting, and all targeting options at no surcharge, plus a dedicated account manager.

## Which plan you actually want for sticky work

If your sticky sessions are for account flows, carts, or anything behind a login, **standard residential at $1/GB is the sensible starting point**. It's the cheapest tier with household IPs, and at $5 to test there's very little downside to measuring your own cost per successful request before scaling.

Move to **premium residential** when your targeting is granular enough that the surcharge would bite anyway, or when failed requests cost more than the traffic does — enterprise-scale automation, sensitive verification work, anything where a block ruins a run. At $5/GB it's five times the standard rate, so it needs a reason.

**Datacenter** is for throughput. **Mobile** is for the hardest targets — social platforms, native apps, anything that treats mobile carrier IPs with more trust. Neither is a default for sticky work, but both support sticky sessions according to DataImpulse's documentation, which means you can keep the same session logic and just change the pool.

## Quick answers

**Do sticky sessions use more traffic than rotating ones?** No. You're billed per gigabyte either way. Sticky sessions can be slightly more efficient because fewer requests fail and get retried, which matters more than any rate difference.

**Can I get a specific IP address on demand?** Not guaranteed. A session tag routes you consistently to one address for its window, but residential IPs come from real devices that go offline. If that address drops, you're moved to another available one.

**How many sticky sessions can I run at once?** Each session tag is its own session, so parallel jobs need distinct tags. Parallel sessions can't share an IP — the same address won't be assigned to two tags simultaneously.

**Is a sticky session enough to avoid blocks on its own?** No. It fixes consistency, not reputation. A clean residential IP that keeps a coherent cookie/header profile usually gets through; the same IP with a mismatched fingerprint still gets flagged. If you're weighing whether to start at all, 👉 [running a 5 GB test on residential traffic](https://bit.ly/dataimPulse) costs less than an hour of debugging against a proxy that keeps changing under you.
