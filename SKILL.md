---
name: review-health-audit
description: Audit your team's code review health using GitLab/GitHub MR data. Calculates industry metrics (DORA, SPACE, Project Aristotle proxies) plus novel metrics like Effort Inversion Ratio.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch, Task
---

# Code Review Health Audit

Analyze your team's code review process against industry benchmarks (Google, SmartBear/Cisco, Meta, Project Aristotle). Produces a health report with actionable recommendations.

## What It Measures

### Industry Metrics
| Metric | Source | What It Shows |
|--------|--------|---------------|
| Review Turnaround P50/P90 | Google ICSE-SEIP 2018 | How fast reviews happen |
| Rubber Stamp Rate | SmartBear/Cisco | % of MRs merged without review |
| Self-Merge Rate | Meta 2023 | % of MRs merged without reviewer assigned |
| Knowledge Distribution (Gini) | Pluralsight/GitPrime | How evenly reviews are spread |
| Bus Factor per subsystem | ICSE 2022 | Single-point-of-failure risk |
| Cycle Time (MR created → merged) | DORA | End-to-end delivery speed |
| Formal vs Actual Review Gap | Original | GitLab "Assign reviewer" ≠ actual review |

### Novel Metrics (original research)
| Metric | Formula | What It Shows |
|--------|---------|---------------|
| **Effort Inversion Ratio (EIR)** | `avg_comments_as_last_reviewer / avg_comments_as_first_reviewer` | When EIR > 1.0, late reviewers generate MORE comments than first reviewers — causes rework churn |
| **Review Join Rhythm** | Median delay when joining as non-first reviewer | Individual review tempo, independent of context |

### Psychological Safety Proxies (Project Aristotle)
- Tone asymmetry by addressee (confrontation rate WHO → WHOM)
- Defensive behavior frequency + target distribution
- Contrast test: same question from different people → different response?

## Input

The skill needs MR data in JSON format. You can collect it with the GitLab/GitHub API.

### Required data structure
```json
{
  "team": {"207": "alice", "143": "bob", "146": "carol"},
  "projects": {
    "project-name": {
      "id": 123,
      "path": "group/project-name",
      "merge_requests": [
        {
          "iid": 1,
          "title": "...",
          "source_branch": "...",
          "author_id": 207,
          "state": "merged",
          "created_at": "2025-01-01T10:00:00Z",
          "merged_at": "2025-01-02T14:00:00Z",
          "reviewers": [{"id": 143}],
          "notes": [
            {
              "author_id": 143,
              "body": "comment text",
              "system": false,
              "created_at": "2025-01-01T12:00:00Z"
            }
          ]
        }
      ]
    }
  }
}
```

### Collecting data from GitLab
```bash
# Set your GitLab instance and token
export GITLAB_URL="https://gitlab.example.com"
export GITLAB_TOKEN="your-token"

# For each project, fetch MRs with notes:
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_URL/api/v4/projects/GROUP%2FPROJECT/merge_requests?state=all&per_page=100&page=1" \
  | jq '.'

# For each MR, fetch notes:
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$GITLAB_URL/api/v4/projects/GROUP%2FPROJECT/merge_requests/IID/notes?per_page=100" \
  | jq '.'
```

### Collecting data from GitHub
```bash
gh api repos/OWNER/REPO/pulls --paginate -q '.[] | {number, title, user, created_at, merged_at, state}'
# For each PR:
gh api repos/OWNER/REPO/pulls/NUMBER/reviews
gh api repos/OWNER/REPO/pulls/NUMBER/comments
```

## Usage

```
/review-health-audit <path-to-data.json> [team-size] [period-split-date]
```

- `path-to-data.json` — path to collected MR data
- `team-size` — number of team members (for norm calculation)
- `period-split-date` — optional date to split before/after analysis (e.g. new team member joined)

## Process

### Step 1: Load and validate data
Read the JSON file, verify structure, count MRs and comments.

### Step 2: Calculate metrics
For each metric in the table above, compute the value and compare against benchmarks.

### Step 3: Industry benchmarks

| Metric | Elite | Good | Needs Work |
|--------|-------|------|------------|
| Turnaround P50 | < 4h | < 12h | > 24h |
| Turnaround P90 | < 12h | < 24h | > 48h |
| Rubber stamp rate | < 10% | < 20% | > 30% |
| Self-merge rate | < 5% | < 15% | > 30% |
| Knowledge Gini | < 0.2 | < 0.3 | > 0.5 |
| Bus factor (per subsystem) | >= 3 | >= 2 | = 1 |
| EIR (per person) | 0.6-0.9 | 0.9-1.1 | > 1.2 |

### Step 4: Generate report
Output a structured markdown report with:
1. **Dashboard** — all metrics with traffic-light status
2. **Per-person profiles** — turnaround, EIR, first/last reviewer frequency
3. **Risk areas** — bus factor, rubber stamp hotspots
4. **Recommendations** — prioritized by impact, with specific actions

### Step 5: Key rules
- **Context > numbers** — never present a metric without explaining WHY
- **WHO → WHOM** — every interaction metric must specify direction
- **Goodhart's Law** — metrics are diagnostic tools, NOT individual KPIs
- **Median over mean** — for timing data, median is more representative (less skewed by outliers)
- **Define every metric** — what counts as "confrontational", "defensive", etc.

## Output format

The report should be saved as `review-health-report.md` in the current directory.

## References

- DORA Metrics: https://dora.dev/guides/dora-metrics-four-keys/
- SPACE Framework: https://queue.acm.org/detail.cfm?id=3454124
- Project Aristotle: https://rework.withgoogle.com/guides/understanding-team-effectiveness
- Google Code Review: https://research.google/pubs/modern-code-review-a-case-study-at-google/
- SmartBear/Cisco: https://smartbear.com/learn/code-review/best-practices-for-peer-code-review/
- Meta Bystander Effect: https://engineering.fb.com/2022/11/16/culture/meta-code-review-time-improving/
