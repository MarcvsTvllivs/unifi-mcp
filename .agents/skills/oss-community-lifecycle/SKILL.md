---
name: myco:oss-community-lifecycle
description: |
  Apply this skill when managing external community engagement in unifi-mcp:
  setting up GitHub health files and issue routing infrastructure, designing
  or updating bug report templates, triaging incoming bug reports with an
  evidence-first protocol, scoping follow-up comments in open issues, and
  unblocking contributor PRs stalled on trivial CI failures. Activates for
  any task touching .github/ISSUE_TEMPLATE/, .github/config.yml,
  CONTRIBUTING.md, or SUPPORT.md, or when reviewing external contributor
  PRs — even if the user doesn't explicitly ask for community workflow
  guidance.
managed_by: myco
user-invocable: true
allowed-tools: Read, Edit, Write, Bash, Grep, Glob
---

# OSS Community Lifecycle and Contributor Management

This skill covers the full lifecycle of managing external community engagement in unifi-mcp: from the first time a user files an issue through PR merge. Because unifi-mcp is a hardware-coupled MCP server (real UniFi controllers, real credentials, firmware that varies by hardware SKU), community bugs require more upfront context than typical software projects — and the GitHub infrastructure should be designed to collect that context automatically.

This skill is complementary to `myco:community-pr-review`, which covers the PR quality gate checklist (f-string logger ban, ruff enforcement, validator registry registration). This skill covers the upstream community infrastructure and triage lifecycle; `community-pr-review` covers what happens once a PR is technically ready for review.

## Prerequisites

- GitHub repository admin access (to create or modify `.github/` files)
- Familiarity with GitHub Issue Forms (YAML-based `.github/ISSUE_TEMPLATE/*.yml` files)
- For triage: access to a live UniFi controller with real credentials, or the ability to request a reproduction from the reporter
- For contributor unblocking: a maintainer GitHub token with fork-push rights, and confirmation that the contributor enabled "Allow edits from maintainers" when opening their PR

## Procedure A: GitHub Health File and Issue Routing Setup

Set up the `.github/` infrastructure that makes triage scalable. Do this once when the project goes public, and revisit whenever a new issue category emerges.

### File structure

```
.github/
  ISSUE_TEMPLATE/
    bug_report.yml          # Structured bug form (see Procedure B)
    feature-request.yml     # Optional: structured feature form
    config.yml              # Routing rules
  CONTRIBUTING.md           # Contribution guidelines
  SUPPORT.md                # Support channels
```

### config.yml routing

Route sensitive and support traffic away from Issues before it arrives:

```yaml
# .github/config.yml
blank_issues_enabled: false
contact_links:
  - name: Security Vulnerability
    url: https://github.com/ORG/REPO/security/advisories/new
    about: Report a security vulnerability via GitHub Security Advisories
  - name: Usage Question / Support
    url: https://github.com/ORG/REPO/discussions
    about: Ask questions and get help in GitHub Discussions
```

Setting `blank_issues_enabled: false` forces all issues through the form templates, preventing unstructured "it doesn't work" filings. Security reports go to Advisories (private by default); support questions go to Discussions so the issue tracker stays actionable.

### SUPPORT.md

Keep it short. Point to Discussions for questions, to the issue tracker for confirmed bugs, and to Security Advisories for vulnerabilities. Do not over-promise response times.

### CONTRIBUTING.md

Cover: fork/branch strategy, how to run local tests, CI gate expectations (ruff, mypy), PR description requirements, and the evidence-first policy for bug fixes (see Procedure C). The CONTRIBUTING.md is where the evidence-first doctrine is published so contributors know what to expect before filing.

## Procedure B: Bug Report Template Design

The bug report template is the primary defense against 3-comment follow-up cycles. Because unifi-mcp behavior varies by controller firmware, hardware SKU, and install method, the template must collect all triage-critical context at first filing.

### Required template file

`.github/ISSUE_TEMPLATE/bug_report.yml`

### Required fields

These fields are required to start triage. Do not make them optional:

| Field | Type | Why required |
|-------|------|--------------| 
| **Controller hardware** | Dropdown | API behavior varies by hardware family (UDM Pro vs. Cloud Key vs. self-hosted) |
| **UniFi OS version** | Text input | Firmware version determines which API fields are present |
| **Install method** | Text input | Determines whether aiounifi version is pinned (uvx/pip/Docker) vs. flexible (source) |
| **Client OS** | Text input | Needed for install-method-specific issues |

**Controller hardware dropdown options** (exact list from `.github/ISSUE_TEMPLATE/bug_report.yml`):
```
UDM Pro
UDM Pro Max
UDM SE
UDM (base)
UDR / UDR7
UCG Max / UCG Ultra / UCG Fiber
Cloud Key Gen2 / Gen2+
Self-hosted (Linux)
Self-hosted (Docker)
UniFi-hosted (Site Manager / Cloud Console)
Other
N/A — not an MCP server bug
```

