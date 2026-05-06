---
tags: [angular, intermediate, rxjs]
aliases: [Observable, Subject, BehaviorSubject, RxJS Basics]
level: Intermediate
---

# RxJS Fundamentals

> **One-liner**: RxJS models async values over time as **Observables** — lazy producers that emit zero-or-more values until they complete or error, with operators that compose like LINQ.

---

## Quick Reference

| Concept | Symbol / API |
|---------|-------------|
| Stream | `Observable<T>` |
| Subscribe | `obs.subscribe({ next, error, complete })` |
| Cleanup | `subscription.unsubscribe()` |
| Multicast | `Subject<T>` (no initial), `BehaviorSubject<T>(init)`, `ReplaySubject<T>(n)` |
| Hot vs cold | Cold: each subscriber starts fresh. Hot: shared, in flight |
| Of values | `of(1, 2, 3)` |
| From array/promise | `from([1, 2, 3])`, `from(fetch(...))` |
| Interval | `interval(ms)`, `timer(delay, period?)` |
| Combine | `combineLatest`, `merge`, `zip`, `forkJoin` |

---

## Core Concept

An **Observable** is a producer of values. Until something subscribes, it does nothing — it's a *recipe*, not a running task. When you subscribe, the producer starts emitting `next` values; eventually it `complete`s or `error`s, and your subscription is over.

That's the key difference from a Promise: Promises produce **one** value, eagerly, and aren't cancelable. Observables produce **many** values (or zero), lazily, and **are** cancelable via `unsubscribe()`.

A **Subject** is an Observable you can also push values into. It's how you *make* a stream from imperative code (button clicks, websocket messages). `BehaviorSubject` adds an initial value and replays the latest to new subscribers — the canonical way to model "current state."

In Angular, the two main RxJS sources are **HttpClient** (every HTTP call returns a cold Observable) and the **Router** (events, params, query params are Observables). The `async` pipe and `toSignal()` consume them without manual subscriptions.

---

## Diagram

```mermaid
sequenceDiagram
    participant Sub as Subscriber
    participant Obs as Observable
    participant Prod as Producer
    Sub->>Obs: subscribe({next, error, complete})
    Obs->>Prod: start producing
    Prod-->>Sub: next(value)
    Prod-->>Sub: next(value)
    Prod-->>Sub: complete()
    Note right of Sub: subscription ends; resources released
```

---

## Syntax & API

### Subscribing

```ts
import { Observable } from 'rxjs';

const obs = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});

obs.subscribe({
  next: v => console.log(v),
  error: e => console.error(e),
  complete: () => console.log('done'),
});

// Shorthand
obs.subscribe(v => console.log(v));
```

### Creation operators

```ts
import { of, from, interval, timer, fromEvent, EMPTY } from 'rxjs';

of(1, 2, 3);                          // 1, 2, 3, complete
from([1, 2, 3]);                      // same
from(fetch('/api'));                  // wraps a promise
interval(1000);                       // 0, 1, 2, … every second
timer(2000, 500);                     // first after 2s, then every 500ms
fromEvent<MouseEvent>(window, 'click'); // hot stream of clicks
EMPTY;                                // completes immediately, no values
```

### Subjects

```ts
import { Subject, BehaviorSubject, ReplaySubject } from 'rxjs';

const clicks = new Subject<MouseEvent>();
clicks.subscribe(e => console.log('A:', e));
clicks.next(new MouseEvent('click'));
clicks.subscribe(e => console.log('B:', e)); // misses past values

const state = new BehaviorSubject<number>(0);
state.subscribe(v => console.log('state:', v)); // logs 0 immediately
state.next(1);                                  // logs 1
state.value;                                    // 1 (synchronous read)

const buffered = new ReplaySubject<string>(2);
buffered.next('a'); buffered.next('b'); buffered.next('c');
buffered.subscribe(v => console.log(v)); // logs 'b', 'c'
```

### Combining

```ts
import { combineLatest, forkJoin, merge, zip } from 'rxjs';

combineLatest([user$, settings$]).subscribe(([u, s]) => /* both latest */);
forkJoin({ user: api.user(), prefs: api.prefs() }).subscribe(result => /* both completed */);
merge(clicks$, key$).subscribe(/* either */);
zip(a$, b$).subscribe(/* paired by index */);
```

---

## Common Patterns

```ts
// Pattern: takeUntilDestroyed in a component
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class FeedComponent {
  private socket = inject(SocketService);
  messages: Message[] = [];

  constructor() {
    this.socket.messages$
      .pipe(takeUntilDestroyed())
      .subscribe(msg => this.messages.push(msg));
  }
}
```

```ts
// Pattern: BehaviorSubject as a service-state singleton
@Injectable({ providedIn: 'root' })
export class SessionService {
  private _user$ = new BehaviorSubject<User | null>(null);
  user$ = this._user$.asObservable();
  setUser(u: User | null) { this._user$.next(u); }
  get currentUser() { return this._user$.value; }
}
```

```ts
// Pattern: prefer the async pipe over manual subscribe
@Component({
  template: `
    @if (user$ | async; as u) { <p>Hi, {{ u.name }}</p> }
  `,
})
export class HeaderComponent {
  user$ = inject(SessionService).user$;
}
```

---

## Gotchas & Tips

- **Cold observables restart per subscriber.** Two `http.get(...)` subscriptions = two HTTP requests. Use `share()` / `shareReplay(1)` to multicast.
- **HTTP observables complete after one emission** — the `async` pipe handles this fine; `toSignal` needs `initialValue` because it emits `undefined` until the first value arrives.
- **Always pair `subscribe()` with cleanup.** Use `takeUntilDestroyed()` (Angular interop) or the `async` pipe rather than manual `ngOnDestroy` plumbing.
- **Don't call `next` on a Subject from another component directly.** Encapsulate it: expose `obs$ = subject.asObservable()` and a method like `setUser(u)`.
- **`BehaviorSubject.value` is a sync escape hatch** — handy in tests and rare imperative reads, but resist using it as a general state-read pattern.
- **Promises ≠ Observables.** Wrapping a promise with `from()` is one-shot; if you want to retry/cancel, use `defer(() => fetch(...))` and a fresh subscription.

---

## See Also

- [[03 - RxJS Operators]]
- [[01 - Signals]]
- [[04 - HttpClient]]
- [[08 - RxJS Advanced]]
