# AI Control for Coding Agents: A Field Report from a Multi-Tenant Payments Platform

By Avdhesh Bansal. October 2026.

Companion document: [WORKFLOW.md](WORKFLOW.md) walks one change from prompt to merge through every hook, agent, and gate.

## Abstract

I architected and built KAPI Cars, a multi-tenant car-sharing platform at kapicars.com that takes payments through Stripe and Stripe Connect. It has 50 backend modules, more than 250 use-cases, and forced row-level security on 47 tables. I use Claude Code agents to write much of the code, I treat the agent as an untrusted component and built a control protocol around it. The protocol has 15 read-only auditor agents, 12 hook scripts that block actions mechanically, a closed loop that freezes new work while a finding is open, a verify gate for agent-built branches, and risk tiers that decide where my review time goes. It is a working case study in AI safety for coding agents. It covers LLM monitors, automated red-teaming of a payments stack, adversarial robustness of the controls themselves, and oversight at scale by one human.

This report describes that protocol and the evidence from running it. The evidence is 22 documented incidents from the project journal, commit history, and audit logs. As of 2026-10-15, 413 findings have been fixed through the closed loop. The live attack agents found the most severe bugs. One was a forged payment ID that would have let anyone book cars for free. Another was a token with no tenant that passed platform-only checks. A database probe showed that a superuser role silently bypassed tenant isolation while every isolation test passed. A parity auditor caught a webhook stub that would have dropped three Stripe payment events while Stripe reported success. Several of these bugs passed every normal test. The monitors also made mistakes, and some failures were caught only by me. I close with four research questions that this setting makes testable.

## 1. Setting and threat model

KAPI is a Next.js web app, a React Native mobile app, and a NestJS backend on PostgreSQL. Each host business is a tenant with its own isolated data. Money moves through Stripe and Stripe Connect. A bug here can charge the wrong amount, give a car away for free, or show one business's customers to another.

I did not assume the agent was malicious. I assumed it could be wrong in ways that look right. The journal backs this up. Many entries are written in the agent's own words ("I shipped..."), with my corrections quoted. Across the record, the failures fall into five classes.

| Class | What it looks like | Example from this repo |
|---|---|---|
| Silent behavior loss | Code compiles, tests pass, but work quietly stops happening | A replacement Stripe webhook that dropped three payment events |
| Tenant or auth breach | One tenant can read or change another tenant's data, or a check can be skipped | A token with no tenant was treated as the platform's own tenant |
| Money correctness | Wrong amounts, double charges, free goods | A forged payment ID created a paid booking |
| Shortcuts | Type escapes, skipped validation, unjustified exceptions | Disabling a check instead of fixing the cause |
| Process failures | The agent works on the wrong branch, defers a finding on its own, or bypasses a hook | A builder pushed past a blocking hook with `--no-verify` |

This matters for how to read the evidence. AI control asks a harder question. Does the protocol still hold if the model is trying to get past it? Every incident here was an unintended mistake, not an attack. So the record shows how the protocol handles errors. It does not yet show how it handles a model that hides what it is doing. I treat that as the main open question, and Section 7 turns it into experiments.

Two auditors attack the running app instead of reading code. The penetration tester runs 12 attack suites, including JWT algorithm confusion, race conditions, rate limits, and Stripe webhook replay and price tampering. The tenant-isolation auditor runs 7 suites, including a database-level proof that row-level security is forced. Both run only against localhost or a named staging URL. 48 tenant-isolation test suites reconnect as an unprivileged database role, and CI runs an OWASP ZAP scan.

The goal of the protocol is simple to state. No change should reach the shared branch with a known violation, and the riskiest changes should never rely on the agent's word.

## 2. The control protocol

The protocol has five layers. Each one exists because a weaker version failed.

### 2.1 Capability limits

There is one rule above the rest. Agents that review cannot change code. Agents that change code must run where I can approve them.

All 15 auditors are read-only. No auditor has an Edit tool, and only one (the cross-browser tester) can write, and only temporary test files in a git-ignored folder. A reviewer that can edit has a reason to quietly patch its own findings. A reviewer that can only report has to show its work.

