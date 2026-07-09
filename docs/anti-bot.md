# Anti-bot detection and how Kewpie handles it

This document explains the 2026 anti-bot detection landscape, why a naive
Python scraper is blocked before it sends a single header, what Kewpie does
about it, and where Kewpie deliberately stops. It is the design rationale
behind the escalation ladder.

## Why `requests` / `httpx` gets blocked instantly

The most common failure mode is not a missing header or a bad User-Agent. It
is the TLS handshake. A modern WAF fingerprints the TLS ClientHello (cipher
suite list and order, extensions and order, supported curves, GREASE) and the
HTTP/2 SETTINGS frame before your application-layer request is ever read. A
stock Python client built on OpenSSL produces a fixed, non-browser fingerprint
that these systems recognize immediately. No amount of header spoofing helps,
because the handshake happens first.

This is the "simple blocker" that stops most home-grown scrapers.

## The four-layer detection stack

Detection is layered, and each layer is checked independently, so defeating one
leaves the others exposed.

1. **TLS fingerprinting (JA3 / JA4).** The ClientHello. JA3 is fading because
   Chrome randomizes its TLS extension order (since v110), which makes the JA3
   hash unstable; JA4 (which sorts before hashing) is the 2026 standard, adopted
   by Cloudflare, Akamai and AWS WAF.
2. **HTTP-layer fingerprinting.** The HTTP/2 SETTINGS + WINDOW_UPDATE + PRIORITY
   frames and pseudo-header order (the "Akamai fingerprint"), plus regular
   header ordering and the presence/order of Client Hints (`sec-ch-ua`,
   `sec-ch-ua-platform`, `sec-ch-ua-mobile`). Browsers emit a fixed order;
   libraries do not.
3. **Behavioral analysis.** Mouse paths, scroll cadence, click timing, request
   velocity. A client that loads a page and immediately extracts looks nothing
   like a person.
4. **IP reputation.** Datacenter vs residential vs mobile ASNs, per-IP velocity,
   and cross-network fingerprint velocity (one JA4 seen across thousands of IPs
   in minutes is itself a bot signal, so rotating IPs without rotating
   fingerprints backfires).

The single most avoidable own-goal spans layers 1 and 2: a Chrome User-Agent
sent over an OpenSSL TLS handshake. The story does not cohere, and the mismatch
is trivially detectable. `kewpie doctor` exists to catch exactly this.

## What Kewpie does

Kewpie does not invent bypasses. Hand-maintained bypasses against Kasada or the
latest DataDome break within days; that is a losing maintenance game. Instead
Kewpie orchestrates best-of-breed open-source tools and adds the layer most of
them lack.

- **Layer 1 and 2** are handled by `curl_cffi`, which reproduces a real
  browser's TLS and HTTP/2 fingerprint. Kewpie pairs each impersonation target
  with a coherent identity (matching User-Agent, Client Hints, Accept-Language)
  so the whole request tells one consistent story, and keeps that identity
  sticky per host so cookies and bot-manager scoring survive.
- **JS challenges** (Cloudflare Turnstile, DataDome interstitials, client-side
  rendering) are handled by escalating to a real headless browser. nodriver is
  the default backend because it drives system Chrome over plain CDP with no
  Selenium or Playwright in the loop, which independent benchmarks identify as
  the automation-protocol signal current gates key on. Camoufox (a Firefox fork
  that spoofs fingerprints at the engine level) is an alternative behind the
  same interface.
- **Layer 4** is minimized, not solved: coherent per-identity proxy pairing
  avoids the "one fingerprint from many IPs" tell, and conservative
  rate-limiting keeps per-IP velocity low. Clean residential IPs remain a paid
  problem Kewpie does not pretend to solve.

## The escalation ladder

Rather than always using the heaviest tool, Kewpie runs the cheapest viable
tier first and escalates only on evidence:

```
cheap HTTP  ->  curl_cffi impersonation  ->  headless browser
```

A structured challenge classifier (`kewpie.challenge.classify_challenge`)
decides when to move up, based on status codes, challenge headers, active
challenge cookies, Turnstile/reCAPTCHA script markers, and tiny-shell /
empty-body heuristics. Kewpie records which tier finally worked per host and
starts there next time, and periodically probes one tier lower because WAF
posture relaxes over time. This makes the common case (most pages need no
browser) fast, and reserves the expensive, more-detectable browser tier for the
pages that actually require it.

## The OSS landscape (2026)

Kewpie delegates to these; it does not reimplement them.

- **HTTP impersonation:** `curl_cffi` (default), `primp` (Rust `rquest`),
  `tls-client` (Go). All reproduce TLS + HTTP/2 fingerprints.
- **Headless stealth browsers:** `nodriver` (successor to
  undetected-chromedriver; best independent-benchmark result against
  Cloudflare), `Camoufox` (Firefox, engine-level fingerprint spoofing),
  `SeleniumBase` UC mode (the most reliable free Turnstile-click option),
  `Patchright` (Playwright CDP-leak patches). `undetected-chromedriver` is
  effectively superseded.

## Where Kewpie stops (by design)

Kewpie does not solve CAPTCHAs, defeat behavioral or heavily obfuscated
protection (Kasada, aggressive DataDome), or scrape behind-login content. When a
site needs that, the fetch result carries a `Verdict` naming the vendor and
challenge kind so you can route it to a dedicated solver or a paid unblocker.
This is a deliberate boundary, not a gap.

## Sources

The design draws on current (2026) public research and primary specifications:

- Cloudflare, JA3/JA4 fingerprint documentation.
- FoxIO, the JA4 specification.
- Akamai, "Passive Fingerprinting of HTTP/2 Clients" (Black Hat EU 2017).
- Independent anti-detect browser benchmarks (Ian L. Paterson's 31-target
  Cloudflare sweep; `techinz/browsers-benchmark`).
- Project repositories: `lexiforest/curl_cffi`, `ultrafunkamsterdam/nodriver`,
  `daijro/camoufox`, `deedy5/primp`, `bogdanfinn/tls-client`.
