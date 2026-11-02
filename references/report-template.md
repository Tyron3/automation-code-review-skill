# Report Template

Follow this structure. Consistency is what lets a team compare reports across repos and track the same codebase over time.

Findings use IDs prefixed by area: `TEST-`, `CI-`, `SEC-`, `CODE-`, `PERF-`, `DEPS-`, numbered within each prefix.

---

```markdown
# Code Review — <Repository / Component>

**Reviewed:** <YYYY-MM-DD>
**Scope:** <what was examined — paths, branch, commit or MR ref>
**Reviewer:** Automated senior-engineer review
**Stack:** <languages, frameworks, test runners, CI platform>

---

## Executive summary

<Four to eight sentences for a lead who will read nothing else. State the overall
health, the single most urgent issue and its consequence, the dominant theme across
findings, and what to do first. Plain language, no jargon, no hedging.>

**Findings:** <N> total — <n> critical, <n> high, <n> medium, <n> low

---

## Scorecard

| Area | Rating | Note |
|---|---|---|
| Test reliability | Weak / Fair / Good / Strong | <one line> |
| Test coverage of critical paths | | |
| CI/CD & gating | | |
| Code structure | | |
| Error handling | | |
| Security | | |
| Dependency hygiene | | |
| Documentation | | |

<Omit rows that don't apply to this codebase rather than rating them blindly.>

---

## Findings at a glance

| ID | Finding | Severity | Effort | Area |
|---|---|---|---|---|
| SEC-001 | Database credentials committed in config | Critical | S | Security |
| TEST-001 | Suite passes with API unreachable | Critical | M | Test automation |
| CI-001 | Test failures don't fail the pipeline | Critical | S | CI/CD |
| ... | | | | |

<Sort by severity, then by effort ascending within severity — cheapest high-impact
work surfaces first.>

---

## Detailed findings

### SEC-001 — Database credentials committed in config

**Severity:** Critical  **Effort:** S  **Area:** Security

**Evidence**
`config/database.yml:14-17` contains a live username and password for the staging
database. The same values appear in `docker-compose.yml:22`. `git log` shows the
file was added in commit `a3f9c21` (2024-03-08) and has been present since.

**Why it matters**
Anyone with read access to the repository — including anyone who has ever cloned or
forked it — holds these credentials. Because the file is in history, deleting it now
does not revoke access. Staging databases frequently contain copies of production
data, so the blast radius may extend well beyond the staging environment.

**Recommended fix**
Rotate the credentials first, then remove them from the working tree and load from
the environment:

```yaml
# config/database.yml
username: <%= ENV.fetch("DB_USER") %>
password: <%= ENV.fetch("DB_PASSWORD") %>
```

Add `config/database.yml` to `.gitignore`, commit a `database.yml.example` with
placeholder values, and add a secret-scanning step to the pipeline so this is caught
automatically in future.

**How to verify**
`git grep -i password -- config/` returns no literal values, the application boots
with the variables set and fails loudly with a clear message when they are absent.

---

### TEST-001 — Suite passes when the API is unreachable

**Severity:** Critical  **Effort:** M  **Area:** Test automation

**Evidence**
`tests/api/test_orders.py:31-58` wraps each request in a try/except that logs and
returns `None` on failure, then asserts `assert result is not None or SKIP_ON_ERROR`.
With `SKIP_ON_ERROR` defaulting to true in `tests/conftest.py:12`, all 14 tests in
this module report success against a stopped service. The same pattern appears in
`tests/api/test_payments.py:44` and `tests/api/test_users.py:29`.

**Why it matters**
This is false confidence rather than missing coverage, which is the more dangerous of
the two. The pipeline gate reports green, the team ships, and the API contract is
never actually exercised. Any regression in these three modules reaches production
undetected.

**Recommended fix**
Remove the escape hatch and let connection failures fail the test:

```python
def test_order_creation(api_client):
    response = api_client.post("/orders", json=valid_order_payload())
    assert response.status_code == 201, f"expected 201, got {response.status_code}: {response.text}"
    assert response.json()["status"] == "pending"
```

If some environments legitimately lack the service, gate at collection time with an
explicit marker and a health check, so skipped tests are visibly skipped rather than
falsely passing.

**How to verify**
Stop the API and run the suite: it should report 14 failures, not 14 passes.

---

<Repeat this structure for each finding. Keep Critical and High findings at this level
of detail. Medium and Low findings can be more compact — evidence, why it matters, and
a one-line fix — to keep the report readable.>

---

## What's working well

<Three to five specific, evidenced observations. Not padding — this tells the team
which patterns to replicate and which parts of the codebase not to disturb.>

- **Fixture design in `tests/conftest.py`** — factory fixtures generate run-scoped
  unique identifiers, which is why the suite parallelises cleanly at four workers.
  Worth extending to the UI suite, which still uses fixed usernames.
- ...

---

## Remediation plan

### Now — before the next release
| ID | Action | Effort |
|---|---|---|
| SEC-001 | Rotate and externalise credentials | S |
| CI-001 | Remove `allow_failure` from the test stage | S |

### Next — this quarter
| ID | Action | Effort |
|---|---|---|
| TEST-001 | Remove the skip-on-error escape hatch across the API suite | M |

### Later — opportunistic
| ID | Action | Effort |
|---|---|---|
| CODE-004 | Extract shared HTTP client construction | M |

<Sequence matters: note where one item should precede another. Fixing the gate
before fixing the tests means the newly failing tests will actually be noticed.>

---

## Coverage and confidence

**Reviewed:** <paths and file counts actually opened>
**Not reviewed:** <what was out of scope and why — size, access, relevance>
**Could not verify:** <claims made without direct evidence, e.g. "flakiness suspected
from hardcoded waits but no run history was available", "branch protection settings
not visible from the repository">
**Open questions for the team:** <things a human needs to answer before some findings
can be confirmed or prioritised>
```

---

## Notes on using the template

**Sizing.** Ten to twenty-five findings on a full codebase review. Under ten suggests a shallow pass; over thirty means nits weren't filtered and the important items will drown.

**Fragments.** When only a snippet or single file was provided, keep the same structure but state the limitation prominently in the executive summary and be explicit that architectural findings cannot be made from a fragment.

**Clean codebases.** If the code is genuinely solid, say so and keep the report short. A three-finding report with strong evidence is honest and useful. Inventing findings to hit a count destroys trust in every other finding.

**The Coverage and confidence section is not optional.** It is the section that makes the rest of the report trustworthy, because it draws the line between what was verified and what was inferred.