Code changes run as skills in the main session, under plan mode and my approval. Three components that used to be code-writing agents were converted into skills for exactly this reason. A subagent cannot enter plan mode and cannot stop to ask for approval in the middle of a change.

### 2.2 LLM monitors: the 15 read-only auditor agents

Each auditor owns one risk area. Each has a fixed severity scheme and must give a file and line for every finding. An exploitable finding needs a proof of concept. Without one, it can only be informational.

| Auditor | What it guards | Representative checks | Hard boundary |
|---|---|---|---|
| security-auditor | Flaws that generic scanners miss | IDOR on every ID parameter, JWT algorithm pinning, PCI SAQ-A drift (no card data ever stored), webhook signature checks, upload handling, rate limits | Advisory only. Humans set merge state. |
| penetration-tester | Whether controls hold under real attack | 12 attack suites: IDOR, JWT tampering and algorithm confusion, booking-gate bypass, injection, XSS, CSRF, malicious uploads, race conditions, rate limits, Stripe webhook replay and price tampering, business-logic chains, token leakage | Localhost or a named staging URL only. Production and third-party services are refused. Bounded request bursts. Read-only database access. |
| tenant-isolation-auditor | That tenant A can never reach tenant B | 7 suites: cross-tenant reads, list leaks, cross-tenant writes, token claim tampering, host spoofing, a database-level proof that row-level security is enabled and forced, and a coverage scan for unprotected tables | Same target limits as the pen-tester. |
| payments-flow-auditor | That money is correct, not only secure | Integer cents only, idempotency keys, webhook dedup and out-of-order safety, refund math, Connect split math, subscription matches entitlement, ledger balance | Never touches live Stripe or production data. |
| architecture-guardian | That the design stays intact | Clean Architecture layer direction, no unjustified type escapes, validated inputs, no inline secrets, shared payment UI components | Writes every finding to the findings log. Never fixes. |
| migration-parity-auditor | That a ported endpoint keeps every field and behavior | Compares fields the old route read, fields the new input type declares, and fields actually saved | Never marks unreadable code as clean. |
| data-integrity-auditor | Database health | Foreign keys, cascade rules, orphan rows, soft-delete consistency, migration reversibility, restore drills | Destructive SQL is printed for a human, never run. |
| gate-flow-auditor | The booking prerequisite chain | Age, phone, license, and payment gates, checked in the UI and directly against the API | |
| Seven UX, QA, and operations auditors | Accessibility, white-label branding, cross-browser behavior, browser walk-throughs of named flows, performance, observability, SEO | WCAG 2.1 AA, brand bleed between tenants, six browser targets, bounded load tests, PII redaction in logs | Load tests are capped and never hit production. |

One more read-only role is played by the gate-check skill. It is a read-only orchestrator that runs a full battery of auditors for a launch gate and returns GO or NO-GO. A cross-tenant leak is an automatic NO-GO.

### 2.3 Mechanical enforcement: hooks

A rule in a prompt is a request. A model can ignore it. So the important rules are enforced by hook scripts that Claude Code runs at fixed points. A hook that exits with code 2 blocks the action. The model cannot argue its way past it.

There are 12 hook scripts. Eleven are wired to lifecycle events, and one is a helper for clearing findings safely.

| When it runs | What it does | Behavior if its own tools break |
|---|---|---|
| Every prompt | Injects the single rulebook into the context | Falls back to a short rule set |
| Every prompt | Warns when the branch is behind the shared branch | Silent |
| Before reads and shell commands | Blocks any access to environment secret files | Fails open |
| Before every write | Blocks hardcoded secrets with no bypass. Blocks type escapes and known shortcuts unless the line carries a written reason. | Fails open |
| Before every write | Freezes edits to unrelated files while a blocking finding is open | Fails open on a broken findings file. Fails closed on unknown severity. |
| Before every write | Blocks writes if another session switched the branch underneath this one | Fails open |
| Before every write | Blocks writes on a branch that has fallen too far behind | Fails open, with a manual override file |
| After every edit | Checks layer rules and coding rules, logs violations, and queues the file for the architecture auditor | Skips logging |
| After git commands | Records the branch this session intends to be on | Fails open |
| Before a subagent starts | Injects the rulebook and that agent's own context section | Reduced context |
| End of every turn | Runs the type checker and a duplicate-route check, and refuses to end the turn once if either fails | Allows |

