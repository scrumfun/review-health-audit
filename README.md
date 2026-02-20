# Review Health Audit

**A Claude Code skill that audits your team's code review process against industry benchmarks.**

Calculates metrics from DORA, SPACE, Project Aristotle, and Google/SmartBear/Meta research — plus **two original metrics** we couldn't find in any published research.

## What You Get

A health report with traffic-light status for every metric:

| Category | Metric | Source |
|----------|--------|--------|
| Speed | Review Turnaround P50/P90 | Google ICSE-SEIP 2018 |
| Coverage | Rubber Stamp Rate | SmartBear/Cisco |
| Coverage | Self-Merge Rate (no reviewer assigned) | Meta 2023 |
| Knowledge | Knowledge Distribution (Gini coefficient) | Pluralsight/GitPrime |
| Knowledge | Bus Factor per subsystem | ICSE 2022 |
| Delivery | Cycle Time (created → merged) | DORA |
| Process | Formal vs Actual Review Gap | Original |
| **Novel** | **Effort Inversion Ratio (EIR)** | **Original** |
| **Novel** | **Review Join Rhythm** | **Original** |
| Safety | Psychological Safety proxies | Project Aristotle |

## Novel Metrics

### Effort Inversion Ratio (EIR)

```
EIR = avg_comments_as_last_reviewer / avg_comments_as_first_reviewer
```

**Why it matters:** The first reviewer does the "heavy lifting" — finds the most issues. Late reviewers should add less (the field is already covered). When someone's EIR > 1.0, they generate *more* comments when joining last — which means they're adding new issues on top of already-reviewed code, causing **rework churn**.

| EIR Value | Meaning |
|-----------|---------|
| 0.6–0.9 | Normal: first reviewer works harder |
| ~1.0 | Equal effort regardless of position |
| > 1.2 | **Inverted**: late reviewer adds more → rework churn |

**No analogues in published research.** SmartBear described "diminishing returns of sequential reviewers" qualitatively. Meta studied the bystander effect (delay, not effort). Google measured latency, not depth by position. Nobody formalized the *inversion*.

### Review Join Rhythm

Median delay when someone joins a review as non-first reviewer. Shows **individual tempo** independent of who reviewed first. Stable across all pairings = it's a personal characteristic, not a contextual one.

## Why This Is Better Than LOC

| Traditional KPI | Problem | Our Approach |
|----------------|---------|--------------|
| Lines of Code | Penalizes refactoring, rewards verbosity | Measures *behavior*, not volume |
| Commit Count | Rewards tiny meaningless commits | Measures *process health*, not activity |
| Story Points | Inflation makes velocity meaningless | Measures *outcomes*, not estimates |
| Code Coverage % | Tests on getters, ignores critical paths | Measures *review quality*, not test quantity |

**Goodhart's Law:** "When a measure becomes a target, it ceases to be a good measure."

All metrics in this skill are **diagnostic tools** for team health, not KPIs for ranking individuals. The question is always "are we getting better?" — not "who is the best?"

## Quick Start

### 1. Install the skill

```bash
# Copy to your Claude Code skills directory
cp -r review-health-audit ~/.claude/skills/
# Or for project-level:
cp -r review-health-audit .claude/skills/
```

### 2. Collect your data

**GitLab:**
```bash
export GITLAB_URL="https://gitlab.example.com"
export GITLAB_TOKEN="your-read-api-token"

# Fetch MRs (paginate as needed)
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_URL/api/v4/projects/GROUP%2FPROJECT/merge_requests?state=all&per_page=100"

# Fetch notes for each MR
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_URL/api/v4/projects/GROUP%2FPROJECT/merge_requests/IID/notes?per_page=100"
```

**GitHub:**
```bash
gh api repos/OWNER/REPO/pulls --paginate
gh api repos/OWNER/REPO/pulls/NUMBER/reviews
gh api repos/OWNER/REPO/pulls/NUMBER/comments
```

Format your data as JSON (see [SKILL.md](review-health-audit/SKILL.md) for the expected structure).

### 3. Run the audit

```
/review-health-audit path/to/data.json
```

### 4. Read the report

The skill generates `review-health-report.md` with:
- Dashboard (all metrics + traffic lights)
- Per-person profiles
- Risk areas (bus factor, rubber stamp hotspots)
- Prioritized recommendations

## Benchmarks Reference

| Metric | Elite | Good | Needs Work |
|--------|-------|------|------------|
| Review Turnaround P50 | < 4h | < 12h | > 24h |
| Review Turnaround P90 | < 12h | < 24h | > 48h |
| Rubber Stamp Rate | < 10% | < 20% | > 30% |
| Self-Merge Rate | < 5% | < 15% | > 30% |
| Knowledge Gini | < 0.2 | < 0.3 | > 0.5 |
| Bus Factor (per subsystem) | >= 3 | >= 2 | = 1 |
| EIR (per person) | 0.6–0.9 | 0.9–1.1 | > 1.2 |

Sources: [DORA](https://dora.dev), [SPACE](https://queue.acm.org/detail.cfm?id=3454124), [Project Aristotle](https://rework.withgoogle.com/guides/understanding-team-effectiveness), [Google Code Review](https://research.google/pubs/modern-code-review-a-case-study-at-google/), [SmartBear/Cisco](https://smartbear.com/learn/code-review/best-practices-for-peer-code-review/), [Meta](https://engineering.fb.com/2022/11/16/culture/meta-code-review-time-improving/)

## Background

This skill was born from analyzing 1,140 merge requests and 5,212 comments across a 4-person backend team over 12 months. The **Effort Inversion Ratio** was discovered when we noticed that one team member consistently left *more* comments when joining reviews last — the opposite of what every published study predicted.

The full analysis also uncovered:
- **56% of MRs merged with zero review comments** (vs <10% industry Elite benchmark)
- **Self-merge vs rubber stamp** are different problems with different root causes
- **"Hot potato" anti-pattern**: fast self-merge (P50=5.4h) creates hidden cleanup cost for the team
- **Psychological safety measured through code review data** — not surveys, but actual behavioral patterns

Read the full story: [article.md](article.md)

## License

MIT
