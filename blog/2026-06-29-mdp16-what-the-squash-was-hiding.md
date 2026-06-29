---
layout: post
title: "What the squash was hiding"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-connectors]
tags: [workflow, git, squash, tooling]
---

We came back to connectors to check whether Pages finishing its WebSocket support (#52, #53) unlocked anything here. It did — connectors #27, the chat-demo WebSocket endpoint, had been explicitly blocked on those two Pages issues.

But #27 was already done. The code had been on main for two days. The commit message said `Closes #27`. GitHub never closed it.

The commit lived on branch `issue-26-demo-chat-service`, which work-end had squashed into a single commit on main with `Closes #26`. The squash dropped the inner `Closes #27` because git-squash only carried the COVERS list — the primary issue. Sub-issue references from individual commits vanished.

The branch itself was still sitting there, six commits ahead of its merge base, looking for all the world like unmerged work. No `chore: branch closed` stamp. The workspace had `EPIC-CLOSED.md`, confirming work-end had run — but the project branch was untouched. A fresh session would reasonably conclude this was an abandoned branch with lost work.

Two gaps, both in work-end's Step 8j:

The squash message needs to carry forward every `Closes`, `Fixes`, or `Resolves` reference found in branch commits, not just the issues in COVERS. Before squashing, scan the range for these patterns, diff against COVERS, and append any extras to the squash commit message. GitHub does the rest.

The project branch needs stamping as a mandatory step after all pushes complete. `git log -1 --format="%s" <branch>` returning `chore: branch closed` is the universal signal that a branch is archived. Without it, the branch is indistinguishable from live work.

Both fixes landed in soredium (`c6fce1b`). The hygiene scan in Step 8i already checked for unstamped branches on OTHER closed branches — the update clarifies that the current branch is now stamped in 8j itself, and the hygiene check only catches older branches that predate the fix.

The deeper pattern: work-end's close ceremony was thorough about artifacts (journals, specs, blogs, ADRs) but had a blind spot for git metadata. The issue tracker and the branch state are as much part of the close as the journal merge.