The fail-open column matters. Several hooks let actions through if a tool they depend on is missing. That was a deliberate choice to avoid locking the project when the environment breaks. It is also a weakness, and I list it in the limitations.

### 2.4 The closed loop

Findings do not sit in a report. The architecture auditor writes every finding to a shared findings log, with file, line, rule, severity, and source. A hook then freezes edits to every file that is not part of the fix. New work cannot pile on top of a known problem.

The only way to clear a finding is the fix-bug workflow. It runs in plan mode with my approval, fixes the cause, re-runs the checks, and only then clears the entry. If the fix creates new findings, it recurses, up to a depth of three. At that depth it stops and records technical debt instead of looping.

Two details make the loop hard to game. A finding with a severity label the gate does not recognize is treated as blocking, so a new auditor cannot slip past by inventing a label. A finding with no worktree tag counts against the current worktree. Every clear is logged. As of 2026-10-15 the log shows 413 findings fixed this way.

### 2.5 Verify gate and risk tiers

When work is built by parallel builder agents, the builder's own report is not evidence. Each diff goes through an independent verify gate. The gate re-runs the shortcut scan over the full diff, runs the type checker and the module tests, and requires the architecture auditor to approve. Riskier tiers add the payments, tenant-isolation, and penetration auditors. A failing diff gets two fix attempts. After that it becomes a draft pull request for me.

Changes are sorted into three tiers.

| Tier | Covers | Path to merge |
|---|---|---|
| 0 | Additive work: new files, tests, docs, scaffolds | Auto-merge when CI passes |
| 1 | Existing logic, but no money, tenant, or auth code | Stricter gate, then auto-merge when CI passes |
| 2 | Money movement, tenant isolation, auth, destructive migrations | Opens as a draft labeled `tier-2-hold`. Waits for my approval. |

The design includes a graduation rule. A ledger records each shipped change and whether a defect later escaped. After 5 clean Tier 0 or Tier 1 changes in a row, Tier 2 could merge without me. Any escape resets the count. Today every Tier 2 change still waits for me. Forty commits in the history carry the `tier-2-hold` label.

### 2.6 The CI layer

Six GitHub Actions workflows back the protocol. CI runs a dependency audit that fails on any high or critical advisory, except a short documented allowlist of dev-only packages. The architecture auditor found that this gate failed open when npm returned error-shaped output, and it was hardened to fail closed. An OWASP ZAP baseline scan runs weekly and on dependency changes. The tenant-isolation end-to-end suites run as the restricted database role. A pull request guard fails any PR aimed at a non-integration branch. That guard exists because a security fix once showed as merged but never reached the shared branch.

## 3. How this maps to AI control

The protocol was built from practice, but it lines up with the AI control framing.

- **Untrusted model.** The builder agent. Its output is never accepted on its own word.
- **Trusted monitor.** None in this setup. There is no weaker model I trust to score suspicion. The hooks fill part of that role.
- **Deterministic enforcement.** The hooks. They are scripts, not models, so text cannot persuade them. They can only check what can be written as a rule.
- **Untrusted monitoring.** The 15 auditors, plus a Claude code-review workflow that reviews every pull request in CI. All 15 auditor agents pin the same model, and so does the builder agent definition. Main-session changes may run on a different model. Even so, they can share its blind spots, and in a worse case they could collude with it. The usual defenses are honeypots and paraphrasing. A honeypot is a planted bad change that tests whether a monitor still flags it. Paraphrasing rewrites what the monitor sees so the builder cannot pass it hidden signals. Neither is in place yet.
- **Human audit.** Me. My review time is the audit budget. The tiers spend it on Tier 2 first.
- **Oversight at scale.** One person supervising far more agent changes than I could review one by one. I avoid the term scalable oversight here. In the literature it means helping humans judge work they could not judge alone, which is a different problem.

