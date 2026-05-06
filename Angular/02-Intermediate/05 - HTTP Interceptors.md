---
tags: [angular, intermediate, http]
aliases: [HttpInterceptor, Auth Interceptor, Functional Interceptor]
level: Intermediate
---

# HTTP Interceptors

> **One-liner**: An interceptor is a function that sits in the HTTP pipeline — every request flows through it on the way out, every response on the way back — used for auth headers, logging, retry, and error normalization.

---

## Quick Reference

| API | Purpose |
|-----|---------|
| `HttpInterceptorFn` | Functional interceptor type — `(req, next) => Observable<HttpEvent>` |
| `withInterceptors([fn1, fn2])` | Register in `provideHttpClient` |
| `req.clone({ setHeaders, setParams })` | Build a modified request (immutable) |
| `next(req)` | Continue down the chain |
| `inject()` | Inject services inside the function (functional context) |

---

## Core Concept

Interceptors are **middleware for `HttpClient`**. They run for every request — they can read or rewrite the request, kick off the call, and read or rewrite the response.

The chain is **ordered**: interceptors run in the order you register them on the way out (request) and in **reverse** order on the way in (response). So an "outermost" interceptor sees the request first and the response last — useful for logging.

Modern Angular uses **functional interceptors** — plain functions with the type `HttpInterceptorFn`. The old class-based `HttpInterceptor` interface still works but is being phased out (and doesn't work as cleanly with standalone bootstrap).

Common interceptor jobs:

- **Auth**: attach a bearer token from a session service.
- **Error normalization**: map `HttpErrorResponse` to your domain error type.
- **Retry**: re-issue idempotent failures with exponential backoff.
- **Logging / metrics**: timing, structured logs.
- **Caching**: short-circuit GET requests with a memory cache.

---

## Diagram

```mermaid
flowchart LR
    C[Component / Service] --> H[HttpClient]
    H --> A[Auth interceptor]
    A --> L[Logging interceptor]
    L --> R[Retry interceptor]
    R --> Be[Backend]
    Be --> R
    R --> L
    L --> A
    A --> H
    H --> C
```

---

## Syntax & API

### Functional auth interceptor

```ts
// core/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  if (!token) return next(req);

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });
  return next(authReq);
};
```

### Register interceptors

```ts
// main.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/auth.interceptor';
import { errorInterceptor } from './core/error.interceptor';
import { loggingInterceptor } from './core/logging.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([loggingInterceptor, authInterceptor, errorInterceptor]),
    ),
  ],
});
```

### Error normalization

```ts
// core/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export interface AppError {
  kind: 'network' | 'auth' | 'validation' | 'server';
  status: number;
  message: string;
}

export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((e: HttpErrorResponse) =>
      throwError(() => normalize(e)),
    ),
  );

function normalize(e: HttpErrorResponse): AppError {
  if (e.status === 0)        return { kind: 'network',    status: 0,   message: 'Network unreachable' };
  if (e.status === 401)      return { kind: 'auth',       status: 401, message: 'Not authenticated' };
  if (e.status === 422)      return { kind: 'validation', status: 422, message: e.error?.message ?? 'Invalid' };
  return                          { kind: 'server',     status: e.status, message: e.statusText };
}
```

### Retry with backoff

```ts
// core/retry.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.method !== 'GET') return next(req); // only retry idempotent
  return next(req).pipe(
    retry({
      count: 3,
      delay: (err, retryCount) => timer(2 ** retryCount * 200),
    }),
  );
};
```

### Logging with timing

```ts
// core/logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { tap, finalize } from 'rxjs';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const start = performance.now();
  return next(req).pipe(
    finalize(() => {
      const ms = (performance.now() - start).toFixed(0);
      console.debug(`[http] ${req.method} ${req.url} — ${ms}ms`);
    }),
  );
};
```

---

## Common Patterns

```ts
// Pattern: bypass interceptors for a single request via a context token
import { HttpContextToken, HttpContext } from '@angular/common/http';

export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);

// In the interceptor:
if (req.context.get(SKIP_AUTH)) return next(req);

// In the call site:
this.http.get('/public', { context: new HttpContext().set(SKIP_AUTH, true) });
```

```ts
// Pattern: caching idempotent GETs in memory
const cache = new Map<string, HttpResponse<unknown>>();

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.method !== 'GET') return next(req);
  const hit = cache.get(req.urlWithParams);
  if (hit) return of(hit.clone());
  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) cache.set(req.urlWithParams, event);
    }),
  );
};
```

---

## Gotchas & Tips

- **Requests are immutable.** `req.url = ...` does nothing; you must `req.clone({...})`.
- **Order matters.** Auth before error-normalization, error-normalization before retry (so retry doesn't kick on auth errors).
- **Don't forget `inject()`** is available inside functional interceptors — that's how you read services. Don't try to `import` a service instance.
- **Context tokens beat URL pattern matching.** "Skip auth for `/public/*`" is brittle; a `SKIP_AUTH` context token is explicit at the call site.
- **Functional interceptors only** if you bootstrap with `provideHttpClient(withInterceptors(...))`. Class-based interceptors registered with `HTTP_INTERCEPTORS` still work but require `withInterceptorsFromDi()`.
- **Watch infinite loops** when an interceptor calls another HTTP request (e.g. token refresh). Always exclude that request from the same interceptor.

---

## See Also

- [[04 - HttpClient]]
- [[12 - Dependency Injection Deep Dive]]
- [[18 - Monitoring and Errors]]
