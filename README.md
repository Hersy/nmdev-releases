# nmdev

English · [Español](README.es.md)

A command-line tool for a two-app mobile team (iOS and Android) on Bitbucket and Jira: it takes a
ticket to a posted PR, answers reviews, reviews other people's PRs, keeps epics aligned with
`develop`, builds and distributes QA/Prod binaries through Firebase App Distribution, and, in the
background, ports changes between the two apps. Agent stages run on the claude, codex or gemini CLI.

macOS only. Every verb takes `--json` (one JSON result object, always English) and `-C <DIR>` (run
in that repo instead of the current directory).

- [Install](#install)
- [Setup](#setup)
- [Pack layout](#pack-layout)
- [Verbs](#verbs)
- [Config reference](#config-reference)
- [Environment](#environment)
- [Troubleshooting](#troubleshooting)

## Install

```sh
brew install hersy/tap/nmdev
nmdev setup
```

The formula installs `nmdev`, zsh/bash/fish completions, `man nmdev` and `man nmdev-<verb>`
(Spanish pages under `man/es`), and the fastlane `Gemfile` used by `ship`. Upgrade with
`brew upgrade hersy/tap/nmdev`.

Tools nmdev checks on the login `PATH` (setup prints the install command for each one missing):
`git` (`xcode-select --install`), `graphify` (`uv tool install graphifyy`), the `firebase` CLI
(`mise use -g npm:firebase-tools`), and the CLI of the agent backend you pick (`claude`, `codex` or
`gemini`). Building apps also needs the platform's own tools (`xcodebuild`, `pod`, Gradle).

## Setup

`nmdev setup` writes `~/.config/nmdev/config.toml`. Any other verb run without a config fails with
`no config: run nmdev setup`; setup never starts on its own.

Sections, in order (`nmdev setup <section>` asks only one):

| Section | What it does |
|---|---|
| `language` | `en` or `es`, asked bilingually; everything after it is in that language |
| `pack` | the team pack: a directory (copied) or git URL (cloned) into `~/.config/nmdev/pack` |
| `tools` | lists missing tools with their install command; setup continues past them |
| `repos` | finds each repo's clone under the usual roots, or takes a path; `-` skips it |
| `forge` | Bitbucket account and token (hidden), checked live, stored in the Keychain |
| `tracker` | Jira email and token (hidden), checked live, stored in the Keychain |
| `firebase` | checks the Firebase CLI login; prints `firebase login` on failure |
| `backend` | agent backend, its default and strong model, effort; checks the CLI login |
| `sync` | background sync on or off; on runs `install` (hooks + LaunchAgent) |

Nothing is written until every section is answered and the merged config validates. Setup then
prints the keys it changed, runs `install` when `sync.enabled` is true, and finishes with a full
`doctor` and the next command to run. Re-running it edits the existing config: current values are
the defaults, existing Keychain items default to "keep". Comments in the config are dropped on
rewrite; the previous file is kept as `config.toml.bak`.

Modes:

```sh
nmdev setup                                  # interactive, every section
nmdev setup backend                          # one section
nmdev setup --pack git@git.example.com:team/nmdev-pack.git
nmdev setup --pack ~/team-pack               # a directory is copied, not linked
nmdev setup --answers answers.toml           # merge a partial config, ask only what is missing
nmdev setup --pack ~/team-pack --answers answers.toml --yes   # unattended
```

- `--pack <DIR|URL>`: team values from `preset.toml` are applied without questions. A cloned pack
  is fast-forwarded on every later `setup`; if it cannot fast-forward (local edits), setup stops and
  writes nothing.
- `--answers <FILE>`: a partial config in TOML plus an optional top-level `language`, merged over
  the current config. Tokens are refused: any key containing `token`, `password`, `secret` or
  `api_key` fails the run. Tokens are only ever typed, hidden.
- `--yes`: ask nothing; keep current values, fill defaults, `language` from `NMDEV_LANG` or the OS
  locale. Fails with the list of missing inputs if required values are still absent. Without a
  terminal, setup never prompts: it fails the same way when inputs are missing.
- No pack: setup opens a commented template of the team sections in `$VISUAL`/`$EDITOR`, re-opening
  it with the errors until it validates. Saving it unchanged aborts. Without a pack, `commit`,
  `review` and sync stay unavailable until the pack files exist.

Example `answers.toml` (neutral values):

```toml
language = "en"

[forge]
account = "dev@example.com"

[repos.app-ios]
path = "~/src/app-ios"

[repos.app-android]
path = "~/src/app-android"

[agent]
backend = "claude"

[sync]
enabled = false
```

## Pack layout

A pack is a directory or git repo the team keeps privately. nmdev defines only its layout:

```
team-pack/
├── preset.toml          partial config: the team's values
├── prompts/
│   ├── triage.md        sync triage prompt
│   ├── port.md          sync port prompt
│   └── rosetta.md       cross-platform mapping notes for porting
├── skills/
│   ├── commit-message/SKILL.md
│   ├── pull-request/SKILL.md
│   ├── pr-review/SKILL.md
│   └── pr-rebuttal/SKILL.md
└── golden.toml          the sync eval set
```

Each skill is `skills/<name>/SKILL.md` plus optional `references/*.md`. Setup points `skills_dir` at
`pack/skills` and `sync.eval_set` at `pack/golden.toml`; prompts are read from `pack/prompts/` in
place. `doctor` checks every file above is present and never contacts the pack's remote.

`preset.toml` holds any config keys. Personal keys in it only pre-fill the question (the user's
answer wins): `language`, `forge.account`, `tracker.email`, everything under `agent`,
`repos.<slug>.path`, `review.repos.<slug>.path`, and `backends.<name>.model` / `strong_model`.
Every other key is a team value and overwrites the config. Minimal example:

```toml
[repos.app-ios]
platform = "ios"
name = "iOS app"
counterpart = "app-android"
slug = "app-ios"
compile = ["xcodebuild", "build", "-quiet", "-workspace", "App.xcworkspace", "-scheme", "Dev"]

[forge]
workspace = "my-workspace"
hotfix_reviewers = ["Reviewer One"]

[tracker]
site = "https://tracker.example.com"
# ...
```

## Verbs

Flags shared by most verbs: `--json`, `-y/--yes` (skip the approval prompts), `--build qa|prod|both`
(build in the background after posting). PR1 is a feature PR into its epic; PAP is the epic's PR
into `develop`.

### Flow

- **`nmdev task <TICKET>`** — take a Jira ticket (key or browse URL) to a posted PR1: brief,
  implement, compile, commit, PR. `--repo` (default: from the ticket's platform tags), `--epic`,
  `--from-file <FILE>` (ticket text instead of Jira), `--answers <FILE>` (answers to a stopped
  brief's questions), `--fresh`, `--build`, `-y`.
- **`nmdev address`** — answer the review on this branch's PR: rebut or answer what needs no
  change, fix the rest in one commit. `--pr`, `--dry-run`, `--fresh`, `--build`, `-y`,
  `--learn-style` (build the `pr-rebuttal` skill from your own replies; spends tokens).
- **`nmdev review [URL | REPO N]`** — review someone else's open PR in your voice; posts on yes. No
  argument lists PRs waiting on you. `--verdict approve|request-changes`, `--dry-run`, `--fresh`,
  `-y`.
- **`nmdev commit`** — commit staged changes with a message from the `commit-message` skill; never
  pushes. `--epic`, `--message-file`, `-y`.
- **`nmdev align`** — merge `develop` into the epic, then the epic into this branch. `--epic`,
  `--all` (every squad epic lacking `develop`), `-y`.
- **`nmdev pr`** — post or update this branch's PR1 as committed (aligns first). `--epic`,
  `--allow-large` (over 30 files unattended), `--build`, `-y`.
- **`nmdev push`** — commit what is staged, align, post or update the PR1, with one approval.
  `--epic`, `--message-file`, `--allow-large`, `--build`, `-y`.
- **`nmdev pap`** — post or update the PAP (epic → `develop`) with its PR1s, issues, reviewers and a
  QA build. `--prod` (Prod too), `--qa-ticket <KEY>` (comment the build links there), `-y`.
- **`nmdev hotfix`** — post or update a hotfix PR into the open release branch, else `master`;
  builds Prod then QA. `--base`, `--qa-only`, `--backport` (after merge: open the `[SYNC] master ->
  develop` PR), `-y`.
- **`nmdev ship`** — build this checkout as it is, uncommitted changes included, and distribute it.
  `--qa`, `--prod`, `--both`, `--dev <N>` (also DEV-N, repeatable; `--devN` works),
  `-m <MESSAGE>` (release header), `--no-attach` (leave the PR alone), `--keep` (keep local
  binaries).
- **`nmdev status`** — where this branch stands against `develop` and its epic, its PR, builds,
  ports, and what to run next. `--all`: every repo, from anywhere.

### Background sync

Commit hooks queue each commit; a LaunchAgent runs `nmdev sync`, which triages it and ports what
the other app lacks, per `sync.modes`.

- **`nmdev sync`** — one sync pass. Run by the LaunchAgent, never by hand.
- **`nmdev sync stats`** — tuning metrics from the ledger. `--since 7d|Nd|all`.
- **`nmdev sync replay <REPO> <SHA>`** — replay one commit on a scratch ledger. `--dry-run`, `--live`.
- **`nmdev sync eval`** — score triage and parity on the eval set. `--golden <FILE>`.
- **`nmdev sync bootstrap verify`** — check the bootstrap kit still applies and compiles on the
  Android app's `origin/develop`.
- **`nmdev sync enqueue <REPO> <SHA> [BRANCH]`** — queue a commit as the hooks do.
- **`nmdev ports [JOB] [REASON]`** — ports waiting for review: list, show one, `--accept [ITEM...]`
  (none: all; `--dry-run` commits locally only) or `--dismiss` with `--wrong` / `--wontfix`
  (`--golden <FILE>` feeds the decision to an eval set).
- **`nmdev retry <JOB>`** — re-run a failed or build-failed job from its last state.
- **`nmdev pause`** / **`nmdev resume`** — stop / restart job processing; the hooks keep queueing.
- **`nmdev install`** — install the commit hooks in every configured repo and the LaunchAgent
  `dev.hersy.nmdev.sync`.

### Maintenance

- **`nmdev doctor`** — check tools, backends, hooks, LaunchAgent, Keychain tokens, Firebase login,
  build lock, pack and config. `--fix` installs the fastlane gem bundle first.
- **`nmdev setup [SECTION]`** — write the config; see [Setup](#setup).
- `nmdev --license` prints the license.

## Config reference

`~/.config/nmdev/config.toml`. Relative paths resolve against the config directory; `~` expands.
Unknown keys are errors. "—" means required, no default.

### Top level

| Key | Default | Meaning |
|---|---|---|
| `version` | — | config format; must be `1` |
| `language` | `NMDEV_LANG`, else OS locale | `en` or `es` |
| `skills_dir` | — | directory with the skills (setup: `pack/skills`) |

### `[agent]`

| Key | Default | Meaning |
|---|---|---|
| `agent.nmdev_bin` | `nmdev` | binary the LaunchAgent and background jobs run |
| `agent.backend` | `claude` | `claude`, `codex` or `gemini`, for every stage |
| `agent.model` | unset | one model for every stage; deprecated in favor of `backends.<name>.model` (setup removes it) |
| `agent.effort` | unset | `low`, `medium` or `high` for every stage |

### `[backends.*]`

Model resolution: `NMDEV_AGENT_MODEL` > `stages.<stage>.model` > `agent.model` > the active
backend's `model` (or `strong_model` for strong stages) > built-in.

| Key | Default | Meaning |
|---|---|---|
| `backends.claude.bin` | `claude` | claude CLI binary |
| `backends.claude.setting_sources` | `""` | passed to claude's `--setting-sources` |
| `backends.claude.model` | `sonnet` | default model |
| `backends.claude.strong_model` | `opus` | model for strong stages (`resolve`, `task_implement`, `sync_resolve`) |
| `backends.codex.bin` | `codex` | codex CLI binary |
| `backends.codex.model` | built-in | default model |
| `backends.codex.strong_model` | built-in | strong model |
| `backends.gemini.bin` | `gemini` | gemini CLI binary |
| `backends.gemini.keychain_item` | `nmdev-gemini` | Keychain item holding the Gemini API key |
| `backends.gemini.model` | `flash` | default model |
| `backends.gemini.strong_model` | `pro` | strong model |

### `[repos.<slug>]`

One table per app repo, keyed by its Bitbucket slug.

| Key | Default | Meaning |
|---|---|---|
| `repos.<slug>.platform` | — | `ios`, `android`, `flutter` or `docs` |
| `repos.<slug>.name` | — | display name, also given to agents |
| `repos.<slug>.counterpart` | — | slug of the other app (sync and mirrored replies) |
| `repos.<slug>.path` | empty | local clone; empty means skipped (`doctor` reports it) |
| `repos.<slug>.slug` | — | Bitbucket repo slug |
| `repos.<slug>.compile` | — | argv that compiles the app (agent and sync gate) |
| `repos.<slug>.prepare` | `[]` | argv run before compiling in a fresh worktree (e.g. `pod install`) |
| `repos.<slug>.prepare_when_changed` | `""` | re-run `prepare` only when this file changed |
| `repos.<slug>.symlinks` | `[]` | untracked files linked into worktrees (e.g. `local.properties`) |
| `repos.<slug>.ship.workspace` | unset | iOS: `.xcworkspace` to build |
| `repos.<slug>.ship.output_dir` | unset | where binaries land |
| `repos.<slug>.ship.firebase_app` | unset | Firebase app ID for every target |
| `repos.<slug>.ship.targets.<target>.tag` | — | build tag shown in releases (e.g. `QA`) |
| `repos.<slug>.ship.targets.<target>.scheme` | unset | iOS scheme |
| `repos.<slug>.ship.targets.<target>.task` | unset | Android Gradle task |
| `repos.<slug>.ship.targets.<target>.variant_dir` | unset | Android variant output dir |
| `repos.<slug>.ship.targets.<target>.firebase_app` | `ship.firebase_app` | per-target Firebase app ID |

### `[stages.<stage>]`

Per-stage overrides. Stages: `message`, `assess`, `resolve`, `task_brief`, `task_implement`,
`address_triage`, `address_fix`, `address_style`, `pr`, `review`, `review_style`, `triage`,
`parity`, `port`, `sync_message`, `sync_resolve`, `sync_pr`.

| Key | Default | Meaning |
|---|---|---|
| `stages.<stage>.backend` | `agent.backend` | backend for this stage |
| `stages.<stage>.model` | resolved as above | model for this stage |
| `stages.<stage>.effort` | `agent.effort` | `low`, `medium`, `high` |
| `stages.<stage>.max_turns` | per stage | turn cap |
| `stages.<stage>.read` | per stage | may read files |
| `stages.<stage>.edit` | per stage | may edit files |
| `stages.<stage>.shell` | per stage | allowed shell command prefixes |

### `[forge]` (Bitbucket)

| Key | Default | Meaning |
|---|---|---|
| `forge.workspace` | — | Bitbucket workspace |
| `forge.account` | — | your Bitbucket account (personal) |
| `forge.reviewers` | `work` | PR1 reviewers: `default` (repo defaults), `work` / `pap` / `sync` (copy your last PR of that kind), or comma-separated names |
| `forge.hotfix_reviewers` | — | hotfix reviewers, by name |

### `[tracker]` (Jira)

| Key | Default | Meaning |
|---|---|---|
| `tracker.site` | — | e.g. `https://tracker.example.com` |
| `tracker.cloud_id` | — | Jira cloud ID |
| `tracker.epic_key` | — | project key of epics (`NMDEV_EPIC_KEY` overrides) |
| `tracker.project_keys` | — | project keys tickets come from |
| `tracker.started` | — | status set when `task` starts |
| `tracker.review` | — | status set when the PR is posted |
| `tracker.order` | — | workflow statuses in order |
| `tracker.task_types` | — | issue type → branch prefix, e.g. `{ Bug = "fix/", Task = "feat/" }` |
| `tracker.task_tags` | — | ticket platform tag → repo slug, e.g. `{ IOS = "app-ios" }` |
| `tracker.refused_tags` | `[]` | tags `task` refuses (another team's platform) |

### `[distributor]` (Firebase App Distribution)

| Key | Default | Meaning |
|---|---|---|
| `distributor.firebase_project` | — | Firebase project ID |
| `distributor.tester_groups` | — | tester groups, comma-separated |
| `distributor.team_id` | unset | Apple team ID for iOS signing |
| `distributor.dev_envs` | — | DEV environment numbers `ship --dev` accepts |

### `[filters]`

| Key | Default | Meaning |
|---|---|---|
| `filters.message_patterns` | — | regexes: commits whose message matches are not synced |
| `filters.plumbing_globs` | — | files treated as plumbing (not ported) |
| `filters.forbidden_globs` | — | files agents and ports must never touch |

### `[sync]`

| Key | Default | Meaning |
|---|---|---|
| `sync.enabled` | `true` | background sync on (hooks queue, LaunchAgent runs) or off |
| `sync.modes` | — | `"from->to" = "shadow" \| "local" \| "live"`: record only, commit locally, or push and open the PR |
| `sync.bootstrap_kit` | — | directory of the bootstrap kit |
| `sync.eval_set` | `golden.toml` | eval set (setup: `pack/golden.toml`) |
| `sync.max_ports_per_day` | `10` | daily port cap |
| `sync.max_jobs_per_day` | `30` | daily job cap |
| `sync.release_prefixes` | `["release/", "releases/"]` | release branch prefixes |
| `sync.assess_threshold` | `0.6` | upkeep declines a merge assessed at or above this risk |
| `sync.epic_upkeep` | `false` | keep epics aligned with `develop` in the background |
| `sync.pap_upkeep` | `false` | keep PAPs updated in the background |

### `[review]`

| Key | Default | Meaning |
|---|---|---|
| `review.legacy_runs` | unset | directory of earlier review runs to reuse |
| `review.repos.<slug>.platform` | — | platform of a repo you review |
| `review.repos.<slug>.path` | the `repos` clone | clone to review in; required when the slug is not under `repos` |

### `[address]`

| Key | Default | Meaning |
|---|---|---|
| `address.rebuttal_repos` | — | repos whose PRs feed `address --learn-style` |
| `address.bot_names` | — | commenters ignored as bots |
| `address.rebuttal_thin` | `15` | below this many replies the learned style is flagged thin |

## Environment

| Variable | Effect |
|---|---|
| `NMDEV_LANG` | `en` or `es`; overrides `language` |
| `NMDEV_AGENT_BACKEND` / `NMDEV_AGENT_MODEL` / `NMDEV_AGENT_EFFORT` | override backend, model, effort for every stage |
| `NMDEV_EPIC_KEY` | overrides `tracker.epic_key` |
| `NMDEV_FLOW_STATE` | flow state dir (default `~/.local/state/nmdev/flow`) |
| `NMDEV_BUILD_LOCK=0` | disable the build lock; `NMDEV_BUILD_LOCK_DIR` moves it |
| `NMDEV_BUILD_TIMEOUT` | build timeout in seconds |
| `NMDEV_SHIP_KEEP=1` | as `ship --keep` |
| `NMDEV_GEMFILE` | fastlane Gemfile for dev builds |
| `CODEX_API_KEY` / `CODEX_HOME` | codex authentication |

## Troubleshooting

**Start with `nmdev doctor`.** One line per check: `✓` passed, `✗` failed, `·` informational.

```
✓ git on PATH: /usr/bin/git
✓ app-ios repo: /Users/me/src/app-ios
✗ app-android repo:
· LaunchAgent runs `nmdev sync`: not enabled
✓ Bitbucket token in Keychain: dev.hersy.nmdev.bitbucket
✗ Jira token in Keychain (task, pr, pap --qa-ticket): dev.hersy.nmdev.jira
✓ pack (commit, review, sync): /Users/me/.config/nmdev/pack
✓ worker not paused: running
```

Fix what fails and re-run. A config error names the file and the offending key.

**Keychain items** (login keychain, written by setup, never on the command line):

| Item (service) | Account | Holds |
|---|---|---|
| `dev.hersy.nmdev.bitbucket` | `forge.account` | Bitbucket token |
| `dev.hersy.nmdev.jira` | your Jira email | Jira token |
| `nmdev-gemini` (`backends.gemini.keychain_item`) | — | Gemini API key, only with the gemini backend |

Rotate a token with `nmdev setup forge` / `nmdev setup tracker` and pick "replace". Inspect with
`security find-generic-password -s dev.hersy.nmdev.jira`.

**CLI logins** are owned by each CLI; nmdev checks, never stores them. On failure setup prints the
command, offers retry or skip; log in from another terminal:

| Backend / tool | Login |
|---|---|
| claude | `claude auth login` |
| codex | `codex login` (or set `CODEX_API_KEY`) |
| gemini | API key in the Keychain item above (`nmdev setup backend`) |
| Firebase | `firebase login` |

**Background sync on/off.** `nmdev setup sync` toggles `sync.enabled`. On: setup runs `install`
(hooks in every repo, LaunchAgent `dev.hersy.nmdev.sync`). Off: hooks stop queueing and `doctor`
shows hooks and LaunchAgent as `· not enabled`. To pause without changing config: `nmdev pause` /
`nmdev resume`. Worker log: `~/.local/state/nmdev/sync/worker.log`.

**Language.** Order: `NMDEV_LANG` > `language` in the config > OS locale (`LC_ALL`, `LANG`; `es*` →
Spanish) > English. `NMDEV_LANG=en nmdev doctor` for a one-off. `--json` output is always English.
Change it permanently with `nmdev setup language`.

**Missing repo.** Clone it, then `nmdev setup repos`. Setup never clones repos.

**Pack won't update.** Setup stops if `~/.config/nmdev/pack` cannot fast-forward; resolve the local
changes there and re-run `nmdev setup`.
