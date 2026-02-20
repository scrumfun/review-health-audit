# We Analyzed 5,212 Review Comments and Found a Metric Nobody Had Formalized

**TL;DR:** I took a year of GitLab data (1,140 merge requests, 4-person Go backend team), ran it through Google/Meta/SmartBear frameworks — and discovered a pattern that no published research had formalized: **Effort Inversion Ratio**. When a late reviewer leaves *more* comments than the first reviewer, it's a direct predictor of rework churn.

---

## The Problem: LOC, Commits, and Other Useless KPIs

Every engineering lead eventually asks: "how do I measure what's happening in my team?"

The standard answers — lines of code, commit count, story points — don't work. McKinsey tried to propose an individual developer productivity framework in 2023. The industry response was unanimous:

> **Kent Beck** (creator of Extreme Programming): "absurd & naive"
> **Dan North** (creator of BDD): criticized them for ignoring a decade of Dr. Nicole Forsgren's research

**Goodhart's Law:** "When a measure becomes a target, it ceases to be a good measure." Lines of code → bloated code. Story points → estimate inflation. Coverage % → tests on getters.

I needed metrics that:
1. Measure *behavior*, not volume
2. Can't be gamed
3. Directly connect to outcomes (delivery speed, code quality)

## What I Did

Collected all MRs from GitLab API over 12 months. 1,140 merge requests, 5,212 human comments, 4-person team, Go microservices.

Applied four research frameworks:

| Framework | Source | What It Measures |
|-----------|--------|-----------------|
| **DORA** | Google, 39,000+ engineers | Delivery speed and stability |
| **SPACE** | GitHub + Microsoft | 5 dimensions of productivity |
| **Project Aristotle** | Google, 180 teams, 2 years | What makes teams effective |
| **Code Review Health** | Google + SmartBear/Cisco + Meta | Review speed and quality |

And found things that aren't in any of them.

## Finding 1: Effort Inversion Ratio (EIR)

Simple idea: **the first reviewer does the heavy lifting** — finds bugs, asks questions, suggests improvements. Second/third reviewers join an already-worked MR and typically leave fewer comments. This is confirmed by SmartBear/Cisco ("diminishing returns of sequential reviewers") and intuitively obvious.

Formula:
```
EIR = avg comments as last reviewer / avg comments as first reviewer
```

Expected value: **< 1.0** (first reviewer → more work).

What I found in our data:

| Reviewer | As First | As Last | EIR |
|----------|----------|---------|-----|
| Alice | 5.3 comments/MR | 4.2 comments/MR | **0.79** — normal |
| Bob | 5.3 | 3.7 | **0.70** — normal |
| Charlie | 3.8 | **6.6** | **1.74** — inverted |

Charlie leaves **1.7x more** comments when joining last. He's not "validating" what's already been found — he *generates new issues* on top of an already-reviewed MR. The author has already aligned with the first reviewer, and then a second round begins.

**Why it matters:** EIR > 1.0 directly correlates with rework churn — reworking already-approved code. This extends cycle time and demotivates authors.

### Who takes first review — and who arrives late

| Reviewer | First reviewer (%) | Last reviewer (%) | Median delay when late | EIR |
|----------|-------------------|-------------------|----------------------|-----|
| **Bob** | 43/115 (37%) | 46/115 (40%) | 20.1h | 0.70 |
| **Alice** | 40/115 (35%) | 23/115 (20%) | 4.2h | 0.79 |
| **Charlie** | 29/115 (25%) | 44/115 (38%) | 47.0h | **1.74** |
| Dave | 3/115 (3%) | 2/115 (2%) | 73.4h | — |

> Alice takes first review in 35% of cases but is last only 20% — she joins fast. Charlie is the mirror: first only 25%, last 38%, with a median delay of **47 hours** (~2 business days). And when he finally arrives — he leaves *more* comments, not fewer.

### Cross-matrix: who waits how long after whom (median hours)

| Late ↓ \ First → | Alice | Bob | Charlie |
|---|---|---|---|
| **Alice** | — | 3.8h | 4.5h |
| **Bob** | 20.6h | — | 19.2h |
| **Charlie** | 47.7h | 43.1h | — |

