---
name: contribute
description: How to contribute to the upstream Leonida project (ubiquitarianism/leonida) from this fork - check who is already working on what, claim an item on the coordination issue, base the branch on upstream main, test, and prepare the pull request, the done comment or a finding. Use this whenever the user asks to contribute, pick up or claim work, fix something for upstream, prepare or open a pull request for Leonida, report a finding, check whether someone already has an item, or bring the fork up to date with upstream, even if they never say contribute.
---

# Contributing to Leonida upstream

`AGENTS.md` and `CONTRIBUTING.md` hold the project's rules; read them first. This skill adds the order to do things
in, and what a cloud session can and cannot reach, which those files do not cover. Where they disagree with this
skill, they win: they are maintained upstream and this file is not.

## The two repositories

- **Upstream** is `ubiquitarianism/leonida`: public, one branch (`main`), and the only place pull requests,
  claims and findings go.
- **The fork** is `origin`. Its `main` falls behind upstream within hours, and it carries files upstream does not
  have (this `.claude/` folder).

## What a cloud session can reach

Know this before promising anything, because the claim and the pull request are the two steps a session usually
cannot do itself.

| Action on upstream | Works? | How |
|---|---|---|
| Fetch code | yes | `git fetch https://github.com/ubiquitarianism/leonida main` (anonymous) |
| Read an issue or pull request body | yes | `gh api -X GET search/issues` (below) |
| Tell whether a term appears in a comment | yes | the same search matches inside comments |
| Read the comments themselves | no | the search returns the issue, not the comment |
| Post a comment, open a pull request | no | the GitHub tools are scoped to the fork |

So the session drafts the exact text and the user posts it in the browser. If a GitHub tool call on upstream
succeeds, the user has since granted access: use the tools, but still post only what the user asked for, since it
goes out under their name in a public thread.

## The steps, in order

### 1. Fetch upstream and check that the rules have not changed

```sh
git fetch https://github.com/ubiquitarianism/leonida main
git rev-list --count HEAD..FETCH_HEAD        # how far behind this checkout is
git diff --stat HEAD FETCH_HEAD -- CONTRIBUTING.md AGENTS.md .github/pull_request_template.md
```

If the last command prints anything, read upstream's copy (`git show FETCH_HEAD:CONTRIBUTING.md`) and follow it.

### 2. Find the coordination issue and read its body

Its number changes when the thread gets long, so look it up every time:

```sh
gh api -X GET search/issues \
  -f q='repo:ubiquitarianism/leonida is:issue is:open in:title "Coordinating who works on what"' \
  -q '.items[] | "#\(.number) comments=\(.comments) updated=\(.updated_at)", .body'
```

The body has the claim rules and an "Open work" table. The table lags the thread (rows marked open that a comment
already took), so never treat a row as free without step 3.

### 3. Check that nobody has the item

Search for the item three ways, with more than one term each (the ID, the file name, the test or function name).
A query must contain `is:issue` or `is:pull-request`, or GitHub answers 422.

```sh
# pull requests that mention it (drafts included)
gh api -X GET search/issues -f q='repo:ubiquitarianism/leonida is:pull-request <term>' \
  -q '"hits: \(.total_count)", (.items[] | "#\(.number) \(.state) \(.user.login): \(.title)")'
# issues that mention it, the coordination thread's comments included
gh api -X GET search/issues -f q='repo:ubiquitarianism/leonida is:issue <term>' \
  -q '"hits: \(.total_count)", (.items[] | "#\(.number) \(.title)")'
# an existing claim of this ID
gh api -X GET search/issues -f q='repo:ubiquitarianism/leonida is:issue "claiming: <ID>"' -q '.total_count'
```

How to read the result:

- A pull request for the same thing, open or draft, means it is taken. Many drafts are `old-pr/<number>` branches
  carried over from the old repository; `CONTRIBUTING.md` says how to carry one on instead of rewriting it.
- A hit on the coordination issue means the term appears somewhere in its comments. The session cannot see who
  wrote it or when, so say so and ask the user to look, or pick another item.
- No hits anywhere is the only result that reads as free.

Do this right before claiming, not at the start of the session: the thread has grown by about a hundred comments
an hour. Keep to a few searches a minute.

### 4. Reproduce on upstream `main`, not on the fork

A failure seen on the fork may already be fixed upstream. Check out `FETCH_HEAD` in a scratch worktree outside the
repository, run the failing test there, and remove the worktree afterwards (`git worktree remove`).

### 5. Pick the ID