**Install method** is a free-text input field (not a dropdown). Use a placeholder to hint at accepted values:

```yaml
- type: input
  id: install_method
  attributes:
    label: Install method
    placeholder: "uvx, pip, Docker image tag, plugin marketplace, source"
  validations:
    required: true
```

### Per-application version fields

Label these as "required if reporting a bug in this app." GitHub Issue Forms don't support conditional `required`, so enforce via description text:

```yaml
- type: input
  id: network_app
  attributes:
    label: Network application version
    description: Required if reporting a Network MCP bug. Settings → System → Updates → "Network".
    placeholder: "e.g. 9.0.108"
```

Apply the same pattern for `protect_app` and `access_app`. This pattern — required by convention, not schema — is the best GitHub Forms can do for conditional required fields.

### AI-agent context fields (optional, four separate fields)

Because unifi-mcp is used by AI agents, many reported bugs are actually agent misuse rather than server bugs. Collecting agent context upfront separates the two. Use four separate fields (matching the current `bug_report.yml` layout):

```yaml
- type: input
  id: ai_model
  attributes:
    label: AI model used
    description: Optional — which model invoked the MCP tools. Useful for behavioral/agent issues.
    placeholder: "Claude Opus 4.7, Sonnet 4.6, GPT-5, etc."

- type: textarea
  id: prompt
  attributes:
    label: Exact prompt
    description: Optional — the verbatim prompt that triggered the unexpected behavior.

- type: textarea
  id: tool_calls
  attributes:
    label: Tool calls observed
    description: Optional — which unifi_* tools the agent invoked, with arguments.
    render: shell

- type: textarea
  id: raw_tool_output
  attributes:
    label: Raw tool output (sanitized)
    description: Paste the raw response from the relevant tool call. Scrub MACs/IPs/SSIDs/hostnames.
    render: json
```

Make these optional so non-MCP bugs aren't friction-stalled by irrelevant fields.

### Canonical example of template impact

- **Issue #297** (before template): Reporter wrote "MacOS, Claude Code Desktop via uvx" with zero controller info. Three follow-up comments later, triage still couldn't reproduce.
- **Issue #298** (after template, commit `b8e8054`): Reporter included hardware ("UDM Pro") and raw `/rest/user` output on first filing. Fix confirmed within 5 minutes of having live credentials.

The template generalizes the winning pattern of issue #298. The goal is to make #298-quality reports the default, not the exception.

## Procedure C: Evidence-First Bug Triage

Do not implement code changes based on user reports without first establishing reproduction evidence. UniFi API behavior varies by controller firmware — a defensive patch without understanding the variance is either unnecessary (the field is already present on test controllers) or incomplete (treating a symptom, not the root cause).

### Triage checklist (execute in order)

1. **Check template completeness** — Did the reporter provide controller hardware, UniFi OS version, and install method? If not, request those fields before proceeding. Refer them to the bug report template if they filed a blank issue.

2. **Establish reproduction** — Run a live smoke test against a real controller (or request a reproduction from the reporter with raw API output). The live-smoke Docker environment is the reference reproduction point.

3. **Inspect raw API payloads** — Before any code change, get the raw JSON from the relevant endpoint (`/stat/sta`, `/rest/user`, `/proxy/protect/api/cameras`, etc.). The missing field may be present under a different name, absent only in older firmware, or already handled by the codebase.

4. **Rule out alternative explanations:**
   - Version drift (plugin users have pinned aiounifi; source installs may not)
   - Cached vs. live data (controller-side polling lag vs. stale snapshot — `/stat/sta` is live-polled; `/rest/user` is a historical snapshot updated infrequently)
   - API contract differences between hardware families (UDM Pro, Cloud Key, and self-hosted behave differently at the same OS version)

5. **Confirm hypothesis before coding** — Only after steps 1–4 confirm the bug is real and understood should you begin implementation.

### When a reporter provides good evidence upfront