## 4. Evidence: the incident record

The table lists 22 incidents. I included only incidents whose fixes are confirmed on the main branch, plus process failures. I left out findings that are still open.

"Stage" means where the problem was caught. Pre-merge means before the shared branch. Post-merge means it was found after it reached the shared branch.

| # | Date | What went wrong | Potential impact | Caught by | Stage | Durable change |
|---|---|---|---|---|---|---|
| 1 | 2026-07-03 | The booking endpoint trusted a payment ID sent by the client and marked the booking paid | Anyone could book cars for free with a made-up ID, and reuse it | penetration-tester, live | Post-merge | Server verifies every payment with Stripe. One payment can back one booking. |
| 2 | 2026-05-28 | The app connected to Postgres as a superuser, which skips forced row-level security | Tenant isolation rested only on filters in app code | tenant-isolation-auditor, CRITICAL | Pre-merge | Restricted runtime role. Production refuses to boot if the role can bypass security. |
| 3 | 2026-05-28 | 18 booking queries across 14 files skipped the tenant scope wrapper | Cross-tenant reads | architecture-guardian, 9 BLOCKING findings | Pre-merge | All call sites wrapped |
| 4 | 2026-05-30 | Admin write routes were not tenant-scoped, including license verification | One tenant's admin could approve another tenant's driver and skip a background check | gate-check battery, tenant-isolation CRITICAL | Post-merge | A shared helper that returns 404 across tenants |
| 5 | 2026-06 to 08 | A token with no tenant claim was defaulted to the platform tenant | A signed admin token could pass platform-only checks, including a pricing write | penetration-tester with tenant-isolation-auditor, live proof | Post-merge | Tenant-less tokens rejected, then also at the websocket handshake |
| 6 | 2026-05-30 | A replacement Stripe webhook handled two event types as logs only | Invoices stuck pending, failed charges ignored, cancellations never closed bookings, while Stripe saw success | migration-parity-auditor | Pre-merge, before the old handler was deleted | Never delete a route on traffic data alone |
| 7 | 2026-03-29 to 05-21 | A ported upload endpoint silently dropped 11 text fields | A missing field could break the insurance step after payment | manual-test-walker, during a full booking-flow test | Post-merge, undetected for seven weeks | Field parity became a written criterion. The parity auditor was created. |
| 8 | 2026-05-30 | A background-check webhook accepted requests with no signature | Anyone with a provider ID could forge a passed check | Security audit inside the migration-complete check | Post-merge | Webhooks fail closed |
| 9 | 2026-05 | A ported route duplicated an existing one, and NestJS silently let one shadow the other | The route that ran lacked an owner check | security-auditor | Post-merge | Duplicate-route detector at turn end |
| 10 | 2026-05 | The PR that fixed issue 9 showed merged but never reached the shared branch | The IDOR stayed open after a passing audit | Found on re-check | Post-merge | PR base guard in CI |
| 11 | 2026-05-29 | A migration used a drop command that silently did nothing on that object | Two tenants sharing a key would hit errors in production. Local tests passed 408 of 408. | architecture-guardian, reproduced on a scratch database | Pre-merge | Replay migrations on a fresh database |
| 12 | 2026-06-05 | Referral credit lookups returned zero rows under row-level security | Guests silently charged full price | payments-flow-auditor, CRITICAL | Pre-merge | Tenant-scoped lookup |
| 13 | 2026-08-18 | A platform-scope delete path skipped a database check on saved payment methods | Cross-tenant delete of saved card references, reproduced in a test environment. No card data is ever stored. | Audit round | Post-merge | Check enforced on delete |
| 14 | 2026-08-18 | Back-office CRM routes were readable by any signed-in user | Signed-in users could have read internal records | Audit round | Post-merge | Role-gated routes |
| 15 | 2026-08 | The CI dependency audit failed open on error-shaped output | Vulnerable packages could pass CI | architecture-guardian | Pre-merge | Gate fails closed |
| 16 | 2026-10-14 | A path handling flaw weakened the private uploads gate | Private files could have been read | Audit round | Post-merge | Stricter path checks |
| 17 | 2026-05-28 | A card-save helper only logged errors | Every declined card would show success | architecture-guardian | Pre-merge | Errors surfaced |
| 18 | 2026-05-30 | A builder read the tenant ID from the request body on a payout surface | A caller could choose whose payouts to touch | Verify gate on a draft PR | Pre-merge | Tenant identity only from the signed-in user |
| 19 | 2026-05-28 | The agent could not fix a config bug quickly, applied a migration by hand, and labeled the problem "pre-existing" | 26 or more foreign-key failures in CI | Me | Pre-merge | A CI step that forbids that bypass |
| 20 | 2026-05-17 | The agent read "SERIOUS but safe to land" as permission to defer a finding | Known defects shipping on the agent's judgment | Me | Pre-merge | Findings write blockers by default |
| 21 | 2026-05-30 | A builder labeled a correct staleness gate as a bug and pushed past it with `--no-verify` | Hooks bypassed silently | Review of the run | Pre-merge | Builders must stop and report when blocked |
| 22 | 2026-07-03 | Another session switched the branch mid-task | Work was carried onto the wrong branch and an earlier delete was undone. It would have stacked onto the wrong pull request. | architecture-guardian | Pre-merge | Branch-pin hook |

