---
layout: post
title: "Six runs, eight findings, one live rollback: what broke when agents ran a GitLab DevSecOps pipeline end to end"
date: 2026-10-22 06:00:00 +0000
tags: [gitlab, devsecops, agents, ci]
---

Over one week in October I let GitLab Duo flows and CI jobs plan, build, review, merge, release, deploy, monitor and roll back a small service, with a written policy as the only authority. Six live runs produced eight defects in the automation itself, a false positive created by two correct fixes, and, on the last run, an automatic rollback that reached production 2 min 8 s after the monitor failed.

## The setup in one screen

The project is [gitlab-autopilot](https://github.com/vigohan95/gitlab-autopilot) (MIT). The application under it, `pulse`, is a small FastAPI site-health monitor with pinned dependencies, a container image, a wheel and two deployable environments, so every stage has something real to act on.

It covers nine DevSecOps stages, plan, create, verify, package, secure, release, configure, monitor and govern, over twelve CI stages in `.gitlab-ci.yml`. Everything that must be exact, from scanners and the policy gate to deployments and the rollback trigger, is a CI job with a script under `scripts/`. Everything that needs judgement is one of five custom Duo flows, F1 to F5, started by five triggers or, when a trigger cannot fire, by a CI job calling the API.

The policy is one file, `AUTOPILOT.md`: sections 1 to 9 in prose, then a YAML block that `scripts/policy.py` enforces in CI; if they disagree the YAML wins. Every agent note ends with `autopilot: rules=[…]`. The merge gate, §3, requires a green pipeline, the reviewer agent's signed verdict, no new High or Critical findings, at most 400 changed lines and 20 files, no draft, no conflicts and, since run 4, no unresolved review threads (§3.7).

"Zero humans in the loop" needs stating precisely. A human files the issue and, in the drills, presses Run on the monitor schedule, because GitLab's flow triggers fire only for actions a human performed. After that click nothing should ask a person to act. Only runs 4 and 6 held to that; the others are why the fixes exist.

The judged project's deployments are dry runs with no Render key, so "production going down" was simulated by pointing the monitor's `PULSE_PRODUCTION_URL` at `https://httpbin.org/status/503?probe=`.

## The drill, run by run

Times are UTC. Run 1 is from 2026-10-10, runs 2 to 5 from 2026-10-14, run 6 from 2026-10-16.

**Run 1, the first full journey.** A human filed issue #2; F1 posted a plan citing §2.1, §2.2 and §3.4 and opened MR !1; Code Review found nothing to comment on. `auto-merge` then refused twice: it counted all 44 pre-existing High/Critical base-image findings as new, and after that fix its finding key still carried the per-commit image tag. With the gate fixed and the MR rebased, !1 merged at 14:10Z and the Closer flow wrote the narrative at 14:18Z. The gate's notion of "new" was already the weak point.

**Run 2, the security loop (06:04 to 06:56).** A human opened MR !2 pinning `fastapi==0.125.0`. At 06:05 the F2 reviewer answered HUMAN REQUIRED, citing §2.2 and §3.3; at 06:07 `pip-audit` failed pipeline 2945456466 on 10 known starlette vulnerabilities, so `auto-merge` never ran. At 06:10 the F3 security fixer opened MR !3 into the feature branch with `fastapi==0.143.0`; F2 approved it at 06:13. Then the gate held the correct fix: its §3.3 baseline was the target branch's latest green pipeline, the feature branch had none, and every pre-existing base-image CVE looked new (finding 1). A human merged !3 at 06:50; `auto-merge` merged !2 at 06:56 with an empty net diff. The pin never reached `main`, but the loop had needed a hand.

**Run 3, the incident loop (06:59 to about 07:06).** The schedule was played at 06:59:46. At 07:02:30 `monitor` reported production DOWN after five attempts; at 07:03:04 the `incident` job opened Incident #3 with `v0.1.2` as the rollback target, and one second later wrote that the rollback could not start: `400 Bad Request … Insufficient permissions to set pipeline variables` (finding 3). At 07:04:25 the F4 incident responder's RCA named the real cause, a misconfigured probe target, and added that rolling back to `v0.1.2` would not change the result. That exposed finding 5: with `/version` unreachable the job had guessed "newest release", which was the release already running. And the "monitor-only" schedule had run `gemnasium-python-dependency_scanning` for 54.0 s and 47.6 s (finding 4).

**Run 4, hands-off, after the fixes (07:29 to 07:47).** A human filed issue #4 at 07:29:28 and touched nothing else. F1 posted the plan at 07:31 and opened MR !4 at 07:33:00; F2 approved at 07:35. At 07:37 GitLab Duo Code Review left a valid finding: the status-page assertions could not tell the new `uptime 24h` cell from the existing `uptime` cell, so the new value was untested. At 07:39:10 `auto-merge` cited "§3.2 reviewer agent verdict: APPROVED; §3.3 no new High/Critical findings (44 pre-existing…)" and merged at 07:39:18, straight past the comment (finding 6). `deploy-production` finished at 07:47:46: 9 min 50 s to merge, 18 min 18 s to production, nothing asked a human to act (one click did happen, a retry after the team's own docs push cancelled the main pipeline). The run also showed `review-approve`'s approve call answers 401, so §3.2 rests on F2's signed note, not an MR approval.

**Run 5, the follow-up, two interventions (07:53 to 09:34).** Issue #5 asked for the test run 4 lacked. F1 opened MR !5 at 07:56:31, test-only, +30 lines; F2 approved; Code Review found nothing left to say. At 07:59:50 `build-image` failed on a container-registry `500 Internal Server Error`; GitLab classified it as `script_failure`, nothing retried it, and the job sat until a human retried it at 09:15:17 (finding 8). At 09:18:38 `auto-merge` held the MR with "8 NEW High/Critical findings (absent on main)". The MR had the same 44 findings as `main`'s push pipeline and none new; the baseline had been the 08:08 schedule run, green and scanner-free since finding 4's fix (finding 7). The fix landed at 09:20:37, a human rebased at 09:21:54 because an MR pipeline runs the gate from the MR's own commit, and !5 merged at 09:28:02, reaching production at 09:34:08. Lead time 1 h 40 min 30 s, of which 75 min 27 s was a failed job waiting for a person.

**Run 6, release and live rollback (2026-10-16, 08:42 to 08:56).** Finding 5's fix meant production first had to run a release with an older release below it. Commit `9c8280d1` bumped `pulse` to 0.1.3; main pipeline 2954320350 created release `v0.1.3` at 08:46:47 and `deploy-production` recorded that commit at 08:48:41. The 503 URL was set, the schedule played, and `monitor` failed at 08:52:44. At 08:53:14 the `incident` job opened Incident #6, read production's last deployment from the Deployments API, resolved the deployed release as `v0.1.3` and the target as `v0.1.2`, created `rollback/v0.1.2/incident-6` and at 08:53:19 started pipeline 2954357168 on it with no pipeline variables. That pipeline deployed staging, verified it and deployed production at 08:54:52, with no scanners and no build. F4 posted its RCA at 08:54:46. Monitor failure to production rolled back: 2 min 8 s. Human touches after the schedule click: zero.

## The eight findings

| # | Symptom | Root cause | Fix | Test added |
|---|---|---|---|---|
| 1 | Gate held a correct security fix (run 2) | Baseline was the target branch's latest green pipeline; a feature branch had none | `abee55e`: baseline from target *and* default branch | `test_baseline_falls_back_to_default_branch` |
| 2 | A label outranked the reviewer (run 2) | `policy-check` set `autopilot::approved`; GitLab dropped F2's `needs-human` | `838637f`: newest signed verdict decides; label is a fallback | `test_reviewer_gate_respects_agent_verdict` |
| 3 | Rollback refused, 400 (run 3) | Needed a pipeline variable; minimum role is above Developer | `5f2d251`: tag and incident in the branch name `rollback/<tag>/incident-<iid>`, no variables | `tests/test_rollback.py`, asserts no variables |
| 4 | Monitor schedule ran a scanner (run 3) | Dependency Scanning template rule matches every branch pipeline on Ultimate | `914c354`: scanners on MR and `main` pipelines only | `tests/test_ci_rules.py` rule evaluation |
| 5 | Rollback target was the running release (run 3) | `/version` unreachable; "newest release" was guessed | `82614ff`: deployed release from `/version` or Deployments API; target is the next older release, else a human | `test_incident_main_replays_the_drill_without_a_guessed_target` |
| 6 | Merge went past a valid review finding (run 4) | Nothing in §3 read discussion threads | `6339d18`: §3.7, unresolved threads hold; waits up to 5 min for Code Review | `tests/test_review_threads.py` |
| 7 | False "8 NEW findings" hold (run 5) | Fixes 1 and 4 interacted: newest green `main` pipeline was scanner-free | `f3b096e`: baseline skips `schedule`, `trigger`, `pipeline`, `parent_pipeline`, `chat` | `test_baseline_skips_scanner_free_schedule_pipeline` |
| 8 | Registry 500 stalled an MR 75 min (run 5) | Registry error is `script_failure`; job-level retry cannot target it safely | `3c85877`: `retry: max 2` on infrastructure failures plus `scripts/registry_retry.sh` on every docker call | `tests/test_ci_rules.py` wrapper replays |

Finding 7 is the one to remember: fixes 1 and 4 each had a passing test, and no test combined them. The live run did.

## Three rules that came out of it

**A "new findings" baseline must come from a pipeline that actually ran the scanners.** Findings 1 and 7 are one mistake seen twice, and run 1's two refusals were its earlier form. A green pipeline is a baseline only if it produced findings to compare against. The gate now takes the newest green pipeline on the target or default branch whose source is not in `NON_SCANNING_SOURCES`, and a test evaluates the CI rules for scheduled, rollback, `main` and MR pipelines together.

**A policy label must never override a reviewer verdict.** Finding 2 was harmless in effect and wrong in principle: a deterministic check overwrote a reasoning step's conclusion through a side channel, GitLab's scoped-label replacement. Now the newest signed verdict note decides and the label applies only when no reviewer agent exists. The 401 on `review-approve` made the same point: rely on the artefact the agent signed, not on a platform state you assumed.

**Unresolved review threads hold the merge.** Finding 6 showed that a review the gate does not read is advice, and advice does not stop a merge. §3.7 makes any unresolved, resolvable thread a hold, counts a human's objection as much as Code Review's, and waits up to five minutes for a late review. Resolving a thread needs nothing above the Developer role, so it cannot deadlock; issue #5 was how the agents cleared their own finding.

## What is still unproven

The registry retry has never met a real registry error; finding 8's wrapper is tested against the run-5 message, timeouts, a denied push and a clean success, and nothing more.

The rollback that ran in 2 min 8 s deployed nothing. The judged project has no Render key, so `deploy-production` is a dry run that records a deployment, and the "outage" was an httpbin URL. The Deployments API, the branch convention and the pipeline rules were exercised; the Render path, and the caveat that a protected `RENDER_API_KEY` is invisible on an unprotected `rollback/*` branch, were not.

The service is tiny, and the 44 pre-existing High/Critical findings are tracked on `main`, not fixed; the gate's claim is "nothing new", not "nothing". The 401's cause is likely, not verified. After run 6 production runs `v0.1.2`, the oldest release, so the next incident correctly ends in "a human must roll back". And the judging freeze (§1.3, 2026-10-27T13:00Z) is enforced by `policy.py` and tested, but the date has not arrived.

## Reproducing the drill in your own project

Nothing below costs money. Locally:

```bash
pip install -r requirements-dev.txt
python -m pytest -q                              # 153 tests, no network
python scripts/check_ci_schema.py                # validates .gitlab-ci.yml against GitLab's schema
python scripts/policy.py --files pulse/app.py --labels autopilot --lines 42
```

Then, in a group with GitLab Duo Agent Platform enabled, create a public project from the repository and set these CI/CD variables:

| variable | value | flags |
|---|---|---|
| `AUTOPILOT_TOKEN` | project access token, scope `api`, Maintainer | masked, not protected |
| `AUTOPILOT_F2_CONSUMER_ID` | the `f2-reviewer` flow's consumer id (`scripts/start_flow.py`) | — |
| `RENDER_API_KEY` | optional; without it deploys dry-run | masked, protected; protect `rollback/*` too |
| `PULSE_STAGING_URL` / `PULSE_PRODUCTION_URL` | URLs the monitor probes | — |

Enable flow execution, foundational flows and external MCP tools in the group's Duo settings; register the four agents, five flows and five triggers from `.gitlab/duo/`; create a pipeline schedule on `main`:

```text
cron: 0 */2 * * *     description: monitor
```

To replay run 6 you need two releases and production deployed from the newer one's commit: bump `pulse/__version__`, push to `main`, wait for `release` and `deploy-production`. Then point the monitor at the always-503 endpoint and press Run on the schedule:

```text
PULSE_PRODUCTION_URL=https://httpbin.org/status/503?probe=
```

Watch Monitor > Incidents, the pipeline on `rollback/<tag>/incident-<iid>` with no variables, and Operate > Environments. Then delete the variable and play the schedule again; `monitor` should report no targets configured.

---

*Weiguo Han builds and breaks CI/CD automation; this field report is drawn from the public drill log of [gitlab-autopilot](https://github.com/vigohan95/gitlab-autopilot) (MIT).*
