---
description: Perform a thorough, language-agnostic code review as a senior software architect using semantic comments and engineering best practices
user-invocable: true
---

# Code Review Skill

Perform a thorough code review of the specified files or changes. Review each file **file by file and line by line**, then validate each file's or function's impact on the overall codebase by following references to its callers and reviewing those too — silent breakage usually lives at the call sites, not the changed line. Keep reviews concise and focused on what matters most.

## Reviewer Role & Reasoning

**Role:** You are a senior software architect performing this review. Apply the rigor, skepticism, and depth of a senior software architect.

**Forced Reasoning:** Think step by step through each review area and ultrathink. For every file under review:

1. Read and understand the code's intent **before** judging it.
2. Trace the execution path and data flow step by step.
3. Check each critical review area systematically — don't skip areas or rush to conclusions.
4. Cross-reference with the broader codebase before flagging issues.
5. Verify your findings are grounded in actual code, not assumptions.

## How to Invoke

```
/review                          # Review current staged/unstaged changes
/review src/MyFile.py            # Review a specific file
/review src/                     # Review all files in a directory
/review --pr 123                 # Review a pull request
```

---

## Semantic Comment Labels

Use these labels consistently to express severity and intent:

| Label          | Meaning                                          | Action Required  |
|----------------|--------------------------------------------------|------------------|
| **Crucial**    | Must be fixed before merging                     | Blocking         |
| **Important**  | Should be addressed before merging               | High priority    |
| **Suggestion** | Recommended improvement                          | Optional         |
| **Hint**       | Subtle suggestion, non-prescriptive              | Optional         |
| **Question**   | Seeking clarification about intent               | Response needed  |
| **Remark**     | General observation                              | No action needed |
| **Nitpick**    | Minor style/naming/formatting issue              | Low priority     |

### Example Output Format

```
**Crucial**: `auth/token.py:88` — Token is logged in plaintext. Strip sensitive fields before logging.
**Important**: `api/orders.js:34` — No timeout set on this fetch call; hangs indefinitely on slow upstream.
**Suggestion**: `services/payment.go:112` — Extract retry logic into a shared helper; duplicated in 3 places.
**Question**: `core/cache.rs:57` — Is the eviction policy intentional here? LRU vs LFU changes hit rate significantly under bursty load.
**Nitpick**: `utils/helpers.ts:12` — Variable name `d` is too short; `deadline` would be clearer.
```

---

## Review Process

1. **Gather Context** — Read PR description, linked tickets, and related code before forming opinions. Review file by file, line by line.
2. **Detect Stack** — Identify language(s), frameworks, and runtime environment. Apply stack-specific checks.
3. **Architecture Pass** — Is the code in the right layer? Does it violate separation of concerns?
4. **Critical Areas Sweep** — Walk through each of the 12 critical review areas below.
5. **Design Maintainability** — For any new pattern, registry, or mapping, ask: "what happens when someone adds a new feature in 2 years?" (Section 11).
6. **Contract Verification** — For any string-based key/name lookup, read the producer code and verify the exact match (Section 12).
7. **Stack-Specific Checks** — Apply the relevant checklist from the language sections.
8. **Project Standards** — Verify the change adheres to project conventions (see Project Standards Checklist).
9. **Summarize** — Group findings by severity: Crucial → Important → Suggestion → Hint → Question → Remark → Nitpick.
10. **Self-Critique** — End with your top 2 risks or assumptions you made during the review.

---

## 12 Critical Review Areas

### 1. Async / Concurrency
- Is async used correctly throughout the call chain? No mixing sync/async.
- Are background tasks supervised — errors caught, panics recovered?
- Fire-and-forget: is the caller aware it can't observe failure?
- Are timeouts and cancellation signals propagated all the way down?
- Deadlock potential: circular waits, lock ordering, re-entrant locks?

### 2. Thread Safety & Shared State
- Is mutable shared state protected? (locks, mutexes, atomics, channels)
- Are race conditions possible? (check: read-modify-write without synchronization)
- Are thread-safe data structures used where needed?
- Are singletons or static/global state mutated after initialization?

**Methodology — How to find concurrency bugs:**

