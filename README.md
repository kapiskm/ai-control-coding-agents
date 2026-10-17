# AI Control for Coding Agents on a Live Platform

By Avdhesh Bansal. October 2026.

An AI safety case study in AI control and scalable oversight for coding agents. It covers LLM monitors, hard blocks, verify gates, earned autonomy, and where red-teaming and interpretability fit next.

I architected and built KAPI Cars, a live multi-tenant car-sharing platform at [kapicars.com](https://kapicars.com). It moves real money through Stripe. Each host business gets its own isolated data. I use Claude Code agents to write and change code on it. I built the platform alone, and I learned that I could not trust those agents on their own. This post explains the control system I built around them to keep their work safe and true to my architecture. It has 15 auditor agents, 12 blocking hooks, a verify gate on every agent change, and 20 tenant-isolation test suites. It stopped a deletion that would have silently dropped three kinds of payment events. I also cover a database bug that passing tests had hidden, what failed, and what I still do not know.

## The problem

A coding agent can ship a change that looks right and passes the tests. That change can still break payments or leak one customer's data to another. The agent does not need to be malicious. It only needs to be wrong in a quiet way.

This happened to me. My tenant-isolation tests all passed. I believed one host could never see another host's data. Then I found that the database guard had been off the whole time. The app connected to Postgres as a superuser. A superuser skips row-level security, even when it is forced on. The tests passed only because a filter in the app code happened to catch the bad queries. Green checks had hidden a bug that could have ended the company.

I fixed it with a restricted database role that cannot bypass the rules. I also added 20 isolation test suites that reconnect as that role, so a leak now fails the build. But the bigger lesson stayed with me. A passing check is not proof. That is the idea behind my whole workflow. Treat the agent as untrusted, and design checks that still hold if it is wrong.

## Rule 1. The monitors cannot change code

I built 15 auditor agents. Each one reviews a single risk area. There is one for security, one for payments, one for tenant isolation, one for architecture, and more. They can read code and run checks, but their tool lists do not let them edit the code. They cannot fix what they find. Only one builder agent changes code, and its work goes through the checks below.

This split matters. A reviewer that can also edit has a reason to quietly patch its own findings. A reviewer that can only report has to show its work to me. In AI control terms, the auditors are LLM monitors and the builder is the untrusted model.

The system guards more than safety. It also keeps the code true to the architecture I designed. The architecture auditor checks every change against Clean Architecture layers and the project's conventions. It blocks a change if the core business logic imports a framework or a database library, or if an API controller skips the use-case layer and talks to the database directly. A fast-moving agent tends to take these shortcuts first. Each one alone looks harmless. Over hundreds of changes they would turn a clean design into a tangle.

## Rule 2. Blocks that do not depend on the model obeying

Rules in a prompt are requests. A model can ignore them. So I enforce the important rules with 12 hooks. A hook is a small script that Claude Code runs automatically before or after an action. If a hook exits with an error, the action is blocked. The model cannot talk its way past it.

Some examples:

- A scan runs before any file is written. It blocks hardcoded secrets and known shortcuts. There is no bypass.
- When an auditor files a blocking finding, a hook freezes edits to unrelated files until that finding is fixed. New work cannot pile on top of a known problem.
- If a finding has a severity label the hook does not recognize, the hook treats it as blocking. A new auditor cannot slip past the gate by using a new label.
- One rulebook holds the project's discipline. Hooks inject it into every prompt and every subagent, so each agent starts from the same rules.

## Rule 3. Do not trust self-reports

An agent will often say its work is done and safe. I do not take that as evidence. Every change built by an agent goes through an independent verify gate. The gate re-scans the diff for shortcuts, runs the type checker and the tests, and asks the architecture auditor to approve. Changes that touch money or tenant data also go to the payments and tenant-isolation auditors.

If the gate fails, the agent gets two tries to fix it. After that, the work becomes a draft pull request for me to review. Nothing merges dirty.

## Rule 4. Graduated autonomy. Trust is earned, and it can be lost

Not every change carries the same risk. I sort changes into three tiers.

- Tier 0 is additive work, like new files, tests, and docs. It can merge on its own after the gate passes.
- Tier 1 touches existing logic but not money, tenant data, or login. It gets a stricter gate.
- Tier 2 covers money movement, tenant isolation, login, and destructive database changes. These wait for my approval.

The system keeps a ledger of every shipped change. After 5 clean Tier 0 or Tier 1 changes in a row, Tier 2 can start to merge without me. Until then, every Tier 2 change waits for my approval. If a defect slips past the gate and is found later, the count resets to zero. Trust is earned slowly and lost at once.

## How this maps to AI control

The builder agent is the untrusted model. The hooks fill the trusted role. They are not a weaker model scoring suspicion, as a trusted monitor would be. They are plain scripts, so clever text cannot persuade or fool them. The 15 auditors are untrusted monitoring. They run on the same kind of model as the builder. So they could share its blind spots, or in a worse case, collude with it. My review time is the audit budget. The tiers decide where I spend it. Tier 2 gets my eyes first. This is scalable oversight in practice. I can supervise far more agent changes than I could review one by one, because my review time goes where the risk is highest.

## What it caught

An old Stripe webhook handler looked safe to delete. A replacement handler already existed, and the logs showed no traffic to the old one. Before the delete, my migration-parity auditor checked behavior instead of traffic, and it flagged the delete. The replacement turned out to be a stub. It handled only two event types, and only by logging them. Three payment events fell through and were silently dropped. These were invoice paid, payment failed, and subscription deleted. If the old handler had been deleted, invoices would have stayed pending forever and cancellations would never have closed bookings. Stripe would have kept reporting success the whole time. The lesson for me was that zero traffic is not proof that something is dead.

## What did not work

I also tried running builder agents in parallel, each in its own git worktree. The idea was to ship more work at once without losing the guarantees. It worked for read-only reviews and docs. It failed for real code. The worktree agents could not compile or run tests, and three waves in a row stalled. I now route all code changes through the main session, where the gate can run. A control system is only as good as the checks it can actually run.

## Open research questions

My setup works for my project. I do not know yet if it works in general. These are the questions I want to study.

1. **Can LLM monitors catch quiet sabotage?** The stub and the database bug both looked fine and passed checks. I want to red-team my own monitors with realistic sabotage-style changes like these. Then I want to measure how many each monitor catches at a fixed audit budget, and what it costs in false alarms.
2. **What should earn an agent more autonomy?** I picked 5 clean changes by judgment. Does a clean streak really predict future safety? Could an agent game the streak by behaving well only when it thinks it is being watched? If so, the ledger rewards the wrong thing. Trust earned on low-risk work unlocks high-risk work. That is exactly the gap a scheming model would use.
3. **How often do self-reports differ from reality?** I want to measure how often an agent's claim about its own work does not match an independent check.
4. **Can interpretability see it from the inside?** Can a model's internal signals show when it is about to write code that looks right but is not?

I built this system because I needed it. Now I want to turn it into research that works for any model and any codebase, including models far more capable than the ones I use today.