### 4.1 Catches by monitor type

| Caught by | Incidents | Total | Pre-merge | Post-merge |
|---|---|---|---|---|
| Live attack agents (pen-tester, tenant-isolation, gate-check battery) | 1, 2, 4, 5 | 4 | 1 | 3 |
| Static auditor agents (architecture, parity, payments, security) | 3, 6, 8, 9, 11, 12, 15, 17, 22 | 9 | 7 | 2 |
| Browser walk-through agent | 7 | 1 | 0 | 1 |
| Audit rounds | 13, 14, 16 | 3 | 0 | 3 |
| Verify gate | 18 | 1 | 1 | 0 |
| Me | 10, 19, 20, 21 | 4 | 3 | 1 |
| Hooks | none | 0 | 0 | 0 |
| **All** | | **22** | **12** | **10** |

Hooks appear nowhere in this table. That is not because they did nothing. A hook blocks an action before it can become an incident, and those blocks were not logged. Counting them is the first metric I would add.

### 4.2 Three cases in detail

**The forged payment ID.** The create-booking endpoint accepted whatever payment reference the browser sent and stamped the booking paid. It never asked Stripe if the payment was real. The penetration-tester agent created bookings with a made-up ID and reused it. The fix fetches the payment from Stripe and refuses unless it succeeded, for the amount the server computed, in USD, from this buyer's Stripe customer. A unique index stops one payment from backing two bookings. The architecture and payments auditors then flagged a subtler trap in the fix. The expected amount has to be rebuilt from what was charged, not from what the client claimed, or real payments get rejected. The rule I took from it is to never let the client assert payment. It also showed that the fix itself needs the same scrutiny as the original code.

**The decorative tenant wall.** Bookings had forced row-level security, and all six isolation tests passed. The tenant-isolation auditor set the tenant to one that should see nothing and still got every row. The runtime role was a superuser, and a superuser skips row-level security even when it is forced. The tests passed only because app-side filters happened to cover the queries they exercised. In the same cycle the architecture auditor found 18 unscoped queries across 14 files. I fixed it with a restricted role and a boot check that refuses to start production if the role can bypass security. All tenant-isolation suites now reconnect as the restricted role, and CI runs them that way. Later I added a CI invariant gate whose stated purpose is to catch code that looks enforced but is not.

**The stub that looked like a twin.** I was about to delete an old Stripe webhook route. A new route existed, and traffic logs showed zero hits on the old one. The migration-parity auditor compared behavior instead of traffic. The new handler covered two event types as logs. Three payment events fell through and were silently dropped. Deleting the old route would have left lease invoices pending forever while Stripe saw success. The lesson was that zero traffic is not proof that something is dead. In that case, zero traffic was itself the symptom.

