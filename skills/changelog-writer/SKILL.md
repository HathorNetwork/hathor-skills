---
name: changelog-writer
description: Generate changelogs for Hathor release PRs and GitHub Releases. Use when bumping versions, writing release notes, or creating changelogs for bump PRs. Only applicable in client repositories (wallet-lib, wallet, wallet-headless, wallet-mobile, wallet-service, rpc-lib, explorer, explorer-service).
---

# Release Changelog — Step by step

## Allowed repositories

This skill is scoped to client repositories only. Before proceeding, verify the current repo by running:

```bash
git remote get-url origin
```

The remote URL **must** match one of these repositories:

- `HathorNetwork/hathor-rpc-lib`
- `HathorNetwork/hathor-wallet-headless`
- `HathorNetwork/hathor-wallet-mobile`
- `HathorNetwork/hathor-wallet`
- `HathorNetwork/hathor-wallet-lib`
- `HathorNetwork/hathor-explorer`
- `HathorNetwork/hathor-explorer-service`
- `HathorNetwork/hathor-wallet-service`

If the current repo is **not** in this list, **stop immediately** and inform the user:
> "The changelog-writer skill is only available for client repositories (wallet-lib, wallet, wallet-headless, wallet-mobile, wallet-service, rpc-lib, explorer, explorer-service)."

---

This guide covers two complementary artifacts:
- **Bump PR**: detailed description with breaking changes and migration guide
- **Release changelog**: summarized version for the GitHub Release

---

## Part A — Bump PR

### 1. Identify included PRs

Fetch the release PR associated with the bump to list commits:

```bash
gh pr view <RELEASE_PR> --json commits --jq '.commits[].messageHeadline'
```

Filter only relevant PRs — ignore sync merges (e.g., `Merge pull request #XXXX from HathorNetwork/chore/sync-release-*`).

### 2. Find breaking changes

For each identified PR, fetch title and body:

```bash
gh pr view <PR_NUMBER> --json title,body --jq '{title, body}'
```

Look in the bodies for:
- Explicit "Breaking Changes" sections
- Type, interface, or public function renames
- Method signature changes
- API removals
- Default behavior changes that affect consumers

### 3. Build the bump PR description

The bump PR contains the **full details** of breaking changes, including code examples and migration instructions when needed.

Structure:

```markdown
### Acceptance Criteria
- Bump package to v<VERSION>

### Breaking Changes

From **#<PR> - <title>**:

<breaking change details with migration examples>

From **#<PR2> - <title2>**:

<details>

### What's included

| PR | Type | Title |
|---|---|---|
| #XXXX | feat | ... |
| #XXXX | fix | ... |
| #XXXX | refact | ... |
| #XXXX | test | ... |
| #XXXX | chore | ... |

### Security Checklist
- [x] Make sure you do not include new dependencies in the project unless strictly necessary and do not include dev-dependencies as production ones. More dependencies increase the possibility of one of them being hijacked and affecting us.
```

Sort the "What's included" table by type: feat → fix → refact → test → chore.

### 4. Update the PR via API

Use the REST API since `gh pr edit` has a bug with Projects v2:

```bash
gh api repos/HathorNetwork/<REPO>/pulls/<PR_NUMBER> -X PATCH \
  -f title="chore: bump to v<VERSION>" \
  -f body="$(cat <<'EOF'
<body here>
EOF
)"
```

---

## Part B — Release Changelog

The release changelog is a **summarized** version that references the bump PR for details.

### 5. Build the release changelog

Rules:
- Breaking changes are **one-liners** with a reference to the source PR — do not repeat migration details
- Point to the bump PR as the source for full details and migration guide
- Group PRs by type with author: **Features**, **Bug Fixes**, **Tests & Chores**
- Format for each item: `- Description (#PR) by @author`
- Include a Full Changelog link at the end

Structure:

```markdown
### Breaking Changes

See PR #<BUMP_PR> for full details and migration guide.

- Breaking change summary 1 (#SOURCE_PR)
- Breaking change summary 2 (#SOURCE_PR)

### Features
- PR title (#PR) by @author
- PR title (#PR) by @author

### Bug Fixes
- PR title (#PR) by @author

### Tests & Chores
- PR title (#PR) by @author
- PR title (#PR) by @author

**Full Changelog**: https://github.com/HathorNetwork/<REPO>/compare/v<PREVIOUS>...v<VERSION>
```

### 6. Update the release

```bash
gh release edit v<VERSION> --notes-file <file_with_body>
```
