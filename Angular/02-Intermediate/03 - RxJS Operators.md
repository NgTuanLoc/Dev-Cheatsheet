---
tags: [angular, intermediate, rxjs]
aliases: [Operators, switchMap, mergeMap, debounceTime]
level: Intermediate
---

# RxJS Operators

> **One-liner**: Operators are pure functions that take an Observable and return a new Observable — chained with `pipe()` to filter, transform, combine, and time streams.

---

## Quick Reference

| Operator | Purpose | Mental model |
|----------|---------|--------------|
| `map(fn)` | Transform each value | `arr.map` |
| `filter(pred)` | Keep matching values | `arr.filter` |
| `tap(fn)` | Side effect, no change | "spy" |
| `take(n)` / `takeWhile(p)` | First N / while predicate | early stop |
| `skip(n)` / `skipWhile(p)` | Drop first N / while predicate | |
| `debounceTime(ms)` | Emit after silence | search-as-you-type |
| `throttleTime(ms)` | Emit at most every N ms | scroll/resize |
| `distinctUntilChanged()` | Skip consecutive duplicates | |
| `switchMap(fn)` | Cancel prev inner, switch to new | search, navigation |
| `mergeMap(fn)` | Run all inners in parallel | independent requests |
| `concatMap(fn)` | Queue inners sequentially | order matters |
| `exhaustMap(fn)` | Ignore new while inner runs | submit-button guard |
| `combineLatest` / `withLatestFrom` | Combine streams | |
| `catchError(fn)` | Recover from error | |
| `retry(n)` / `retryWhen` | Retry on error | |
| `share()` / `shareReplay(n)` | Multicast cold to many subs | |

---

## Core Concept

Operators are how you **transform pipelines**. Each one is a function from `Observable<A>` to `Observable<B>`. You compose them inside `pipe()`:

```ts
src$.pipe(
  filter(x => x > 0),
  map(x => x * 2),
  debounceTime(200),
).subscribe(...);
```

The four **higher-order mapping operators** (`switchMap`, `mergeMap`, `concatMap`, `exhaustMap`) are the most important — and the most commonly confused. Each takes a function returning an inner Observable and decides what to do when a **new outer value arrives while an inner is still running**:

- **switchMap**: cancel the running inner, start the new one. Right for searches and "latest wins."
- **mergeMap**: run them all in parallel. Right when each is independent.
- **concatMap**: queue — wait for the running one to finish, then start. Right when order matters.
- **exhaustMap**: ignore the new one until the running one finishes. Right for submit buttons (prevent double-submits).

Pick the wrong one and you get race conditions, duplicate requests, or stuck UIs.

---

## Syntax & API

### `map`, `filter`, `tap`

```ts
import { map, filter, tap } from 'rxjs';

source$
  .pipe(
    filter(x => x.active),
    map(x => x.name),
    tap(name => console.log('saw', name)),
  )
  .subscribe(name => render(name));
```

### Time-based: `debounceTime`, `distinctUntilChanged`

```ts
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';
import { fromEvent } from 'rxjs';

const input = document.getElementById('q')!;
fromEvent(input, 'input')
  .pipe(
    map(e => (e.target as HTMLInputElement).value),
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(q => api.search(q)),
  )
  .subscribe(results => render(results));
```

### `switchMap` (search-as-you-type)

```ts
this.query$
  .pipe(
    debounceTime(250),
    switchMap(q => this.api.search(q)), // cancels prior search
  )
  .subscribe(results => this.results = results);
```

### `mergeMap` (parallel)

```ts
ids$
  .pipe(
    mergeMap(id => this.api.fetchDetail(id), 5), // up to 5 in flight
  )
  .subscribe(detail => store.add(detail));
```

### `concatMap` (sequential, ordered)

```ts
saves$
  .pipe(
    concatMap(item => this.api.save(item)), // one at a time, in order
  )
  .subscribe();
```

### `exhaustMap` (submit guard)

```ts
clicks$
  .pipe(
    exhaustMap(() => this.api.submit(this.form.value)), // ignore further clicks while in flight
  )
  .subscribe();
```

### Error handling: `catchError`, `retry`

```ts
import { catchError, retry, of } from 'rxjs';

api.getUsers()
  .pipe(
    retry({ count: 2, delay: 500 }),
    catchError(err => {
      console.error(err);
      return of([] as User[]);    // fall back to empty
    }),
  )
  .subscribe(users => this.users = users);
```

### Combining: `combineLatest`, `withLatestFrom`

```ts
combineLatest([user$, theme$]).subscribe(([u, t]) => render(u, t));

clicks$
  .pipe(withLatestFrom(currentUser$))
  .subscribe(([_click, user]) => doSomething(user));
```

### Multicasting: `shareReplay`

```ts
this.config$ = this.http.get<Config>('/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true }),
);
// Multiple subscribers share one HTTP call; latest value replayed to late subscribers.
```

---

## Common Patterns

```ts
// Pattern: typeahead with cancellation + minimum-length
this.suggestions$ = this.queryControl.valueChanges.pipe(
  filter((q): q is string => typeof q === 'string'),
  map(q => q.trim()),
  filter(q => q.length >= 2),
  debounceTime(250),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q).pipe(
    catchError(() => of([])),
  )),
);
```

```ts
// Pattern: poll while visible, stop when not
import { interval, switchMap, takeWhile } from 'rxjs';

interval(5000).pipe(
  switchMap(() => this.api.status()),
  takeWhile(s => s.kind !== 'done', true),
).subscribe(s => this.status = s);
```

---

## Gotchas & Tips

- **`switchMap` is the right default** for searches, navigations, and "latest wins" UI. Don't use `mergeMap` unless you genuinely need parallel inners.
- **`debounceTime` then `switchMap`**, not the other way around — debounce in the outer stream, then map.
- **`distinctUntilChanged()` for primitives works.** For objects, pass a comparator (`(a, b) => a.id === b.id`).
- **`shareReplay({ refCount: true })` is what you want for "share an HTTP request among subscribers."** Without `refCount`, the source never tears down.
- **`tap` is for side effects only.** Don't use it to mutate values — that's what `map` is for.
- **`subscribe()` always last.** Operators are inside `pipe()`. A common bug is `pipe(...subscribe(...))` — wrong shape.

---

## See Also

- [[02 - RxJS Fundamentals]]
- [[04 - HttpClient]]
- [[08 - RxJS Advanced]]
- [[01 - Signals]]
