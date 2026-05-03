# Architecture

## Layering

The system is layered intentionally so that authentication, rate limiting,
and intrusion detection happen at the infrastructure layer, before any
application code runs.

```
┌──────────────────────────────────────────────────────────────────┐
│  L1 - TLS / Reverse proxy                                         │
│  Traefik v3 with Let's Encrypt, HSTS preload, modern cipher       │
│  suites only, CrowdSec bouncer at the edge                        │
├──────────────────────────────────────────────────────────────────┤
│  L2 - Authentication                                              │
│  Authelia forward-auth on admin subdomain (SSO + 2FA TOTP,        │
│  brute-force regulation, audit logs)                              │
├──────────────────────────────────────────────────────────────────┤
│  L3 - Rate limiting & ingress filtering                           │
│  nginx with per-IP rate limit zones on public endpoints,          │
│  explicit deny-by-default for non-public routes                   │
├──────────────────────────────────────────────────────────────────┤
│  L4 - Application                                                 │
│  FastAPI with no auth code, parameterized SQL throughout,         │
│  read-only filesystem, dropped capabilities                       │
├──────────────────────────────────────────────────────────────────┤
│  L5 - Storage                                                     │
│  SQLite (WAL) for relational data, NDJSON files for replay        │
│  events, periodic asyncio cleanup tasks                           │
└──────────────────────────────────────────────────────────────────┘
```

## Data model

```
┌─────────────┐   ┌──────────────┐   ┌──────────────┐
│  visitors   │   │  sessions    │   │  pageviews   │
│             │◄──┤              │◄──┤              │
│ + fp_hash   │   │ + ua + ip    │   │ + duration   │
└──────┬──────┘   └──────────────┘   │ + scroll     │
       │                              │ + clicks     │
       │ alias                        │ + fingerprint│
       ▼                              └──────┬───────┘
┌──────────────┐                             │
│   persons    │   ┌──────────────┐          │
│              │   │   events     │◄─────────┘
│ + properties │   │              │
└──────────────┘   │ + click data │
                   │ + web_vital  │
                   │ + js_error   │
                   └──────────────┘
                          │
                          ▼ ──── replay events ──── NDJSON files
                                                    (one per session)
```

## Tracker JS lifecycle

```
Page load
    │
    ▼
Generate / restore IDs (visitor, session, page)
    │
    ▼
Init Web Vitals observers (LCP, CLS, INP, FCP, TTFB)
    │
    ▼
Compute browser fingerprint (canvas, WebGL, audio, fonts, hardware)
    │
    ▼
POST pageview event with fingerprint
    │
    ▼
Set up listeners: scroll, click, mousemove (sampled),
                  copy, visibilitychange, error, unhandledrejection
    │
    ▼
Lazy-load rrweb recorder, start session replay
    │
    ▼
[user navigates...]
    │
    ▼
On pagehide / visibilitychange:
    flush replay buffer
    POST leave event with aggregated metrics + Web Vitals
```

## Privacy & data retention

- Tracker respects a server-injected skip flag for synthetic visits
  (the screenshot generator)
- Session replay masks form inputs by default (rrweb maskAllInputs)
- Elements with a designated CSS class are blocked from recording
- Events and pageviews retained 90 days, replays 30 days, screenshots 24h
- IP geolocation queried once per IP, results cached permanently

## Capabilities matrix

| Capability                  | PostHog default | This system |
|-----------------------------|:---------------:|:-----------:|
| Pageview tracking           | ✅              | ✅          |
| Autocapture (clicks/scroll) | ✅              | ✅          |
| Funnels                     | ✅              | ✅          |
| Retention cohorts           | ✅              | ✅          |
| Path analysis               | ✅              | ✅ (Sankey) |
| Session replay              | ✅              | ✅ (rrweb)  |
| Heatmaps                    | ⚠️ (toolbar)    | ✅ (full-page) |
| Persistent cohorts          | ✅              | ✅          |
| Web Vitals                  | ⚠️ (paid plan)  | ✅          |
| Canvas fingerprint          | ❌              | ✅          |
| WebGL/GPU fingerprint       | ❌              | ✅          |
| Audio fingerprint           | ❌              | ✅          |
| Bot/human heuristic score   | ❌              | ✅          |
| Related-visitor detection   | ❌              | ✅          |
| Feature flags / A-B         | ✅              | ❌ (skipped)|
| Surveys (NPS / multi-choice)| ✅              | ❌ (skipped)|