Use the ID from the "Open work" table when the item is listed. Otherwise make one up in the thread's style:
upper case, hyphenated, an area prefix and a short name, such as `RT-HANG-DUMP`, `TEST-WIN-STALE` or
`DOC-CUBE-STORE`. Prefixes in use: `SHADER`, `RT`, `D3D`, `VK`, `GPU`, `TEST`, `DOC`, `TOOL`, `PROBE`, `BINK`, and
`OLD-<old number>` for an item from the old-repository index. Run the claim search of step 3 on the new ID; it
must return 0.

### 6. Claim, before writing the fix

Draft this for the user to post on the coordination issue:

```text
claiming: <ID> - <one line: what will change, and in which file>
```

The claim is checked by automation, which is why the details matter:

- It has to exist before the pull request opens, and come from the account that opens the pull request.
- It must never be edited or deleted: either invalidates it, and the only repair is a new claim and a replacement
  pull request. Get the wording right before the user posts it.
- One comment may claim several IDs (comma-separated, or a bulleted list after `claiming:`).

Reproducing the problem before the claim is fine; the fix is what waits for it. Ask the user to confirm the
comment is posted before going on.

### 7. Put the work on a branch that holds only upstream `main` plus the change

A pull request has to be one topic on current `main`. A session branch that started from the fork's `main` breaks
that twice: it is behind, and it carries fork-only commits such as this skill. Check before the first push:

```sh
git log FETCH_HEAD..HEAD                    # must list only the change's commits
git diff --stat FETCH_HEAD...HEAD           # must list only the change's files
```

If anything else shows up and nothing of value has been pushed on the branch yet, restart it from upstream with
`git switch -C <branch> FETCH_HEAD` and redo the change there. This skill's file leaves the working tree when you
do; that is expected, and it stays on the fork's `main`. If the branch already has pushed work, ask the user
before rewriting it.

### 8. Make the change and test it

`AGENTS.md` ("Test without the game first") lists the commands. What it does not say:

- **Take a baseline first.** Run the same tests on untouched upstream `main` and report "before and after", because
  some tests fail here for reasons no change caused. The session runs as root, so a test that expects a read-only
  file to refuse a write fails; re-run such a file as another user to confirm:
  `runuser -u nobody -- env HOME=/tmp python3 -I tests/<file>`.
  The GPU tests run on Mesa's llvmpipe, where a few fixtures fail on every driver; `gpu-test-results/` records
  which.
- **Budget the suites** (4 cores): the translator suite about 4 minutes, the standard-library loop about 10, pytest
  about 1, the runtime build about 3 and `ctest` about 2. Run one file while iterating and each suite once at the
  end.
- **The tools are already installed** by the environment's setup script: LLVM 21 as `llvm-mc`, LLVM 19 under
  `/usr/lib/llvm-19`, SPIRV-Tools, glslang, llvmpipe, the .NET 8 SDK and the pinned Vulkan headers. If one is
  missing, the setup script failed a step; say so instead of letting the tests skip silently.
- Run `python -I scripts/check_no_game_data.py <new files>` on every file the change adds.

### 9. Prepare the pull request

Push the branch to the fork. Open the pull request only if the user asked for one and the session has upstream
access; otherwise give the user the link and the text:

```text
https://github.com/ubiquitarianism/leonida/compare/main...<fork owner>:<branch>?expand=1
```

- **Title:** `<area>: <what changed> (<ID>)`, the style of upstream's merge history.
- **Body:** fill in upstream's template (`git show FETCH_HEAD:.github/pull_request_template.md`), every field:
  the claim ID, `Depends on:` and `Fixes:` (delete a line that does not apply), what and why, what was run with
  its before and after numbers, `GPUs tested:` (the `deviceName` line of `vulkaninfo --summary` plus the OS, or
  n/a), and `AI-assisted: yes`. Tick only the checklist boxes that are true.
- Quote no game data in the body, not even in a pasted log, and cite no issue number from the old repository.

### 10. Report it

Once the pull request has a number, draft for the user to post on the coordination issue:

```text
done: <ID> -> #<PR>
```

A pull request that later conflicts with `main` is labelled `needs rebase`; rebase it on a fresh fetch of
upstream `main`, and ask the user before the force-push that needs.

## Reporting something without fixing it

Found a bug, a stale document or a missing method that will not be fixed in this session? Draft a comment for the
coordination issue rather than leaving it in the chat, so someone else can pick it up:

```text
finding: <one line saying what is wrong>
where: <file>:<line>
why: <what breaks, and for whom>
needs: <game files, particular hardware, or neither>
```

Run step 3 on it first; a finding that already has a pull request is noise in a thread this busy.

## Editing this file

It is a tracked Markdown page, so the repository's document tests read it (`tests/test_docs_cli_flags.py` and the
other `tests/test_docs_*.py`). A long option of a program outside this repository fails the first of them, which
is why the commands above use `gh api -q` and plain `git log`. Run those tests after changing this file.
