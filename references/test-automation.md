# Test Automation Review Lens

What to look for in UI, API, and integration test suites. The organising question behind all of it: **if this suite goes green, does that actually mean the product works?** A suite that answers "no" is worse than no suite, because teams ship on its say-so.

## Contents

1. Trustworthiness and flakiness
2. Waiting and synchronisation
3. Selectors and locators
4. Assertions
5. Test data and state
6. Structure and abstraction
7. Coverage and value
8. Execution and reporting

---

## 1. Trustworthiness and flakiness

The first thing to establish. Ask of the suite, and look for evidence in code and config:

- **Retry masking.** Blanket `retries: 3` or `@flaky` decorators applied suite-wide. Retries hide non-determinism rather than fixing it; targeted retries on genuinely external dependencies are defensible, suite-wide retries are a smell. Whether the team knows *which* tests only pass on retry is the key question.
- **Quarantine with no exit.** Skipped and disabled tests are fine as a temporary measure; look for `@skip`, `xit`, `.only`, `@Ignore`, `pytest.mark.skip` without a linked ticket or a date. Count them — a large silent skip list is a Critical finding, since the coverage the team believes it has does not exist.
- **`.only` / `fdescribe` committed to main.** Silently disables every other test in the file. Should be blocked by lint or CI grep.
- **Order dependence.** Tests that pass in sequence but fail in isolation or under shuffle. Signals: shared module-level state, tests numbered to force ordering, setup performed in test 1 that test 5 relies on.
- **Time and locale dependence.** `datetime.now()`, hardcoded timezones, tests that fail near midnight or month boundaries.

## 2. Waiting and synchronisation

The single largest source of UI flakiness. Hardcoded sleeps are both slow and unreliable — too short under load, wasteful otherwise.

Search for: `time.sleep`, `Thread.sleep`, `waitForTimeout`, `cy.wait(<number>)`, `browser.pause`, `setTimeout` in test code.

The fix is always the same shape — wait for the condition, not the clock:

```python
# Instead of
driver.find_element(By.ID, "submit").click()
time.sleep(3)
assert "Success" in driver.page_source

# Wait on the observable state change
WebDriverWait(driver, 10).until(
    EC.visibility_of_element_located((By.CSS_SELECTOR, "[data-testid='confirmation']"))
)
```

Also check: implicit and explicit waits mixed in the same suite (their interaction is notoriously unpredictable in Selenium), global timeouts set so high that a genuine failure takes minutes to surface, and network idle waits used where a specific element wait would be precise.

## 3. Selectors and locators

Ranked from most to least resilient: `data-testid` / dedicated test attributes > accessible role and label > stable semantic id > short scoped CSS > text content > long CSS chains > XPath with positional indexes.

Red flags worth citing individually:

- Absolute or positional XPath: `/html/body/div[3]/div[2]/span[1]` — breaks on any layout change.
- Generated class names: `.css-1x2y3z`, `.MuiButton-root-284` — change on every build.
- Text selectors on user-facing copy in a product that is localised or A/B tested.
- The same locator string duplicated across many files rather than defined once.

## 4. Assertions

- **Tests with no assertion.** They pass as long as nothing throws. Grep for test functions whose body contains no assert/expect call.
- **Assertions too weak to fail.** `assertTrue(response is not None)`, `expect(page).toBeTruthy()`, checking a 200 status while ignoring the body. These pass against a broken product.
- **Assertions too tight.** Full-object equality against payloads containing timestamps or generated ids, causing failures unrelated to the behaviour under test.
- **Multiple unrelated behaviours in one test.** When it fails, the cause is ambiguous and the fix is slower.
- **No failure message.** A bare `assert x == y` in a CI log costs the on-call engineer ten minutes of archaeology. Custom messages naming the expectation pay for themselves.

## 5. Test data and state

- **Hardcoded environment data** — production URLs, real customer records, specific database ids that exist only in one environment.
- **Credentials in the repo.** Any secret in test code or fixtures is Critical regardless of environment, since test credentials are frequently reused and repos get forked.
- **Tests that leave residue.** Records created and never cleaned up bloat environments and cause later failures. Look for teardown/fixture cleanup, and for cleanup that only runs on the happy path rather than in a `finally` or fixture-scoped teardown.
- **Cross-test data collisions.** Fixed usernames like `testuser@example.com` shared across tests that run in parallel. Factories with randomised or run-scoped identifiers are the fix.
- **Overreliance on a shared mutable environment** where any other team's activity can turn the suite red.

## 6. Structure and abstraction

- **Page objects that assert.** A page object should expose actions and state; putting assertions inside it couples the abstraction to one test's expectations and makes failures hard to trace.
- **Page objects that leak the driver.** Tests reaching through to `page.driver.find_element(...)` mean the abstraction has failed and the coupling it was meant to prevent has returned.
- **Copy-paste duplication** — the same login sequence written inline in thirty test files. One change to the login page then breaks thirty files.
- **Over-abstraction**, which is the opposite failure and just as real: four layers of inheritance where finding what a test actually does requires opening six files. Abstraction should reduce total effort, and past a point it stops doing so.
- **Helper modules that have become a dumping ground** — a `utils.py` of 800 unrelated lines is a sign that no one owns structure.

## 7. Coverage and value

Coverage percentage is a weak signal on its own; what matters is whether the important paths are covered.

- Which **critical user journeys** exist (signup, payment, core transaction), and does a test exercise each end to end?
- Is the suite **inverted** — many slow end-to-end tests, few fast unit tests? This is the classic ice-cream-cone shape and it produces suites that are slow, flaky, and expensive to maintain. Findings should identify specific E2E tests whose logic belongs at unit or API level.
- Are **negative and boundary cases** tested, or only happy paths? Error handling that no test exercises is error handling that does not work.
- **Redundant tests** — many tests covering the same path add runtime and maintenance without adding signal.

## 8. Execution and reporting

- **Runtime.** How long does a full run take, and is that compatible with how often it needs to run? A 90-minute suite will not gate merges; it will be skipped.
- **Parallelisation** — supported at all, and if so, are tests actually safe to run in parallel given the state issues above?
- **Failure diagnostics.** On failure, does the suite capture a screenshot, DOM snapshot, browser console log, network log, or request/response body? Without these, every failure requires a local reproduction. Adding artifact capture is usually a small-effort, high-value finding.
- **Environment reproducibility** — pinned browser and driver versions, containerised runs, or "works on my machine" configuration.
- **Reporting** — whether results reach the team somewhere they'll be seen, and whether historical results are retained so flakiness can be measured rather than guessed at.
