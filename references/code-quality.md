# Code Quality & Security Review Lens

Applies to application, library, and tooling source in any language. The organising question: **what will hurt this team in six months, and what will hurt a user today?**

## Contents

1. Structure and design
2. Error handling
3. Configuration
4. Logging and observability
5. Security
6. Data and persistence
7. Concurrency
8. Performance
9. Dependencies
10. Maintainability signals

---

## 1. Structure and design

- **God objects and god functions.** A 600-line function or a class with twenty responsibilities cannot be tested in isolation or changed safely. Cite the specific file and line span rather than making a general complaint about size.
- **Business logic in the wrong layer** — rules embedded in HTTP handlers, UI components, or database triggers where they can't be reused or unit tested.
- **Circular dependencies** between modules, which make ordering fragile and prevent extraction.
- **Missing seams.** Code that instantiates its own dependencies internally (`self.client = HttpClient(PROD_URL)`) cannot be tested without network access. Constructor injection is usually a small change with large payoff.
- **Inconsistent patterns.** Three different approaches to the same problem in one codebase — each may be fine, but the inconsistency is a real cost to every future reader.
- **Premature abstraction** — interfaces with one implementation, config for things that never vary, plugin systems with no plugins. Just as costly as too little abstraction.

## 2. Error handling

- **Swallowed exceptions.** `except Exception: pass`, empty catch blocks, `.catch(() => {})`. Failures disappear and debugging becomes archaeology. Grep for these directly; they're usually easy wins.
- **Catch-all where specific is needed.** Catching broad exception types hides programming errors alongside expected failures.
- **Errors logged and then re-thrown**, producing the same failure three times in the logs at different layers.
- **No handling of external failure at all** — network calls, file reads, and third-party APIs treated as if they always succeed.
- **No timeout on outbound calls.** A request with no timeout can hang indefinitely and exhaust a connection pool. This is a common Critical-severity, Small-effort finding.
- **No retry with backoff** on transient failures, or retries with no backoff, which turns a blip into a self-inflicted denial of service.
- **Error messages that reach users containing stack traces, SQL, or internal hostnames.**

## 3. Configuration

- **Hardcoded environment values** — URLs, hostnames, ports, feature toggles baked into source.
- **No validation at startup.** Missing config discovered at 3am when the code path finally runs, rather than at boot. Fail fast and loudly on missing required config.
- **Inconsistent config sources** — some values from env, some from a file, some from constants, with unclear precedence.
- **No documented example** (`.env.example` or equivalent), making onboarding a guessing game.
- **Secrets in config files** committed to the repo. Critical.

## 4. Logging and observability

- **No logging on critical paths**, or logging only on success.
- **Sensitive data in logs** — passwords, tokens, full payment details, personal data. A compliance issue as well as a security one.
- **`print` / `console.log` used as logging**, with no levels, structure, or destination control.
- **No correlation or request id**, making it impossible to trace one request across services.
- **Unstructured messages** that can't be queried or alerted on.
- **Log levels misused** — everything at INFO, or errors logged at WARNING so alerting never fires.

## 5. Security

- **Injection.** String-concatenated SQL, shell commands built from user input, `eval` on external data. Parameterised queries and argument-array subprocess calls are the fixes.
- **Missing authorisation checks.** Authentication answers who; authorisation answers whether they may. Endpoints that verify a token but not ownership of the requested resource are a common and serious gap.
- **Secrets in source or history.**
- **Insecure defaults** — TLS verification disabled, permissive CORS (`*` with credentials), debug mode enabled in production config.
- **No input validation at trust boundaries** — anything crossing from user, network, or file into the system.
- **Outdated dependencies with known CVEs**, especially in anything parsing untrusted input.
- **Weak or homegrown cryptography**, MD5/SHA1 for passwords, hardcoded IVs, or custom auth token schemes.

## 6. Data and persistence

- **N+1 queries** — a loop issuing one query per item where a join or batch fetch would do. Common and often a large, cheap performance win.
- **Missing indexes** on columns used in frequent WHERE and JOIN clauses.
- **Unbounded result sets** — `SELECT *` with no limit on a table that grows.
- **No transaction boundaries** around multi-step writes that must succeed or fail together.
- **Migrations that lock large tables** during deploy.
- **No connection pooling**, or a pool sized without regard to concurrency limits.

## 7. Concurrency

- **Shared mutable state without synchronisation.**
- **Async functions never awaited**, silently dropping work and swallowing errors.
- **Blocking calls inside async code**, stalling the event loop.
- **Fire-and-forget background work** with no error handling or completion tracking.
- **Race conditions in check-then-act sequences** — verifying a record doesn't exist and then creating it, without a unique constraint as a backstop.

## 8. Performance

Only raise performance findings where there's a plausible real impact — speculative micro-optimisation is noise.

- Work repeated inside a loop that could be hoisted out.
- Whole files or datasets loaded into memory when streaming would do.
- Missing caching on expensive, frequently repeated, rarely changing computations.
- Synchronous I/O in a request path.
- Algorithmic complexity that will not survive data growth — nested loops over collections that scale with users or records.

## 9. Dependencies

- **No lockfile committed**, making builds non-reproducible.
- **Wildcard or floating version ranges** on critical dependencies.
- **Unmaintained packages** — last release years ago, archived upstream.
- **Heavy dependency for trivial use** — a large library imported for one small function.
- **Duplicate libraries** doing the same job (two HTTP clients, two date libraries), which is both bloat and a source of inconsistent behaviour.

## 10. Maintainability signals

- **Dead code** — unreachable branches, commented-out blocks, unused exports. Version control already remembers; delete it.
- **TODO and FIXME comments** with no ticket or owner. Count them and check the oldest date; a large aging pile signals a backlog nobody is tracking.
- **Comments explaining what rather than why.** The valuable comment records the constraint or decision that isn't visible in the code.
- **Missing or stale documentation** — a README whose setup instructions no longer work is worse than none, since it costs every newcomer an hour before they distrust it.
- **No type hints or annotations** in a codebase large enough to benefit, or types present but effectively disabled by pervasive `any` / `# type: ignore`.
- **Tests absent for the most complex, highest-churn modules** — worth cross-checking churn data from `git log` against test file locations.
