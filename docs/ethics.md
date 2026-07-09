# Responsible use

Kewpie is built for collecting public data politely. Its defaults are
conservative on purpose, and staying within them also keeps you off the
Layer-4 velocity flags described in `anti-bot.md`: politeness and stealth align.

## Defaults

- **robots.txt is honored** in the plain web-scraping mode (via protego). RSS
  and official APIs are sanctioned interfaces and are not gated, but Kewpie
  still rate-limits them.
- **Rate limits are low** (well under one request per second per host by
  default). Raise them only when you know a target's policy and have permission.
- **A descriptive User-Agent with a contact** is set from
  `defaults.user_agent_contact`. Identify yourself; do not impersonate a
  specific person or organization.
- **Conditional GET** (ETag / If-Modified-Since) is used for RSS so servers can
  answer 304 and skip resending unchanged content.
- **Public data only.** Kewpie does not log in, and does not bypass paywalls or
  authentication.

## Per-platform posture

- **RSS:** publisher-sanctioned. The safe default. Poll politely and use
  conditional GET.
- **Reddit:** prefer the no-auth `.rss` feed or the official API with your own
  OAuth credentials. Reddit requires a unique, descriptive User-Agent and
  throttles datacenter IPs. Kewpie does not ship `.json`/HTML scraping as a
  default. Arctic Shift (third-party archive) is opt-in for historical backfill.
- **X / Twitter:** only the official API is clearly within terms. The
  best-effort syndication path reads single public tweets you already have IDs
  for. Do not build anything load-bearing on it.
- **News APIs:** most free tiers forbid commercial use and/or require
  attribution. Check each provider's terms; the license is surfaced in config so
  downstream users can comply.

## Legal note

Accessing public data is generally permissible, but a site's terms of service
are a civil matter, and the law varies by jurisdiction. Honoring robots.txt and
rate limits materially reduces your exposure. You are responsible for how you
use this tool. When in doubt, ask the site operator, or use an official API.

## What this tool does not do

- No CAPTCHA solving.
- No defeat of behavioral or heavily obfuscated bot protection.
- No paywall or authentication bypass.
- No scraping of private or logged-in content.

These are deliberate boundaries. The escalation ladder reports when a target
needs capabilities beyond Kewpie so you can make an informed, lawful decision
about how to proceed.