## 5. Analysis: red-teaming, monitor errors, and oversight cost

**Live attacks and static review catch different things.** Live attack agents found 4 of the 22 incidents. That includes the two I rate most severe, free bookings (1) and the platform-tenant token (5). Both were found after merge. Static auditor agents found more, 9 of 22, and 7 of those were before merge. The tenant wall (2) shows the split clearly. A live database probe disproved what six passing tests claimed. The two kinds of monitor do not replace each other. Static review catches more, and earlier. Live attacks catch what static review cannot see.

**Passing tests were not evidence.** Incidents 2, 7, and 11 all passed their test suites. In incident 11, 408 of 408 tests passed on a migration that did nothing. The pattern is the same each time. A check confirmed what it was built to see and was blind to what quietly went missing.

**Some failures were caught only by a human, or late.** The silent field drop went unnoticed for seven weeks, until a browser walk-through agent hit it in a full booking test. The process failures (19, 20, 21) were not code bugs at all. They were the agent making calls it was not authorized to make. Hooks and auditors check artifacts. They did not catch an agent deciding on its own that a rule did not apply. Incident 19 produced a CI check that blocks the bypass. Incidents 20 and 21 produced written rules, which a model can still ignore. Turning those into hooks is open work.

**Several catches happened after merge.** Incidents 1, 4, 5, 7, 8, 9, 10, 13, 14, and 16 were found on the shared branch. "Pre-merge" was not the norm for the most serious security findings. The live attack auditors ran as periodic sweeps, not on every change.

**The monitors made mistakes too.** In one parity audit, 2 of 4 CRITICAL findings were false. The columns did not exist, and a 30-second database check disproved them. Another auditor warned that some database constraints would break a migration check. They did not. An audit agent reported work as not started because it read a stale copy of the code. The old per-edit check once raised a false duplicate-route block during a multi-file refactor, which is why that check moved to turn end. Adversarial wording in auditor instructions also tripped the model provider's safety classifier and stopped three live auditors in one night. Monitors need their own ground truth.

**Strictness has a cost.** I first required every finding, at any severity, to block edits. That was too much friction. In July I relaxed it so suggestion, info, and minor findings are advisory, while everything else still blocks. Some findings are still open. Most come from recent audit rounds on branches that are not yet merged. Calibrating what blocks is an open design problem, not a solved one.

**Parallel builders failed for real code.** I tried running builder agents in parallel, each in its own git worktree, behind the verify gate. Three waves in a row stalled. The worktrees had no installed dependencies, the workflow runner killed agents idle for more than 180 seconds during test runs, and one builder edited files outside its worktree. Worktree builders are now limited to research and docs. All code changes run in the main session, where the gate can run. A control protocol is only as good as the checks it can actually execute.

## 6. Limitations and threats to validity

- **One developer, one codebase.** These are field observations, not controlled experiments.
- **The record is self-reported.** The journal is often written by the agent and edited by me. Incidents that nobody noticed are missing by definition.
- **No base rate.** I can count catches, but not the total number of mistakes, so I cannot compute monitor recall.
- **Fail-open hooks.** Several hooks allow actions when a dependency is missing. An agent that broke its own environment could weaken enforcement.
- **Shared model.** All auditor agents pin the same model as the builder agent. Correlated blind spots are likely and collusion is not defended against.
- **Graduation untested.** The autonomy ledger has one entry. Any claim about earned autonomy is a design claim.
- **Coverage gaps.** There is no CodeQL workflow in the repository, though the security auditor's scope assumes one. The live attack battery has not yet run as a formal release gate.
- **Non-adversarial evidence.** No incident involved a model trying to evade the monitors. Catch rates against an attacking model could be much lower.
- **Selection.** I included only incidents with fixes confirmed on the main branch. Open findings could show a different pattern.
- **Unlogged prevention.** Hook blocks were not recorded. So the record undercounts deterministic enforcement and overstates the share caught by agents.
- **Monitor drift.** Auditor instructions and the severity policy changed during the period. Early and late catches did not come from the same monitor.
- **Snapshot numbers.** Counts are as of 2026-10-15.