> Each row is stable — Charlie takes ~2 days regardless of who reviewed first. Alice responds within 3-5 hours. This means it's an **individual rhythm**, not a reaction to context.

**Why it's better than LOC:**
- Measures *process behavior*, not output volume
- Can't be gamed — you can't "fix" the metric without genuinely changing your review approach
- Actionable — if EIR > 1.0, the solution is clear: assign this person as the **first** reviewer (not second)

**No analogues in published research.** I checked: SmartBear described diminishing returns qualitatively, Meta studied the bystander effect (delay), Google measured latency. Nobody formalized *effort inversion by position*.

## Finding 2: Self-Merge ≠ Rubber Stamp

The standard metric is "% of MRs without review." Ours was 56%. Sounds equally bad for everyone. But dig deeper:

| Pattern | Alice | Charlie |
|---------|-------|---------|
| MRs without review comments | 77% | 74% |
| Assigns a reviewer | **95%** | **0%** |
| True self-merge (no review AND no reviewer) | **4%** | **74%** |

**Different root causes:**
- **Alice** assigns a reviewer, but they don't comment → **rubber stamp** (process exists but is passive)
- **Charlie** never assigns a reviewer → **self-merge** (process doesn't exist)

These are different problems with different solutions. Rubber stamp → need a culture of mandatory commenting. Self-merge → need mandatory reviewer assignment.

Most dashboards collapse both into "% MRs without review" and lose the signal.

### The full picture: process vs no process

| Person | True self-merge (no review + no reviewer) | Assigns reviewer | Cycle time P50 |
|--------|-------------------------------------------|-----------------|----------------|
| **Alice** | 22/503 (**4%**) | **95%** of MRs | 20.7h |
| **Bob** | 133/203 (**66%**) | **0%** of MRs | 21.8h |
| **Charlie** | 203/276 (**74%**) | **0%** of MRs | 5.4h |
| **Dave** | 81/109 (**74%**) | **2%** of MRs | 126.4h |

> This is a **team-wide process problem**, not one person's issue. Charlie, Bob, and Dave *never* assign a formal reviewer. The only person using the review process is Alice (95%). But even when she assigns a reviewer, they don't comment 77% of the time — the reviewer "sees" the MR but doesn't actually review it.

## Finding 3: "Hot Potato" — Hidden Cost Transfer

Charlie merges his MRs in 5.4 hours (P50). Alice — in 20.7 hours. Seems like Charlie is more efficient.

But: Charlie merges without review, and the team cleans up afterward. Real example — a library migration: 23 MRs across all services in 4 days, 0 formal reviewers, merged in one day. Then Alice fixes issues in services where Charlie applied the change without deep knowledge of each service's context.

**Cycle time P50 = 5.4h** doesn't account for cleanup cost to the team. Fast merge saves *the author's* time but creates work-in-progress for *everyone else*. In industry terms, this is **downstream cost transfer** — the author's metrics look great, but the cost is hidden in other people's metrics.

### Review turnaround: who responds fastest as a reviewer

| Reviewer | P50 | P90 | Benchmark |
|----------|-----|-----|-----------|
| Alice | **18.4h** | 93.3h | ⚠️ Best on the team, but still > 12h (Good tier) |
| Bob | 26.1h | 163.5h | 🔴 Needs Work |
| Charlie | **66.7h** | 160.8h | 🔴 Critical (2.8 business days) |
| Dave | 106.6h | 143.8h | — (small sample, N=4) |

> Charlie's asymmetry: merges his own code in 5.4h but takes **66.7h** to review others' — a **12x gap**. He optimizes for his own throughput at the team's expense.

### Contribution by period (with expected norms)

**Before Dave joined (10 months, 3 people):**

| Person | MRs | Share | Norm (3 people) | Deviation |
|--------|-----|-------|-----------------|-----------|
| Alice | 459 | **52.3%** | 33.3% | **+19 pp** (×1.6) |
| Charlie | 244 | 27.8% | 33.3% | −5.5 pp |
| Bob | 175 | 19.9% | 33.3% | −13.4 pp |

**After Dave joined (~2.7 months, 4 people):**

| Person | MRs | Share | Norm (4 people) | Deviation |
|--------|-----|-------|-----------------|-----------|
| Dave | 111 | **42.4%** | 25% | +17.4 pp (newcomer energy) |
| Alice | 62 | 23.7% | 25% | −1.3 pp (≈ norm) |
| Charlie | 49 | 18.7% | 25% | −6.3 pp |
| Bob | 40 | 15.3% | 25% | −9.7 pp |

> In period 1, Alice carried **1.6x the expected load** while others were below norm. After Dave joined, her share normalized — but Charlie and Bob remained below norm in *both* periods. The newcomer didn't cause the imbalance — it was already there.

### Bus factor: single points of failure

| Risk | Subsystem | Top contributor | Share | Bus Factor |
|------|-----------|----------------|-------|------------|
| 🔴 | Library A | Alice | 74% | 1 |
| 🔴 | Service D | Alice | 76% | 1 |
| 🔴 | Service E | Charlie | 94% | 1 |
| 🔴 | Library B | Bob | 85% | 1 |
| 🔴 | Library C | Alice | 89% | 1 |
| ⚠️ | Service A | Alice | 60% | 2 |
| ⚠️ | Service B | Alice | 57% | 2 |
| ⚠️ | Service C | Charlie | 56% | 2 |

> Alice is the sole significant contributor in 5 subsystems and dominant (>50%) in 5 more. Her vacation, illness, or departure blocks development of **10 out of 18 key components**.

## Finding 4: Psychological Safety Measured Through Data

Google's Project Aristotle found that psychological safety is the #1 factor in team effectiveness. Usually it's measured with surveys. But code review data contains objective proxies:

| Indicator | Value | Signal |
|-----------|-------|--------|
| Equal participation in discussions | A:1475, C:1037, B:806, D:90 | ⚠️ Skewed, but explainable by workload |
| Defensive behavior in reviews | 8/8 instances → Alice, 0 → others | 🔴 One person can't safely give feedback |
| Personal attacks in reviews | 3+ instances (C→A), 0 in reverse | 🔴 Asymmetry violates safety |
| Confrontation rate by addressee | 11% (→A) vs 4.3% (→B/D) — 2.5× | 🔴 Differentiated treatment |
| Reaction to same question (same MR) | To Bob → humor; To Alice → authority appeal + deadline pressure | 🔴 Contrast test confirms asymmetry |
| Newcomer's ability to give feedback | Dave reviews leads' code by week 6 | ✅ Good signal |

This isn't a survey — it's behavioral data. A 2.5x confrontation asymmetry toward one person is a measurable signal that something is off.

## The Dashboard

| Metric | Our Value | Benchmark (Good) | Status |
|--------|-----------|-------------------|--------|
| Review Turnaround P50 | 26.8h | < 12h | Red |
| Rubber Stamp Rate | 56% | < 20% | Red |
| Self-Merge (one person) | 74% | < 15% | Red |
| Knowledge Distribution (Gini) | 0.052 | < 0.3 | Green |
| Bus Factor (min across subsystems) | 1 (5 subsystems) | >= 2 | Red |
| EIR (max) | 1.74 | < 1.1 | Red |
| Psych Safety proxy | 3/10 | — | Red |

5 out of 7 metrics in the red zone. The team considered itself "working fine" — because **nobody was measuring**.

## How to Replicate

1. **Collect data** — GitLab/GitHub API, all MRs for 6-12 months with comments
2. **Install the skill** — `cp -r review-health-audit ~/.claude/skills/`
3. **Run** — `/review-health-audit path/to/data.json`
4. **Read** — report with traffic-light statuses and recommendations

Or replicate manually — the full methodology is described in [SKILL.md](SKILL.md).

## What's Next

EIR is a simple metric that any team can calculate. If you have data from GitHub/GitLab — try it. I'm curious what EIR values you'll see in your teams.

My hypothesis: EIR > 1.2 is a process health marker for any team, regardless of size or tech stack.

---

*Tools: Claude Code + Python + GitLab API. Data: 1,140 MRs, 5,212 comments, 12 months. Frameworks: DORA, SPACE, Project Aristotle, Google/SmartBear/Meta.*

*Skill and methodology: [github.com/scrumfun/review-health-audit](https://github.com/scrumfun/review-health-audit)*
