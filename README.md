## 🪞 github

A nightly GitHub Actions run that mirrors every repository under
[jshvn](https://github.com/jshvn) and [katoptra](https://github.com/katoptra) into Proton
Drive. Each repository becomes one git bundle, `<owner>/<name>.bundle`, under one folder of
your choosing: a mirror clone's every ref (branches, tags, pull-request heads) in one file
that `git clone name.bundle` restores. A repository that did not change uploads nothing,
because its bundle's bytes are the same and Proton's CLI skips a file whose content it
already holds; one that changed becomes a new revision, and Proton's version history is
the history of the mirror. A repository that left GitHub has its bundle moved to Proton's
trash.

```
GitHub (read-only tokens)
  -> GitHub Actions run (this repo, inside one pinned toolbox image)
       list -> clone --mirror, bundle each -> upload (one CLI call) -> confirm -> prune -> report -> ping
  -> Proton Drive <destination folder>/<owner>/<name>.bundle
  -> R2 .state/session.tar.age                    (the CLI session, and nothing else)
```

The repository is public and holds no account: every credential and every account
identifier lives in one 1Password vault and reaches a run by name. Nothing is kept between
runs but the Proton CLI session, encrypted in a Cloudflare R2 bucket.

The toolbox that runs this, its image, the Proton engine and the two workflows this
repository calls are [katoptra/lib](https://github.com/katoptra/lib)'s, at `v2`. What is
here is the mirror's own: which owners, the floor under a listing, the `stage` and `prune`
hooks the engine leaves to a mirror, its rows of the report, and one offline check.

## 🧭 How it works

One run is `task pipeline`, executed inside the toolbox image. The engine's verbs, in
order, with this mirror's two hooks:

| Step | Does |
|---|---|
| `clock` | Records the run's start. |
| `session` | Pulls the encrypted Proton CLI session from R2 and decrypts it into `.run/session`. Every CLI call afterwards seals it back when its refresh token rotated, whatever the call's exit. |
| `destination` | Lists the parent of the destination folder and refuses the run unless exactly one folder of that name exists and its UID is the one the vault names. |
| `stage` | This mirror's. `list` asks the API for every repository under each owner, with that owner's token, and refuses a listing under `LIST_FLOOR`. Then each repository is cloned as a mirror, bundled with `git bundle create --all` into `staging/<owner>/<name>.bundle`, and the clone deleted. |
| `upload` | One `proton-drive filesystem upload -f create-new-revision -d merge` of `staging/*` into the destination. The CLI skips identical files and makes a revision of changed ones. |
| `confirm` | The CLI's own summary must account for every staged file and folder with no failure, or the run fails and the next night retries everything, which costs nothing for what already landed. |
| `prune` | This mirror's. Lists each owner's folder in Proton and trashes bundles whose repository is no longer listed. A renamed repository is trashed under its old name and uploaded under the new. |
| `report`, `ping` | The run summary on the Actions job page, then the healthchecks.io ping. |

Bundles are byte-for-byte reproducible for an unchanged repository; `task run -- task
offline` bundles a throwaway repository twice inside the image and compares, since the
skip rests on it.

## 📦 Requirements

- **A container engine**, running: Apple `container` on macOS, or Docker. Every command
  here runs inside `ghcr.io/katoptra/toolbox:proton-v2`. Nothing else is installed on the
  host.
- **[go-task](https://taskfile.dev/)**: `brew install go-task`.
- **The 1Password CLI** `op`, signed in, for anything that needs the vault on the laptop.
- **Accounts**: two GitHub tokens, Proton Drive, a Cloudflare R2 bucket, a healthchecks.io
  check, and a 1Password vault.

## 🚀 First-time setup

Every value that names an account is stored in the vault and referenced in
[op.env](op.env), ten `op://<vault>/github/<section>/<field>` lines, one item with five
sections:

| Section | Fields | Reaches a run as |
|---|---|---|
| `github` | `token_jshvn`, `token_katoptra` | `MIRROR_GITHUB_TOKEN_JSHVN`, `MIRROR_GITHUB_TOKEN_KATOPTRA` |
| `proton` | `destination`, `destination_uid` | `MIRROR_PROTON_DESTINATION`, `MIRROR_PROTON_DESTINATION_UID` |
| `r2` | `access_key_id`, `secret_access_key`, `endpoint`, `bucket` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL_S3`, `MIRROR_R2_BUCKET` |
| `age` | `identity` | `MIRROR_AGE_IDENTITY` |
| `healthcheck` | `url` | `HEALTHCHECK_URL` |

1. **1Password.** An item `github` with the five sections above, in the vault `op.env`
   names by UUID. This organization keeps one vault, `Katoptra`, with one item per mirror,
   and one service account that reads it, stored as the organization secret
   `OP_SERVICE_ACCOUNT_TOKEN`. A fork makes its own vault and service account and puts
   the token in a repository secret of the same name.
2. **GitHub.** One fine-grained personal access token per owner, since a fine-grained token
   has exactly one resource owner: one with jshvn as the owner, one with katoptra, each
   for all repositories with `Contents: read` and `Metadata: read` and nothing else. The
   mirror can never write to GitHub. An owner whose token lists another owner's
   repositories fails the run. Store them as the two fields of section `github`.
3. **Proton Drive.** The CLI can only be seeded by a browser sign-in, so the session is
   made once on the laptop and carried to CI encrypted. It is this mirror's own: two
   mirrors sharing one session race its rotating refresh token, and the loser needs a
   fresh login. Install the `proton-drive` CLI at the version
   [katoptra/lib's lock](https://github.com/katoptra/lib/blob/main/toolchain.lock.toml)
   pins, then:

   ```bash
   PROTON_DRIVE_CACHE_DIR=.run/pd PROTON_DRIVE_CREDENTIALS_STORE=unsafe_file proton-drive auth login
   PROTON_DRIVE_CACHE_DIR=.run/pd PROTON_DRIVE_CREDENTIALS_STORE=unsafe_file proton-drive filesystem create-folder /my-files GitHub
   PROTON_DRIVE_CACHE_DIR=.run/pd PROTON_DRIVE_CREDENTIALS_STORE=unsafe_file proton-drive filesystem list -j /my-files
   ```

   Store `/my-files/GitHub` as `destination` and, from the listing, the `uid` of the entry
   whose `name.value` is `GitHub` as `destination_uid`. Then, once the bucket exists,
   `task session-seal -- .run/pd`.
4. **age.** `age-keygen` once; the `AGE-SECRET-KEY-...` line is `identity`.
5. **Cloudflare R2.** A bucket and an API token scoped to it. It holds one object.
6. **healthchecks.io.** A check on the nightly schedule with a grace period of an hour or
   two; its ping URL is `url`.
7. **Prove it.**

   ```bash
   task image                  # pull the toolbox image once
   task check                  # every pipeline command rendered inside the image, diffed against render.txt
   task run -- task offline    # the bundle reproducibility check
   task plan                   # the real thing, read-only: session, destination, list, clone and bundle; nothing uploaded
   ```

8. **First run.** Dispatch the `sync` workflow from the Actions tab or with `gh workflow
   run sync.yml`. The first run uploads every bundle; a quiet night afterwards uploads
   none.
9. **Schedule.** Register `sync.yml` in jshvn/dispatch, once a day.

## ▶️ Running it

`task` alone prints the menu. Everything runs inside the toolbox.

```bash
task plan                       # read-only: list, clone and bundle; prints what an upload would carry
task sync                       # one run, the same thing CI runs
task session-seal -- .run/pd    # encrypt a laptop Proton CLI session into R2
task empty-trash                # permanently delete Proton trash; asks first; never scheduled
task check                      # render every pipeline command inside the image, diff against render.txt
task run -- task offline        # bundle a throwaway repository twice; the two must match
task clean                      # delete .run, staging and the Taskfile cache
```

Never run `task sync` or `task plan` from the laptop while a CI run may be in progress:
both hold the one Proton session.

## ⚙️ GitHub Actions

[sync.yml](.github/workflows/sync.yml) is dispatch-only and calls katoptra/lib's reusable
`sync.yml`, which installs go-task and the 1Password CLI at the versions in lib's lock,
pulls the image and runs `task sync`. `concurrency: {group: sync, cancel-in-progress:
false}` queues a second dispatch behind a running one. A pipeline that fails still reports
and pings `/fail` inside the same container. Nothing in this repo starts a run; the nightly
dispatch comes from jshvn/dispatch.

[check.yml](.github/workflows/check.yml) runs on pull requests: the render diff and the
offline check, neither with access to the vault.

On a public repository the run logs are public. They carry counts, the phase lines and the
report table. They never carry a repository name, a path in Proton, a token or an account
identifier: git's stderr goes to a file, a failure names a repository by its position in
the listing, the CLI's stderr goes to `.run/pd.err`, and `op run` masks every value it
resolved.

## 📊 Reading a run

The step summary is counts alone: repositories listed and bundled, files and MB staged,
what Proton reported (uploaded, skipped as identical, failed), bundles pruned, whether the
session was restored. A quiet night reads "0 uploaded, 24 skipped as identical": the 22
bundles and their two folders.

## 🔧 Configuration

Three root vars in [Taskfile.yml](Taskfile.yml): `OWNERS`, the space-separated owners,
each needing a `MIRROR_GITHUB_TOKEN_<OWNER>` line in `op.env`; `LIST_FLOOR`, the count
under which a listing is refused; and the includes' `IMAGE`. The account is the
environment, all of it `op.env`.

## 🚫 What is not mirrored

Issues, pull-request discussion, release notes and attachments are not in git and GitHub's
migration export API is closed to ordinary accounts, so they are not here. Wikis would be
one more clone each; no repository under either owner has one today.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
