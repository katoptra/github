# github

This is a daily mirror of each repository of the owners in `OWNERS`. It puts one git bundle
for each repository into one Proton Drive folder. `README.md` tells:

- Which repositories the mirror copies
- How to get a repository from a bundle
- How the mirror operates
- How to fork it.

The README of [katoptra/lib](https://github.com/katoptra/lib) is the manual for the proton
engine and for the parts that more than one mirror uses. This file gives the rules that each
change must obey.

No part of this repository starts a run. An external scheduler dispatches `sync.yml` daily,
at 17:42 UTC.

## Constraints

- Where a change goes: `Taskfile.yml` and its comments are the design of these parts of this
  mirror:
  - `OWNERS` and `LIST_FLOOR`, in the root vars
  - `list`
  - `stage` and `prune`, the two hooks of the engine
  - `report-mirror`
  - `offline`.
- All other changes go to the proton engine of lib, where each proton mirror gets them. The
  pipeline is a part of the engine.
- The excludes are `report-mirror` on the toolbox include, and `stage` and `prune` on the
  proton include.
- The root vars contain only the values of this mirror. No root var contains a copy of an
  engine default
  ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
- For a change that adds storage, calculate the new total. Compare it with the baseline
  (31 bundles and 325.7 MB, the run of 2026-10-08) and with the storage limit of the Proton
  plan.
- Writing: obey ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

- **A repository that did not change must give the same bundle.** `git bundle create --all`
  makes the same bytes from two mirror clones of a repository that did not change. Thus, the
  upload of the engine does not send that bundle again, and the bucket keeps only the
  session. `offline` does a test of this. Do not add to a bundle an item that is different
  from one clone to the next.
- **One read-only token for each owner, and `list` rejects a listing with a different
  owner.** A fine-grained token has one resource owner. A listing with a repository of a
  different owner is the result of interchanged tokens. A listing of less than `LIST_FLOOR`
  repositories is the result of an incorrect token or an API problem, not an empty account.
  In the two conditions, the run stops before `prune` can move a bundle to the trash. The
  `vars` input of `sync.yml` can set `LIST_FLOOR` and `OWNERS` for `list` and `prune`, and
  `LIST_FLOOR=0` there removes this guard.
- **This mirror has a session that no other mirror uses.**
  [lib, The session](https://github.com/katoptra/lib#the-session) gives the rules for a
  session.
- **Each person can read the run logs.** The stderr of git goes to `.run/git.err`, and the
  stderr of the CLI goes to `.run/pd.err`. But the stderr of `list-folder`, which `prune`
  uses, goes to the log. An error message gives the position of the repository in the
  listing, not its name. The token goes to git in its environment, as a header, not on a
  command line.
- **If a repository gets a new name, its bundle with the previous name goes to the trash.**
  The mirror uploads a bundle with the new name. The trash in Proton Drive keeps the
  previous bundle. Only a person runs `empty-trash`, and it shows a prompt first.
- **A run failure is the only alert.** The check in healthchecks.io has the cron
  `42 17 * * *` UTC and a grace time of 3 h.

## Verifying a change

- `task check` makes a render of each command of the pipeline in the image. Then it compares
  the render with `render.txt`. `task render-update` accepts a change.
- The `check` workflow uses the check workflow of lib, with `offline: true`. Thus, on each
  pull request, it runs `task check` and then `task run -- task offline`.
- `task run -- task offline` is the one check without credentials. It makes two bundles of a
  test repository in the image, and the two bundles must be the same.
- `task plan` gets the listing from GitHub, and makes the clones and the bundles, but it
  uploads no file. For `task plan`, the vault and the session are necessary.
- The checks of the engine are in lib: `cd ../lib/examples/proton && task run -- task offline`.
