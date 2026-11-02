---
name: automation-code-review
description: Perform a senior-automation-engineer code review of a codebase, repo, PR/MR, or set of files, and produce a severity-ranked improvement report as a downloadable markdown file. Covers test automation suites, CI/CD pipelines, and general backend/app code quality. Use this skill whenever the user asks for a code review, an audit, a "second pass", a health check, a tech-debt assessment, a QA or test-suite review, a pipeline review, or asks what could be improved in their code — even if they don't use the words "code review" or "report". Also use it when the user points at a repo, branch, MR, zip, or folder and asks "what do you think of this" or "where are the problems".
---

# Automation Code Review

Review code the way a senior automation engineer with ten years of scar tissue would: someone who has been paged at 2am by a flaky pipeline, who has inherited a 4,000-test suite nobody trusts, and who knows that the most valuable review comment is the one that prevents an incident rather than the one that renames a variable.

The deliverable is a **downloadable markdown report** with severity-ranked, evidence-backed findings and a prioritised remediation plan.

## The stance to adopt

Be direct and specific. A vague review is worse than no review, because it burns the reader's attention without changing anything. Two habits carry most of the value:

**Cite evidence for every finding.** Reference `path/to/file.py:42`, name the function, quote the offending line if short. A finding with no location cannot be acted on and cannot be verified. If evidence is missing, the finding is a hypothesis — label it as one.

**Distinguish what is verified from what is inferred.** Reading a file gives certainty. Reading a filename and guessing does not. If a suite *looks* flaky because of hardcoded sleeps but no run history was available, say "likely flaky — not confirmed against run history". Overclaiming destroys the credibility of the whole report.

Respect the author. Whoever wrote this code had constraints not visible in the diff — a deadline, a legacy interface, a team decision. Critique the code, not the person, and assume there may be a reason before declaring something wrong.

## Step 1 — Get the code

Sources vary; adapt to whichever is present:

- **Repo via connector (GitLab/GitHub MCP)** — list the tree first, then fetch selectively. Never try to pull every file.
- **Uploaded zip/folder** — extract to the working directory and inspect from there.
- **Pasted snippets** — review what's given; note explicitly in the report that scope was a fragment, since architectural findings cannot be made from a fragment.
- **A PR/MR** — review the diff, but read enough surrounding context to judge whether the change is right, not just internally consistent.

If the scope is genuinely unclear (a monorepo with twelve services, no direction), ask which part matters before spending effort. Otherwise proceed and state assumptions in the report.

## Step 2 — Orient before reading

Resist opening files at random. Spend the first pass building a map, because findings only mean something in context:

1. **Shape** — languages, frameworks, test runners, build tooling, rough file counts per area.
2. **Entry points** — where execution starts: main modules, CLI entry, test runner config, pipeline definition.
3. **Configuration** — `package.json`, `pyproject.toml`, `pom.xml`, `.gitlab-ci.yml`, `Dockerfile`, `.env.example`, runner/browser config.
4. **Conventions** — how the existing code does things. Deviations from a codebase's own conventions are findings; deviations from your personal taste are not.

Useful reconnaissance commands when working on a local copy:

```bash
find . -type f -not -path '*/.git/*' | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20   # file types
cloc . 2>/dev/null || find . -name '*.py' -o -name '*.js' -o -name '*.ts' | xargs wc -l | tail -1
grep -rn "sleep\|Thread.sleep\|waitForTimeout\|time.sleep" --include=* . | head -30           # hardcoded waits
git log --format='%an' | sort | uniq -c | sort -rn | head                                     # who knows this code
git log --since='6 months ago' --name-only --format='' | sort | uniq -c | sort -rn | head -20 # churn hotspots
```

Churn hotspots deserve disproportionate attention: files changed constantly are where defects and friction concentrate.

## Step 3 — Choose review lenses and load them

Load only the reference files relevant to what's actually in the codebase. Each contains a detailed checklist of what to look for and the failure modes that matter.

| Lens | Load when the codebase contains | Reference |
|---|---|---|
| Test automation | UI/API test suites, page objects, fixtures, test runners | `references/test-automation.md` |
| CI/CD & pipelines | pipeline YAML, Dockerfiles, deploy scripts, IaC | `references/ci-cd.md` |
| Code quality & security | application/library/tooling source of any kind | `references/code-quality.md` |

Most reviews load at least two. Read the reference before forming findings in that area — the checklists encode failure modes that are easy to miss when reading code linearly.

## Step 4 — Calibrate severity honestly

Severity describes **consequence and urgency**, not how much the pattern annoys you.

| Severity | Meaning | Timeline |
|---|---|---|
| **Critical** | Causes outage, data loss, security exposure, or lets broken code reach production undetected | Before the next release |
| **High** | Actively costing the team time or trust right now — flaky gates, silent failures, unmaintainable hotspot | This sprint |
| **Medium** | Real risk or drag with a workaround in place | Next few sprints |
| **Low** | Quality-of-life improvement, no material risk | Opportunistic |

Two calibration guards:

- If more than roughly a fifth of findings are Critical, the scale has collapsed and the report will be ignored. Re-rank.
- A test suite that passes when the application is broken is **Critical**, not Medium. False confidence is worse than no coverage, because teams ship on it.

Pair every finding with effort so the reader can plan: **S** (under half a day), **M** (one to three days), **L** (more than three days or needs design work). The Critical-and-S findings are the report's most valuable output.

## Step 5 — Write the report

Use the structure in `references/report-template.md`. Follow it closely — the consistency is what makes these reports comparable across repos and over time.

Save as markdown to `/mnt/user-data/outputs/` with a descriptive name (`code-review-<repo>-<YYYY-MM-DD>.md`) and present the file. Keep the chat summary to a few sentences plus the headline counts — the report is the deliverable, not a summary of the report.

Sizing: aim for ten to twenty-five findings on a full codebase review. Fewer suggests a shallow pass; many more means the nits weren't filtered out and the important items will drown.

## What separates a useful report from a worthless one

Failure modes to actively avoid:

- **Restating the code.** "This function loops over users and validates each one" tells the author nothing they don't know. Every sentence should carry information the author lacks.
- **Lint noise.** Formatting, import ordering, and naming preferences belong to a linter. Recommending the linter is one good finding; listing two hundred of its violations is not a review.
- **Inventing problems to fill space.** If a codebase is genuinely solid, a short report saying so — with the specific evidence that makes it solid — is the honest and more useful output.
- **Reviewing files that were never read.** Never generate a finding about a file not actually opened. Line numbers must be real.
- **Advice with no fix.** "Improve error handling" is a complaint. A four-line code sketch showing the intended shape is a recommendation.
- **Ignoring what works.** The "what's working well" section is not padding. It tells the team which patterns to replicate and signals that the review was read in good faith rather than pattern-matched.

## Reference files

- `references/test-automation.md` — flakiness, waits, selectors, assertions, test data, isolation, coverage of critical paths
- `references/ci-cd.md` — pipeline structure, caching, secrets, gating, artifacts, containers, IaC
- `references/code-quality.md` — architecture, error handling, config, logging, dependencies, security, performance
- `references/report-template.md` — the exact report structure, with a worked example finding
