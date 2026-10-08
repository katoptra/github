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

`Taskfile.yml` and its comments are the design of the parts of this mirror. The pipeline is a
part of the engine.

## Constraints

- **Where a change goes.** `Taskfile.yml` has these verbs: `list`, `stage`, `prune`,
  `report-mirror` and `offline`. Make all other changes to verbs in lib: in the toolbox or in
  the proton engine. Then each mirror that includes that file gets the change.
- **Excludes.** The `excludes:` of the two includes contain these verbs:
  - On the toolbox include: `report-mirror`
  - On the proton include: `stage` and `prune`, the two hooks of the engine.
- **Root vars.** Root vars hold only the values of this mirror. Do not put an engine default
  in a root var, because then the command line cannot set it
  ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
  `OWNERS` and `LIST_FLOOR` are root vars of this mirror. `API`, `CURL` and `ACCEPT` are
  also root vars, only for `list`.
- **Storage.** For each change that adds storage, calculate the new storage. Compare it with
  the baseline (31 bundles and 325.7 MB, the run of 2026-10-08) and with the storage limit of
  the Proton plan.
- **Writing.** Use ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

- **A repository that did not change must give the same bundle.** `git bundle create --all`
  makes the same bytes from two mirror clones of a repository that did not change. Thus, the
  upload of the engine does not send that bundle again, and the bucket keeps only the
  session. `offline` does a test to make sure that the two bundles are the same. Do not add
  to a bundle an item that is different in each clone. The upload then sends the bundle
  again at each run.
- **Each owner has one read-only token, and `list` rejects a listing with a different
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
- **A run failure is the only alert.** The healthcheck in healthchecks.io has the cron
  `42 17 * * *` UTC and a grace time of 3 hours.

## Verifying a change

- `task check` makes a render of each command of the pipeline in the image. Then it compares
  the render with `render.txt`. `task render-update` accepts a change.
- The `check` workflow uses the check workflow of lib, with `offline: true`. Thus, on each
  pull request, it runs `task check` and then `task run -- task offline`, with no secrets.
- `task run -- task offline` makes two bundles of a test repository in the image, with no
  credentials. The two bundles must be the same.
- `task plan` gets the listing from GitHub, and makes the clones and the bundles, but it
  uploads no file. For `task plan`, the vault and the session are necessary.
- The checks of the engine are in lib: `cd ../lib/examples/proton && task run -- task offline`.