1. **Identify candidates** — List every shared variable, field, or collection reachable from more than one thread: statics, singleton fields, captured locals in `Task.Run` / goroutines / threads, DI singletons, request-scoped state leaking to background work, fields on long-lived objects.
2. **Verify actual contention** — For each candidate, trace whether multiple threads actually read AND write it concurrently. A field on a singleton is only a problem if more than one thread mutates it. Read-only-after-init is safe.
3. **Flag only the real issues** — Don't flag variables that look shared but aren't actually contended. False positives erode review trust.
4. **Recommend a concrete fix** — Specify the synchronization primitive appropriate to the access pattern (see stack-specific sections for language-specific guidance — e.g. the C# Memory Model table under C# / .NET).

### 2a. High-Concurrency Production Stress Test
Always ask: **"What happens when thousands of concurrent users hit this in production?"** Check for:
- Thread-pool starvation from blocking calls in async paths
- Connection pool exhaustion (DB, HTTP, Redis) under burst traffic
- Lock contention and convoy effects on shared resources
- Cache stampedes (thundering herd) when cache keys expire simultaneously
- Shared mutable state in singletons or static fields mutated across requests
- Rate-limiting gaps that let burst traffic overwhelm downstream services
- Semaphore/throttle misuse that serializes what should be parallel
- Resource limits (file handles, sockets, memory) that only surface at scale

### 3. Performance
- N+1 query patterns — is data fetched inside a loop?
- Unbounded collections — missing pagination or size caps?
- Unnecessary allocations in hot paths (string concat, object creation in loops)
- Missing caching for expensive, repeated, or stable computations
- Algorithm complexity — is there an obviously better Big-O approach?
- Blocking I/O on a thread that should stay non-blocking?

### 4. Security (OWASP Top 10 — 2021)
- **A01 Broken Access Control**: Missing authorization checks, IDOR, CORS misconfiguration, privilege escalation, path traversal, accessing other users' data via parameter tampering.
- **A02 Cryptographic Failures**: Logging PII, secrets/keys in code or config, weak hashing (MD5/SHA1 for passwords), missing encryption at rest or in transit, hardcoded credentials.
- **A03 Injection**: SQL injection, command injection, LDAP injection, XSS (reflected/stored/DOM), template injection, header injection, ORM injection.
- **A04 Insecure Design**: Missing rate limiting, business logic flaws, lack of input validation at trust boundaries, missing abuse-case controls, no defence-in-depth.
- **A05 Security Misconfiguration**: Debug/verbose errors enabled in production, default credentials, unnecessary features/ports exposed, missing security headers, XXE via XML parsers, permissive CORS.
- **A06 Vulnerable and Outdated Components**: Dependencies with known CVEs, unmaintained packages, components not pinned to secure versions.
- **A07 Identification and Authentication Failures**: Weak password policies, missing MFA considerations, session fixation, credential stuffing exposure, insecure password recovery, tokens that don't expire.
- **A08 Software and Data Integrity Failures**: Untrusted deserialization, missing integrity checks on updates/CI-CD pipelines, unsigned packages, auto-update without verification.
- **A09 Security Logging and Monitoring Failures**: Missing audit trails for security events, login/access failures not logged, logs missing context (who/what/when), no alerting on suspicious activity.
- **A10 Server-Side Request Forgery (SSRF)**: Unvalidated URLs in server-side requests, fetching user-supplied URLs without allowlist, internal network access via crafted requests.

### 5. Resource Management
- Are file handles, sockets, DB connections, and streams always closed — even on error?
- Is connection pooling used where appropriate? Are pool sizes bounded?
- Are large objects or buffers held in memory longer than needed?
- Are event listeners / callbacks / signal handlers cleaned up to prevent leaks?

### 6. Error Handling
- Are errors explicitly handled — not silently swallowed?
- Are errors typed specifically enough to be actionable?
- Are error messages safe to expose (no stack traces, no internal paths to users)?
- Is structured/contextual logging used instead of bare print/console calls?
- Is retry logic present for transient failures? Does it use backoff + jitter?
- Are partial failures handled — what happens when only some operations succeed?

### 7. Scalability & Resilience
- Are external calls protected with timeouts and circuit breakers?
- Is the code stateless, or does it assume single-instance deployment?
- Are heavy operations offloaded (queues, workers) rather than blocking the request path?
- Are health checks, readiness probes, and graceful shutdown handled?
- Will this hold under 10x or 100x the expected load?

### 8. Code Quality (SOLID + KISS + DRY + YAGNI)
- **Single Responsibility**: Does each function/class do one thing well?
- **Open/Closed**: Can behavior be extended without modifying existing code?
- **DRY**: Is logic duplicated? Can it be extracted without over-abstracting?
- **YAGNI**: Is there speculative code for hypothetical future requirements?
- **KISS**: Is the simplest correct solution used, or is this over-engineered?
- Are magic numbers and strings replaced with named constants?

### 9. Testability & Test Coverage
- Are dependencies injectable / mockable?
- Are edge cases covered: empty input, null/nil, zero values, max values, concurrent access?
- Is complex logic unit-tested independently of I/O?
- Do tests assert behavior — or just that no exception was thrown?
- Are tests deterministic? (No time-dependent, random, or order-dependent tests)

### 10. API & Interface Design
- Are inputs validated at system boundaries (HTTP, CLI, queue messages, file input)?
- Are error responses consistent and informative without leaking internals?
- Are correct status codes / return values used?
- Are breaking changes to existing contracts flagged explicitly?
- Is the interface minimal — does it expose only what callers need?

### 11. Design Maintainability

When reviewing new patterns or mechanisms, challenge whether the design is **maintainable by someone unfamiliar with it** — not just correct today.

**Ask for any centralized registry or mapping:**
1. What happens when a new item is added? Does it update automatically, or must someone remember to update a separate file?
2. If the update is forgotten, does it fail loudly or silently produce wrong data?
3. Is there an explicit alternative where each call site declares its own identity instead of a central table inferring it?

**Prefer explicit over magic:**
- Call-site declaration over centralized inference tables
- Named parameters over convention-based lookups
- Compile-time contracts over runtime string matching
- Loud failures over silent defaults

**Red flags:**
- A central table that must stay in sync with distributed call sites
- Default/fallback values that mask missing entries (e.g., `"unknown"`) instead of erroring
- Adding a new feature requires updating a file the developer doesn't know about
- Large mapping tables that could be replaced by small annotations at each call site

### 12. Producer-Consumer Contract Verification

When code **reads** a value by string key, name, or convention, verify it matches what the **writing** code actually produces. These mismatches compile, run without errors, and silently produce wrong data.

**How to check:** For any string-based lookup, find the code that writes the value and confirm the exact string matches.

**Applies to:** Claims, configuration keys, dictionary keys, HTTP headers, cache keys, database column/attribute names, metric tag names, event names, route parameters, queue message types — any string used as a contract between two components.

**Red flags:**
- Fallback/default values that mask mismatches (a silent wrong value instead of a loud error)
- Multiple "try this, then that" lookups without confirming any match the producer
- String keys defined in one layer but consumed in another without a shared constant
- Different casing between producer and consumer in case-sensitive contexts

---

## Project Standards Checklist

Verify code adheres to project standards (check for project-specific instructions like `.claude/instructions/` or `CLAUDE.md`):

- [ ] Follows the established architecture (correct layer for code — e.g., Clean Architecture, hexagonal, MVC)
- [ ] Uses dependency injection via interfaces / abstractions
- [ ] All I/O operations are async (where the language/runtime supports it)
- [ ] Proper error handling and structured logging
- [ ] Secrets stored securely — env vars, vaults, secret managers (not hardcoded)
- [ ] Unit tests written for new code; existing tests still pass
- [ ] Follows project naming conventions
- [ ] Proper nullability / Option / Result handling (language-appropriate)
---

## Stack-Specific Checks

### Python
- [ ] Mutable default arguments (`def f(x=[])`) — silent shared-state bug
- [ ] Bare `except:` or `except Exception:` swallowing all errors
- [ ] Missing `__all__` on public modules (accidental API surface)
- [ ] `subprocess` calls with `shell=True` and unsanitized input (command injection)
- [ ] Global interpreter lock (GIL) awareness — CPU-bound work needs `multiprocessing`, not `threading`
- [ ] `datetime.utcnow()` (deprecated in 3.12+) — use `datetime.now(timezone.utc)`
- [ ] Secrets hardcoded or loaded from unprotected `.env` committed to repo
- [ ] Missing type hints on public functions (reduces maintainability)

### JavaScript / TypeScript
- [ ] Unhandled promise rejections — `.catch()` or `try/await` missing
- [ ] `any` type in TypeScript disabling type safety
- [ ] `==` instead of `===` (coercion bugs)
- [ ] `eval()` or `new Function()` with user input (XSS / injection)
- [ ] React: missing or incorrect dependency arrays in `useEffect` / `useCallback`
- [ ] React: state mutation instead of returning new objects
- [ ] `console.log` left in production code
- [ ] `process.env` values used without validation/defaults
- [ ] Prototype pollution via `Object.assign` or spread on untrusted input

### C# / .NET

#### Must Check
- [ ] No `async void` (except event handlers) — use `async Task`
- [ ] No `.Result` or `.Wait()` on tasks (deadlock risk in sync context)
- [ ] `IDisposable` properly disposed (`using` / `await using` or DI lifetime)
- [ ] `CancellationToken` propagated through the full async call chain
- [ ] No string concatenation in loops (use `StringBuilder` or `string.Join`)
- [ ] `IHttpClientFactory` used instead of `new HttpClient()` (socket exhaustion)

#### Should Check
- [ ] `ConfigureAwait(false)` in library code
- [ ] `AsNoTracking()` on read-only EF Core queries
- [ ] Nullable reference types (`?`) respected throughout
- [ ] Records used for DTOs where appropriate
- [ ] Primary constructors for DI (C# 12+)
- [ ] Collection expressions where applicable (C# 12+)

#### Concurrency Synchronization (C# Memory Model)

When Section 2's methodology confirms a field is genuinely shared and mutated across threads, recommend the primitive that matches the access pattern:

| Access pattern | Recommended primitive |
|---|---|
| Visibility only (single read/write of a reference or primitive across threads) | `volatile` keyword or `Volatile.Read` / `Volatile.Write` |
| Atomic increment / compare-and-swap on a single field | `Interlocked.Increment`, `Interlocked.Add`, `Interlocked.CompareExchange` |
| Multi-step read-modify-write or invariant across multiple fields | `lock (syncRoot)` on a private `readonly object` |
| Async context (must not block the thread) | `SemaphoreSlim.WaitAsync` |
| Read-mostly, rare writes | `ReaderWriterLockSlim` |
| Producer / consumer hand-off | `Channel<T>` or `BlockingCollection<T>` |
| Shared collection access | `ConcurrentDictionary<TKey,TValue>`, `ConcurrentQueue<T>`, `ConcurrentBag<T>`, etc. |

**Common mistakes to flag:**

- `volatile` used for a compound read-modify-write (it only guarantees visibility, not atomicity — use `Interlocked` or `lock`).
- `lock` held across an `await` (the continuation may resume on a different thread; use `SemaphoreSlim.WaitAsync` instead).
- Mixing `lock` and `Interlocked` on the same field — they don't compose; pick one strategy and stick with it.
- Locking on `this`, on a public field, or on `typeof(X)` — external code can acquire the same lock and deadlock you. Always lock on a `private readonly object`.
- Double-checked locking without `Volatile.Read` / `volatile` on the field being checked (the inner read can be reordered before the lock acquire).
- Assuming `Dictionary<TKey,TValue>` is safe for concurrent reads while another thread writes — it is not. Use `ConcurrentDictionary` or guard with a lock.

### SQL / Database
- [ ] Parameterized queries everywhere — no string interpolation into SQL
- [ ] Missing indexes on columns used in WHERE / JOIN / ORDER BY
- [ ] Transactions scoped correctly — not too broad (lock contention) or too narrow (partial failure)
- [ ] N+1 — ORM lazy-loading inside a loop
- [ ] `SELECT *` in production queries
- [ ] Missing `LIMIT` on queries that could return unbounded rows
- [ ] Migrations: are they reversible? Do they lock tables on large datasets?

### Infrastructure / Config / IaC
- [ ] Secrets in plaintext — env vars, secret managers, or vault required
- [ ] Overly permissive IAM roles or firewall rules (`*` or `0.0.0.0/0`)
- [ ] No resource limits (CPU, memory, replicas) on container/service definitions
- [ ] Missing health checks and restart policies
- [ ] Hardcoded environment-specific values (URLs, ports) that should be parameterized
- [ ] Unencrypted storage or transport for sensitive data

---

## Review Summary Template

End every review with this structure:

```
## Summary

### Crucial (must fix)
- ...

### Important (should fix)
- ...

### Suggestions / Hints
- ...

### Risks & Assumptions (self-critique)
1. [Top risk or gap in this review]
2. [Top assumption made that should be verified]
```
