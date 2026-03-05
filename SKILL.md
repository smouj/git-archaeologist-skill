---
name: git-archaeologist
description: Analyzes git history to extract code rationale and generate historical documentation
version: 1.0.0
author: OpenClaw Team
tags:
  - git
  - archaeology
  - documentation
  - history
  - code-analysis
maintainer: SMOUJBOT
category: development
type: research
dependencies:
  - git>=2.30
  - python3
  - gitpython>=3.1
  - jinja2
  - pyyaml
  - graphviz (optional)
---

# Git Archaeologist

## Purpose

Real-world use cases:
- Onboarding engineers to legacy codebases by extracting decision-making context from commit messages and pull request discussions
- Compliance audits: trace regulatory changes through code evolution and identify who approved specific logic changes
- Bug triage: determine when and why a problematic function was introduced by analyzing blame + historical context
- Architecture review: map feature evolution over time to identify technical debt accumulation patterns
- Documentation gap analysis: identify code sections with sparse historical records needing manual documentation

## Scope

Concrete commands this skill provides:
- `git-archaeology dig <commit_range> --output=markdown`
- `git-archaeology blame --enhanced <file_path> --since="2024-01-01"`
- `git-archaeology timeline --feature="auth" --include-merges`
- `git-archaeology authorship <directory> --heatmap`
- `git-archaeology rationale <commit_hash> --extract-prs`
- `git-archaeology report --format=html --template=compliance`

Parameters are concrete: `--since`, `--until`, `--author`, `--regex`, `--max-depth`, `--prune-tests`

## Work Process

1. Repository preparation:
   - Ensure clean working directory: `git status --porcelain | grep -q . && exit 1`
   - Fetch all remotes: `git fetch --all --prune`
   - Identify analysis scope: single file, directory, or commit range

2. Extract raw history:
   - Use `git log --all --pretty=format:'%H|%an|%ae|%ad|%s' --date=iso --no-merges` for commits
   - Augment with `git show --stat --name-only <commit>` for file changes per commit
   - Pull PR metadata if GitHub CLI available: `gh pr list --json number,title,body,author`

3. Context enrichment:
   - Correlate commits to PRs using conventional commit patterns (feat:, fix:, chore:)
   - Run `git blame -C -C <file>` to detect code movement across files
   - Extract code review comments via GitHub API when PRs found

4. Pattern detection:
   - Identify "orphaned code" (no commits in >6 months)
   - Detect large single-commit additions (>500 lines) for review
   - Flag commits with vague messages ("fix bug", "update") for follow-up

5. Documentation generation:
   - Render Jinja2 templates with discovered metadata
   - Include code snippets from before/after states
   - Add timeline visualizations if graphviz present

6. Validation:
   - Verify all referenced commits still exist in repo
   - Check file paths haven't been deleted
   - Cross-reference author email consistency

## Golden Rules

1. Never rewrite history. All reads are read-only; no `git rebase` or `filter-branch`.
2. Preserve original commit messages verbatim; do not summarize or paraphrase.
3. Always include commit hashes in outputs for traceability.
4. Handle large repos gracefully: paginate results, don't load entire log into memory.
5. When PR linking fails, log warning but continue analysis.
6. Respect .gitignore and exclude binary files from detailed analysis.
7. For private repos, never cache auth tokens; use session-based GitHub CLI auth.
8. Report uncertainties: "Cannot determine rationale, commit message lacks context" is valid output.
9. Limit automated API calls to GitHub to rate limits (max 5000/hour).
10. Always verify repository root before executing; prevent escape to parent directories.

## Examples

Example 1: Basic archaeology on a feature
```
Input: git-archaeology dig --since="2024-01-01" --output=markdown
Output: archaeology-report-2024-01-01-to-2024-12-31.md containing:
- 234 commits analyzed
- 47 PRs linked
- Top contributors: alice@example.com (32%), bob@example.com (28%)
- Orphaned files: src/legacy/utils.py (last touched 2023-05-12)
- Largest change: commit abc123 added 847 lines in src/auth/
```

Example 2: Enhanced blame with movement tracking
```
Input: git-archaeology blame --enhanced src/auth/login.py --since="2024-06-01"
Output:
File: src/auth/login.py (85% original, 15% moved/copied)
Line 45-67: moved from src/auth/old_session.py in commit def456 (2024-07-21) by bob@example.com
Rationale: PR #342 "Refactor session handling" - "Moving to stateless JWT tokens"
```

