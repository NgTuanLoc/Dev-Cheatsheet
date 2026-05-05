---
tags: [react, advanced, devops, performance]
aliases: [Monitoring, Sentry, Web Vitals, RUM]
level: Advanced
---

# Monitoring and Errors

> **One-liner**: Production React monitoring has three pillars — **error tracking** (Sentry/Rollbar), **real user monitoring (RUM) for web vitals**, and **session replay**; pair with structured logs from server actions / route handlers for full coverage.

---

## Quick Reference

| Concern | Tool |
|---------|------|
| JS errors + stack traces | **Sentry**, Rollbar, Bugsnag, Datadog RUM |
| React error boundaries → tracker | `@sentry/react` ErrorBoundary integration |
| Web vitals (LCP/INP/CLS/TTFB) | `web-vitals` lib, Sentry Performance, Vercel Analytics |
| Session replay | Sentry Replay, LogRocket, FullStory |
| Custom events / funnels | PostHog, Amplitude, Mixpanel |
| Server logs | Pino, Winston, structured JSON to Datadog/Logtail |
| Source maps for stacks | Upload at deploy time |
| Privacy / GDPR | Mask inputs, scrub PII from event payloads |

---

## Core Concept

A "React app" in production is a moving target — different browsers, devices, networks, ad-blockers, third-party scripts. The only way to know what users actually experience is **measurement in the field**.

The basic stack:
1. **Error tracker** — catches uncaught exceptions, promise rejections, errors swallowed by React error boundaries. Symbolicates stacks via uploaded source maps.
2. **Web Vitals** — Core Web Vitals (LCP, INP, CLS) are Google's perceived-performance metrics. Capture them as RUM, not just lab.
3. **Session replay** — when an error happens, replay the user's last N seconds. Saves hours of "can you reproduce?" back-and-forth.
4. **Structured server logs** — JSON logs with request ID tie front-end errors to back-end stack traces.

A common mistake: instrument heavily, **forget to look at the data**. Set up alerting on error rate spikes and INP regressions; that's where the value compounds.

---

## Syntax & API

### Sentry — basic setup

```bash
npm install @sentry/react
```

```ts
// main.tsx (before createRoot)
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  release: import.meta.env.VITE_GIT_SHA,
  tracesSampleRate: 0.2,                 // 20% performance sampling
  replaysSessionSampleRate: 0.1,         // 10% of sessions
  replaysOnErrorSampleRate: 1.0,         // 100% on error
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({ maskAllText: false, blockAllMedia: false }),
  ],
});
```

```tsx
// Wrap with the Sentry ErrorBoundary
import { Sentry } from "@sentry/react";

createRoot(rootEl).render(
  <Sentry.ErrorBoundary fallback={<p>Something broke.</p>}>
    <App />
  </Sentry.ErrorBoundary>,
);
```

### Manual error capture

```ts
import * as Sentry from "@sentry/react";

try {
  await api.do();
} catch (err) {
  Sentry.captureException(err, {
    tags:    { feature: "checkout" },
    extra:   { userId, cartId },
    contexts: { request: { url } },
  });
  throw err;
}
```

### Web Vitals (vanilla)

```ts
import { onCLS, onINP, onLCP, onTTFB, onFCP } from "web-vitals";

function send(name: string, m: any) {
  navigator.sendBeacon("/rum", JSON.stringify({
    name,
    value: m.value,
    rating: m.rating,
    id: m.id,
    url: location.pathname,
  }));
}

onCLS(m => send("CLS", m));
onINP(m => send("INP", m));
onLCP(m => send("LCP", m));
onTTFB(m => send("TTFB", m));
onFCP(m => send("FCP", m));
```

### Source map upload (Vite + Sentry CLI)

```bash
# vite.config.ts: build.sourcemap = "hidden"
# package.json scripts:
{
  "build": "vite build",
  "release": "sentry-cli releases new $VITE_GIT_SHA && sentry-cli sourcemaps upload dist/ --release $VITE_GIT_SHA"
}
```

### Track route changes (React Router)

```tsx
import { useLocation } from "react-router-dom";

function RouteTracker() {
  const location = useLocation();
  useEffect(() => {
    posthog.capture("$pageview", { path: location.pathname });
  }, [location.pathname]);
  return null;
}
```

### Server-side structured logs (Node)

```ts
import pino from "pino";
const logger = pino({ level: process.env.LOG_LEVEL ?? "info" });

logger.info({ userId, requestId }, "user fetched");
logger.error({ err, requestId }, "checkout failed");
```

---

## Common Patterns

```ts
// Pattern: tag every error with a request/correlation ID for cross-stack debugging
Sentry.setTag("requestId", requestId);

// Pattern: scrub PII before send
Sentry.init({
  beforeSend(event) {
    if (event.request?.headers) delete event.request.headers["Authorization"];
    return event;
  },
});

// Pattern: alert on regressions, not absolutes
//   - "INP p75 increased > 20% week-over-week"
//   - "Error rate exceeded 0.5% over 5 minutes"
```

---

## Gotchas & Tips

- **Without source maps, stack traces are useless.** Upload on every deploy.
- **Don't sample errors at 20%.** Sample performance traces, capture all errors.
- **Console errors aren't enough** — many real-user errors never reach the dev console.
- **Privacy matters.** Mask inputs in session replay (`<input data-sentry-mask>`), scrub auth headers, redact emails per regulation.
- **Don't log inside hot paths.** Bursts of `Sentry.captureMessage` from a render loop will flood your quota.
- **Tag releases.** Otherwise old errors from prior deploys pollute new ones.
- **Distinguish front-end vs back-end errors.** Tag origin; correlate via request ID.
- **INP > FID since 2024**. Old monitoring dashboards may still report FID — switch.
- **Set budgets.** "Bundle ≤ 200 KB gzipped, INP p75 ≤ 200 ms, CLS ≤ 0.1." Fail the CI build if exceeded.
- **Investigate replays only when you have an error or complaint.** They're privacy-sensitive and time-consuming.

---

## See Also

- [[15 - Error Boundaries]]
- [[06 - Performance Optimization]]
- [[16 - Build and Bundling]]
