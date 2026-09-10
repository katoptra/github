# github

A nightly mirror of every repository under `OWNERS` into one Proton Drive folder, one
git bundle each. `README.md` says what it mirrors, how to restore a bundle, how it works
and how to fork it; [katoptra/lib](https://github.com/katoptra/lib)'s README is the
manual for the proton engine and everything the mirrors share. This file is what a
change must not break.

Nothing in this repo starts a run: an external scheduler dispatches `sync.yml` nightly.
`Taskfile.yml` and its comments are the design of what is this mirror's own: `OWNERS`
and `LIST_FLOOR` in root vars, `list`, the engine's two hooks `stage` and `prune`,
`report-mirror` and `offline`. The pipeline is the engine's.

## Must knows

- **The skip rests on reproducible bundles.** `git bundle create --all` from two mirror
  clones of an unchanged repository is byte-identical, which is why the engine's upload
  skips it and why nothing but the session is kept in the bucket. `offline` proves it;
  do not add anything to a bundle that varies between clones.
- **One token per owner, read-only, and `list` refuses a listing that names another
  owner.** A fine-grained token has one resource owner. A token that lists the wrong
  owner is a swapped credential, and a listing under `LIST_FLOOR` is a broken token or
  API, never an emptied account; either fails the run before `prune` can trash on it.
- **The session is this mirror's own.** Two mirrors sharing one race its rotating refresh
  token. Never run `sync` or `plan` from a laptop while an Actions run may be going.
- **The logs are public.** git's stderr goes to `.run/git.err`, the CLI's to
  `.run/pd.err`, a failure names a repository by its position in the listing, and the
  token reaches git as a header through its environment, never on a command line.
- **A renamed repository is trashed under its old name and uploaded under the new.**
  Proton's trash keeps it; `empty-trash` is by hand and asks first.
- **Unverified until a real run shows it:** whether the CLI uploads the dotfile bundle
  `katoptra/.github.bundle`, and the exact JSON shape of `filesystem list -j`, which
  `destination` and `prune` unwrap as `{"value": ...}` the way dropbox's provider does.

## Verifying a change

- `task check` renders every command of the pipeline inside the image and diffs it
  against `render.txt`; `task render-update` accepts a change. The `check` workflow does
  the same on every pull request, then `task run -- task offline`.
- `task run -- task offline`: a throwaway repository bundled twice inside the image; the
  two must match.
- `task plan`: the real listing, clones and bundles, nothing uploaded. Needs the vault
  and the session.
- The engine's own checks run in lib: `cd ../lib/examples/proton && task run -- task offline`.
