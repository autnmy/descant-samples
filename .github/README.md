# Descant sample: one real run

This repository holds one pull request that Descant planned, wrote, reviewed and merged on its own: [#2, "Serialize combinations of enum flags in migrations"](https://github.com/autnmy/descant-samples/pull/2). Everything on that pull request is what Descant produced, as it happened. For this page, an internal run id and an internal section were removed from the pull request description, and one issue comment linking to our private dashboard was deleted; nothing else was changed.

## The task

The task is [issue #1](https://github.com/autnmy/descant-samples/issues/1), a real Django bug taken from a public benchmark of open-source issues. The issue text is the benchmark's own wording. It was posted by Descant's bot account when the task was set up, so the account shown as its author did not write it.

## What the repository contains

- **Django's source** at the commit the task starts from. It is under Django's BSD licence, which is in [LICENSE](https://github.com/autnmy/descant-samples/blob/main/LICENSE) unchanged. Django's earlier history is not included, so the first commit is one snapshot of that source, made by the setup account.
- **A few setup commits** that prepared the copy for the run. They are part of the harness, not of the change.
- **The pull request and its review.** Descant opened #2, answered its own review, and merged it.
- **Four open issues, #3 to #6.** While it worked, Descant found that four CI checks (`black`, `flake8`, `docs` and a Windows test job) already fail on `main`, before its change. It filed each one as an issue instead of counting them against its pull request. That is why the pull request reads "Target CI: failure; merged automatically": the failures come from the base branch, not from the change.
- **The merge commit** names the organisation the repository lived in during the run.

## What it cost

This run cost **$12.98** and took about **67 minutes** from start to merge. That time includes waiting on CI runs, review rounds and the merge, not only model work. See [pricing](https://descant.run/pricing) for what a typical pull request uses.