When a reporter includes raw API payloads immediately (as in issue #298), trust the evidence and fix with high confidence. The work was front-loaded to their initial report quality; reward that with fast turnaround. Do not slow down a well-evidenced report by re-requesting information that's already there.

### Gotcha: AI-generated analysis without reproduction

If an AI assistant (including Claude) proposes a code fix for a bug report, treat that proposal as a starting hypothesis — not a confirmed diagnosis. AI analysis of bug descriptions can be overconfident when the raw API payload hasn't been inspected. Issue #297 showed this pattern: initial AI analysis proposed a code change without confirmed reproduction; the developer correctly pushed back. Always verify with live reproduction before merging any AI-proposed fix.

## Procedure D: Issue Scope Management

When follow-up comments on an existing issue or PR propose additional changes, classify each one as exactly one of three options and act accordingly.

### Option 1: Scope into the current PR

**Criteria:** The request is directly related to the root cause being fixed, the scope increase is small, and addressing it now prevents a future regression.

**Action:** Add it to the current branch. Note the addition explicitly in the PR description so reviewers understand the expanded scope.

### Option 2: Already resolved

**Criteria:** The follow-up describes behavior the current change already fixes, but the reporter doesn't realize it.

**Action:** Reply with the specific commit SHA or file/line that addresses it. Close the sub-thread. Do not add work to resolve something that's already resolved.

### Option 3: Open a new issue

**Criteria:** The request is related but distinct — it would require its own reproduction, its own fix, and its own review cycle.

**Action:** Ask the reporter to open a new issue using the bug report template. Link the new issue to the current one for context. Do not expand the current PR to absorb it.

### Decision heuristic

Ask: "Would this change require its own live-smoke reproduction to verify?" If yes → new issue. If no → scope in or already resolved.

## Procedure E: Unresponsive Contributor Unblocking

When a first-time contributor's PR fails on a trivial CI gate and they are non-responsive to change requests, push the fix directly to their fork branch and merge. Waiting for contributor response introduces delay without clear benefit when the fix is semantics-neutral.

### Criteria (all three must be true)

1. **Trivial fix** — The required change is semantics-neutral: pure formatting, linting, or import order. No logic change is involved. If there is any logic change, stop — use Option 3 below instead.
2. **Non-responsive** — The contributor has not responded to a change request within a reasonable window, and project history shows low response rates from first-time contributors.
3. **CI is the only blocker** — The PR is otherwise mergeable: passes code review, has correct logic, fits project patterns. The CI gate is the last obstacle.

### Execution pattern (from PR #288 — ruff format failure)

```bash
# 1. Add the contributor's fork as a remote
git remote add contributor-fork https://github.com/CONTRIBUTOR/REPO.git

# 2. Fetch their branch
git fetch contributor-fork BRANCH_NAME

# 3. Check out their branch locally
git checkout -b fix-contributor contributor-fork/BRANCH_NAME

# 4. Apply the trivial fix (e.g., ruff format)
ruff format src/
# or: ruff check --fix src/

# 5. Commit with a clear message
git commit -m "chore: apply ruff format to clear CI gate"

# 6. Push back to their fork branch
git push contributor-fork HEAD:BRANCH_NAME --force

# 7. Wait for CI to clear, then merge via GitHub UI
```

**Gotcha:** GitHub only allows maintainer fork push if the contributor enabled "Allow edits from maintainers" when opening the PR. If the push fails with a permission error, check that setting. If it's disabled, post a comment asking the contributor to apply the fix themselves, or make the change on a separate branch and merge as a squash commit crediting the contributor.

### Required: document the action in a PR comment

Before merging, post a comment explaining the fork push:

```
Pushed a format-only fix directly to your branch to clear the ruff CI gate.
No logic changes — wanted to get your contribution merged without further
back-and-forth. Thanks for the contribution!
```

This preserves the contributor's intent, documents the maintainer action for future reviewers, and helps the contributor understand what happened.

### Boundary: logic changes require contributor sign-off

If the CI failure requires any logic change, do not push to the fork. Open a blocking review comment explaining exactly what's needed and why, and wait. A logic change without the contributor's review risks shipping code they didn't intend.

## Cross-Cutting Gotchas

**Template fields can't be conditionally required** — GitHub Issue Forms don't support `required: true` conditioned on another field's value. Use description text ("Required if reporting a Network app bug") to communicate intent. Validate completeness manually during triage step 1.

**`blank_issues_enabled: false` doesn't block all unstructured reports** — Users can still file issues via the GitHub API or direct URL manipulation. The template reduces friction; it doesn't enforce completeness. Triage step 1 is still necessary.

**Fork push permission depends on contributor settings** — The "Allow edits from maintainers" checkbox must be enabled when the PR is opened. If a push fails, check this before assuming a permissions misconfiguration on the maintainer side.

**Responsiveness heuristic is calibrated per-project history** — The low-response pattern was established from unifi-mcp's history with first-time contributors. Re-evaluate if contributor engagement shifts (e.g., a surge of active contributors after a high-visibility release may warrant more patience before fork-pushing).

**Evidence-first applies to AI-generated analysis too** — When Claude or another AI assistant surfaces a probable fix during triage, treat it as a hypothesis. Confirm with live reproduction before committing. The cost of a misdiagnosed fix exceeds the cost of one follow-up comment asking for raw payload evidence.