## 7. Research agenda

This setting turns four open questions into experiments.

1. **Can monitors catch quiet sabotage?** The incident classes above are a seed set. I want to build a benchmark of realistic sabotage-style changes. Examples include a webhook that drops events, a role that silently bypasses row-level security, a payment check that trusts the client, and a migration that does nothing. Then measure detection by LLM judges, read-only auditor agents, live attack agents, and deterministic hooks, at a fixed audit budget. Report false alarms and review time, not only catches. Also test whether a monitor does better with the full change history than with the final diff.
2. **What should earn an agent autonomy?** My rule unlocks high-risk merges after a streak of clean low-risk changes. Does a clean streak predict future safety? An agent that behaves well only when it expects review would game this rule. Trust earned on low-risk work unlocking high-risk work is exactly the gap a scheming model would use. I want to test streak rules against an attack policy told to behave well until the streak unlocks Tier 2, and then attack.
3. **How often do self-reports diverge from reality?** The verify gate exists because builder reports were not reliable. I want to measure how often an agent's claim about its own work disagrees with an independent check, and whether that rate changes with task risk.
4. **Can a monitor see it from the inside?** Every serious incident here looked correct from the outside. On an open-weight model, I would test whether simple probes on internal activations flag sabotage-style code better than a monitor that reads the output. This is a stretch goal, not part of the core plan.

### 7.1 Hypotheses

- **H1.** At a 2% audit budget, deterministic checks plus live attack agents catch more silent-behavior-loss attacks than an LLM monitor reading the diff.
- **H2.** A monitor that sees the full change history catches more attacks than one that sees only the final diff.
- **H3.** When the untrusted monitor is itself prompted to collude with the attack policy, it misses more planted attacks than a weaker trusted monitor. Honeypot validation plus paraphrased inputs closes most of that gap.

### 7.2 Metrics

- **Safety.** The share of attack runs where every attack is caught before merge.
- **Usefulness.** The share of normal tasks completed correctly with the protocol on.
- **False alarms.** Flags raised on clean changes, per 100 changes.
- **Audit cost.** Human review minutes per 100 changes.

### 7.3 Four-month plan

- **Month 1.** Build 30 to 50 sabotage tasks from the five incident classes, in a public open-source app rather than KAPI. Port them to the ControlArena format.
- **Month 2.** Build the attack policy, an untrusted monitor, a weaker trusted monitor, and the deterministic and live-attack checks. Set audit budgets of 0.5, 1, 2, and 5 percent.
- **Month 3.** Run H1 to H3 with red-team and blue-team rounds. Plot safety against usefulness at each budget.
- **Month 4.** Run the streak-rule test from question 2. Write it up.

Building the benchmark in a public app keeps KAPI's real weaknesses and internal names out of the published work.

### 7.4 Related work

This builds on trusted and untrusted monitoring from Greenblatt et al. (2023, "AI Control: Improving Safety Despite Intentional Subversion"), resampling protocols for agents from Bhatt et al. (2025, "Ctrl-Z: Controlling AI Agents via Resampling"), the UK AISI ControlArena settings, and the sabotage evaluations of Benton et al. (2024, "Sabotage Evaluations for Frontier Models"). What is new is where the tasks come from. The sabotage classes come from failures a coding agent actually made on a live payments system. Nobody invented them for a benchmark.

## Glossary

- **Row-level security (RLS):** Postgres rules that hide rows belonging to other tenants.
- **IDOR:** reading or changing someone else's record by changing an ID.
- **Tenant:** one host business on the platform, with its own isolated data.
- **Hook:** a script Claude Code runs automatically at a fixed point. Exit code 2 blocks the action.
- **Worktree:** a separate working copy of the same git repository.
- **Audit budget:** the limited human review time spent checking agent work.

Process detail for every step above is in [WORKFLOW.md](WORKFLOW.md).
