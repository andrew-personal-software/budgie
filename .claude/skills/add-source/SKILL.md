---
name: add-source
description: Use when a Sparrow BUDGIE task asks to add a blog, feed or source to budgie (often a one-shot task whose body is just URLs), or when asked to subscribe budgie to a new site.
---

# Add a source

## Overview

A new source is one `sources.yaml` entry with a short `title`. The deploy seeds it. You own the task to the end:
code, PR, merge, deploy, local sync, task status.

## Steps

1. **Read the task.** `sparrow show BUDGIE-N`. The URLs are in `body`.
2. **Branch first, from a clean, current `main`.** `git status --short` must be empty (if not, stop and ask).
   Then `git checkout main && git pull && git checkout -b feat/budgie-N-<slug>`.
3. **Find the feed.** Prefer a real feed over the home page:
   `curl -sL <home> | rg -io '<link[^>]*rel="alternate"[^>]*>'`, then try `/feed`, `/rss.xml`, `/feed.xml`,
   `/index.xml`. Substack: `/feed`. Squarespace: `?format=rss`. Medium: `https://medium.com/feed/<publication>`.
   With several feeds, pick the one with full posts only (e.g. `/atom/entries/`, not `/atom/everything/`).
4. **Probe on a scratch DB** (never the real one). Put every URL from the task in one file. Set `S` to your
   session scratchpad dir (or `S=$(mktemp -d)`). Keep `DATABASE_URL` inline, so later commands do not use it:

   ```sh
   printf 'sources:\n  - url: <url1>\n    title: <Title1>\n  - url: <url2>\n    title: <Title2>\n' > $S/one.yaml
   rm -f $S/t.sqlite; DB=sqlite:///$S/t.sqlite
   DATABASE_URL=$DB uv run alembic upgrade head && DATABASE_URL=$DB uv run newsletter seed $S/one.yaml \
     && DATABASE_URL=$DB uv run newsletter ingest
   ```

   It works when seed prints `ok <url> -> <adapter>: <Title>` and ingest prints `ok <Title>: <n>` with n > 0.
   A first fetch takes at most 5 posts, so 5 is normal. `FAILED` means no adapter reads it: follow README
   "When no feed is found" (`uv run newsletter ticket <url> --parent BUDGIE-N`).
5. **Add the entry** at the end of its section in `sources.yaml` (RSS / Atom, Sitemap, Listing):

   ```yaml
   - url: https://www.curttigges.com/rss.xml
     title: Curt Tigges
   ```

   `title` is the name readers know the writer by, short. A company: `Netflix` (not `Netflix TechBlog - Medium`).
   One person: their name, even when the feed names a show or blog (`Dwarkesh Patel`, not `Dwarkesh Podcast`;
   `Toby Ord`, not `Writing - Toby Ord`). A publication with many writers: its own name (`Asterisk`,
   `ChinaTalk`). No "Blog", "Engineering", "- Medium". Every entry has one.
6. **Check.** `git diff sources.yaml` shows your lines (edit with the Edit tool; multi-line `sd` can match
   nothing and exit 0). Then `uv run pytest -q`.
7. **PR.** Commit and PR title: `feat(sources): add <site>[, <site>] (BUDGIE-N)`. CI fails the title without the
   task key at the end. `git push -u origin HEAD`, then
   `gh pr create --title "<title>" --body "<body>"`. Body: `BUDGIE-N`, one line per source (adapter and feed
   path), and the probe result (`seed` + `ingest` on a scratch DB: n items). Note the PR number it prints.
8. **Merge.** `gh pr checks <pr> --watch`. Checks take a few seconds to show up: "no checks reported" at first
   is not a pass, so wait and run it again. When all pass: `gh pr merge <pr> --squash --delete-branch`.
9. **Deploy.** The `release` run on `main` starts a few seconds after the merge:
   `gh run list --workflow release.yml --branch main -L1 --json databaseId,headSha`. Check that `headSha` is the
   merge commit, then `gh run watch <id> --exit-status`. Jobs `test`, `hawk / release` and `hawk / deploy`
   must succeed. The worker seeds new entries when it starts.
10. **Task.** hawk sets the task to `done` with a "budgie X.Y.Z deployed" comment. Check
    `sparrow show BUDGIE-N`. If it is not done: `sparrow update BUDGIE-N --status done`.
11. **Sync local.** `git checkout main && git pull && uv sync`.
12. **Report.** The PR link, the version deployed, the task status. `/sources` needs the owner's login, so ask
    them to check the new names at `https://budgie.andrewminghanjiang.com/sources`.

## Common mistakes

| Mistake | Fix |
|---|---|
| Unrelated changes end up in the PR | Branch from a clean `main` (step 2) |
| No `title`, so the page shows the raw feed title | Add `title:` (step 5) |
| Home page URL when a feed exists | Use the feed URL (step 3) |
| Edit did not apply, but the PR went up | `git diff` before commit (step 6) |
| Merged on "no checks reported" | Wait for the checks to show and pass (step 8) |
