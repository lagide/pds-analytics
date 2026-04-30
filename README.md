# PDS Analytics

> **Self-hosted visitor forensics — analytics platform comparable to PostHog
> on the fundamentals, with forensic specifics PostHog doesn't ship by default.**

This repository documents the architecture and capabilities of an internal
analytics platform I built to instrument my portfolio. The actual implementation
remains private.

[![Status](https://img.shields.io/badge/status-LIVE-00ff41)]()
[![License](https://img.shields.io/badge/license-Proprietary-lightgrey)]()
[![Security](https://img.shields.io/badge/security_audit-9.7%2F10-00ff41)]()

---

## Why build this?

PostHog is the gold standard of self-hostable product analytics, but it's
designed for SaaS-scale: 4 vCPU + 16 GB RAM minimum, recommended only below
a certain monthly event volume. For a portfolio site doing a few thousand
visits per month, that's 50× more infrastructure than the use case demands.

I wanted the same primitives — events, persons, funnels, retention, replay
— in a footprint compatible with a single VPS, plus a few **forensic
capabilities** that PostHog doesn't expose: deterministic browser fingerprinting,
bot/human heuristic scoring, and related-visitor detection across sessions.

## Capabilities (8 sprints delivered)

1. **Persons + identify()** — anonymous → known, cross-session merge
2. **Web Vitals + JS errors** — auto-captured via `PerformanceObserver` + `window.error`
3. **Visual funnel builder** — multi-step conversion with drop-off cascade
4. **Retention cohort table** — Mixpanel-style heatmap, daily/weekly modes
5. **Session replay (rrweb)** — DOM mutation journal, replayable like a film
6. **Live admin toolbar** — heatmap overlay on the live site, in real time
7. **Sankey paths diagram** — D3 visualization of navigation flows
8. **Persistent cohorts** — visual rule engine on aggregated visitor properties

Plus:
- Canvas / WebGL / audio fingerprinting
- Bot/human score with weighted heuristics
- Full hardware profile (cores, RAM, screen DPR, battery, connection)
- Full-page heatmap with screenshot background
- Cross-viewport filtering (desktop / tablet / mobile)

## Stack

```
Backend       : FastAPI (Python 3.12) · SQLite (aiosqlite, WAL)
Frontend      : Vanilla JS · Chart.js · D3-Sankey · rrweb-player
Sidecar       : Playwright Chromium (full-page screenshots)
Replay        : rrweb (DOM mutation journal)
Reverse-proxy : nginx · Traefik v3 + Let's Encrypt
Auth          : Authelia (SSO + 2FA TOTP, forward-auth)
Security      : CrowdSec firewall bouncer · per-IP rate limiting
Container     : Docker Compose · read-only filesystem · cap_drop ALL
```

## Architecture (high-level)

```
                          Internet
                              │
                       TLS-terminating reverse proxy
                       (HSTS · IDS · rate limiting)
                              │
                ┌─────────────┴──────────────┐
                ▼                            ▼
          Public site                 Admin subdomain
                │                            │
       Public tracker endpoints      SSO + 2FA gate
       (per-IP rate limited)                 │
                │                            ▼
                └────────────┬───────  Analytics backend
                             ▼            (read-only container)
                       Analytics core
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
        Embedded SQL DB           Screenshotter sidecar
        (event store)             (Playwright Chromium)
```

The auth layer is **fully external** — the application has zero auth code.
This was a deliberate refactor mid-project: a single source of truth for
authentication, audit-friendly logs, and clean separation of concerns.

## Pivots

### Application secret → SSO-only auth
First version used a shared application secret in URL parameters. Multiple
problems: secret in access logs, browser history, Referer headers. Refactored
to SSO + 2FA forward-auth — backend has zero auth code, single source of truth.

### Heatmap viewport-only → full-page with screenshot background
Initial heatmap rendered viewport-relative coordinates on a fixed canvas —
clicks at the bottom of long pages collapsed onto the top. Switched to
full-page screenshots and document-relative positioning with the page height
captured at click time.

## Security review

A complete security review at the end of the build identified one HIGH severity
finding (a SSRF in the screenshot endpoint due to insufficient input
validation) and a few low-severity issues. All fixed:

- **HIGH** — SSRF in screenshot generation: input validation + explicit
  allow-list
- **LOW** — JS-readable admin cookie: moved to HttpOnly + server-side
  presence marker
- **LOW** — Stack traces leaking internal paths: regex anonymization
- **LOW** — Database file permissions: tightened

Final security audit rating: **9.7 / 10**.

## Lessons learned

### Delegate auth to infrastructure, not the application
Putting a shared secret in the application is convenient but operationally
leaky. Moving it to a forward-auth layer (Authelia in this case) gave 2FA,
brute-force regulation, centralized sessions, and audit logs essentially
for free.

### Defense-in-depth is not paranoia, it works in practice
During endpoint validation testing I got auto-banned by my own IDS for an
automated-probing pattern. Confirmation that the system actually does what
it advertises. Lesson: integration tests should respect the rate limits or
run from inside the container network.

### Right-size the toolchain to the use case
PostHog is excellent for product teams shipping at SaaS scale. For a single
portfolio site, SQLite + FastAPI handles the same primitives at a fraction
of the resource cost. Choose the tool that fits the size of the problem.

## License

Proprietary. Architecture documentation public for portfolio purposes;
implementation details remain private.

---

## Related

This repository hosts only **architecture documentation**. The full
implementation (backend, infrastructure configs, frontend tracker, dashboard)
remains in a private repository — what's published here is the design rationale,
the lessons learned, and a high-level architecture overview suitable for a
portfolio showcase.

---

*Architecture documentation only. Source code and deployment specifics remain private.*