Example 3: Timeline generation for compliance
```
Input: git-archaeology timeline --feature="payment" --include-merges --prune-tests
Output: payment-evolution.svg timeline showing:
- Milestone: commit 789xyz (2024-03-15) added Stripe integration
- Regression: commit 234abc (2024-05-02) reverted to PayPal due to PCI compliance
- Feature flag: commit 567def (2024-08-10) introduced dual-vendor support
```

Example 4: Authorship heatmap
```
Input: git-archaeology authorship src/components/ --heatmap
Output: authorship-heatmap.csv
Component,authors,last_touch,complexity_owners
Button.tsx,alice (90%),2024-12-01,solo
Modal.tsx,alice (60%),bob (40%),2024-11-15,shared
```

Example 5: Rationale extraction for specific commit
```
Input: git-archaeology rationale abc123 --extract-prs
Output:
Commit: abc123 "Fix login race condition"
PR: #456 titled "Fix concurrent login edge case"
Author: alice@example.com
Reviewers: bob@example.com, charlie@example.com
Discussion excerpt: "This lock-free approach uses atomic CAS operation; benchmark shows 15% throughput increase under load."
Code diff: (+12 -8)
```

## Rollback Commands

Since this skill is read-only, rollback is simply removal of generated artifacts:

- `rm -f archaeology-report-*.md archaeology-report-*.html *.csv *.svg`
- `git restore --staged .` (if any temporary files were accidentally staged)
- Kill any lingering processes: `pkill -f git-archaeology` (if daemonized)

No git repository state is ever modified; rollback is cleanup-only.

## Environment Variables

- `GIT_ARCHAEOLOGY_GITHUB_TOKEN`: optional, for PR metadata (falls back to gh cli)
- `GIT_ARCHAEOLOGY_MAX_COMMITS`: default 10000, max history depth
- `GIT_ARCHAEOLOGY_TEMPLATE_DIR`: path to custom Jinja2 templates
- `GIT_ARCHAEOLOGY_VERBOSE`: set to 1 for debug logging

## Dependencies & Requirements

System:
- Python 3.8+
- Git 2.30+ (for `git log --date=iso` and `--name-only`)

Python packages (install via `pip install git-archaeology`):
- GitPython>=3.1.30
- Jinja2>=3.0
- PyYAML>=5.4
- click (CLI wrapper)
- python-dateutil (date parsing)
- tabulate (table output)

Optional:
- github3.py or gh (GitHub CLI) for PR integration
- graphviz for timeline SVGs

## Verification Steps

After skill installation:
1. Run `git-archaeology --version` → should print 1.0.0
2. In a test repo: `git init && touch foo && git add . && git commit -m "initial"` then `git-archaeology dig` → should produce empty report with 1 commit
3. Simulate multi-commit history: add 3 commits, run `git-archaeology blame --enhanced foo` → should detect no movement and report 100% original
4. Test PR linking (if GitHub token set): create PR with conventional commit, run `git-archaeology dig --since="1 day ago"` → should include PR number in output

## Troubleshooting

Issue: "fatal: not a git repository"
- Solution: Ensure current directory is inside a git repo. Use `git rev-parse --show-toplevel` to verify.

Issue: "GitPython import failed"
- Solution: Install with `pip install GitPython`. Verify Python version: `python3 --version`.

Issue: "rate limit exceeded" from GitHub API
- Solution: Set `GIT_ARCHAEOLOGY_GITHUB_TOKEN` with a personal access token (repo:read scope). Or disable PR linking with `--no-prs`.

Issue: "MemoryError on large repo"
- Solution: Set `GIT_ARCHAEOLOGY_MAX_COMMITS=5000` to limit depth. Use `--since` to narrow range.

Issue: Timeline SVG generation fails
- Solution: Install graphviz: `apt-get install graphviz` or `brew install graphviz`. Ensure `dot` executable in PATH.

Issue: Blame shows no movement detection
- Solution: Verify `git blame -C -C` works manually. Ensure code actually moved; copy detection requires identical chunks.

Issue: Dates in report are UTC, need local timezone
- Solution: Set `TZ=America/Los_Angeles` (or desired tz) before running. Git log dates are converted using system timezone.

Issue: UTF-8 encoding errors in commit messages
- Solution: Ensure locale is UTF-8: `export LC_ALL=en_US.UTF-8`. Git stores raw bytes; Python defaults to locale encoding.
```