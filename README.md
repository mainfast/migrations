# Mainfast Migrations

Database migration guard for GitHub. Fails a pull request whose migrations would
fork or reorder the migration history once the PR is merged into the **current**
main, and re-checks every open PR when main moves, with a commit status that
blocks the stale ones.

Licence: MIT; free for public repos and private repos with up to 5 active contributors ([Free and Pro](https://github.com/mainfast/mainfast/blob/main/README.md#free-and-pro)).

## The problem

Two PRs each add a migration on top of the same main. Each is green on its own.
Merge the first, and the second is now broken (two Alembic heads, two Django
leaves, a Prisma timestamp older than one already applied), but nothing re-runs
its checks, so it merges too and the next deploy fails. Mainfast Migrations
checks the PR against main as it is now, and when main moves it re-checks every
open PR and marks the ones that went stale.

## What it catches

The check builds what main would look like after the merge (main's files, plus
the branch's additions and edits, minus its deletions) and inspects the result.
Problems main already had before the PR are listed separately, not blamed on it.
Every finding carries a one-line fix.

| Framework | Migration ledger | Findings (default severity) |
|---|---|---|
| Alembic | `versions/*.py`, `revision` / `down_revision` graph | `multiple-heads` (error; warning when `branch_labels` are used), `missing-parent`, `duplicate-revision` (error) |
| Django | `<app>/migrations/NNNN_*.py`, `dependencies` | `multiple-leaves`, `missing-dependency` (error), `duplicate-number` (warning) |
| Prisma | `migrations/<14-digit timestamp>_name/migration.sql` | `duplicate-timestamp` (error), `out-of-order` (**error**) |
| Rails | `db/migrate/<timestamp>_name.rb`, `db/<name>_migrate/` | `duplicate-version`, `duplicate-name` (error), `out-of-order` (**warning**) |
| Flyway | `V<version>__desc.sql` (numeric by segment, `1_1` == `1.1`) | `duplicate-version` (error), `out-of-order` (**warning**) |
| Drizzle | `meta/_journal.json` + `meta/*_snapshot.json` | `journal-fork` (**error**), `duplicate-idx`, `sql-not-in-journal`, `entry-without-sql`, `snapshot-fork`, `snapshot-missing-parent`, `journal-unreadable` (error), `idx-not-sequential`, `when-out-of-order` (warning) |

Examples, each with main having moved since the branch forked:

- **Alembic**: main added `m2` with `down_revision = 'a1'`; the branch added
  `b1` with `down_revision = 'a1'`. After the merge there are two heads and
  `alembic upgrade head` refuses. Fix: `alembic merge -m merge m2 b1`, or point
  `b1` at `m2`.
- **Django**: main added `orders/migrations/0005_add_total.py`; the branch added
  `orders/migrations/0005_add_note.py`, both depending on `0004`. `migrate`
  stops with "Conflicting migrations detected". Fix:
  `python manage.py makemigrations --merge orders`, or renumber onto `0005`.
- **Prisma**: main added `20260301000000_main`; the branch added
  `20260201000000_branch`. Databases that ran main's migrations apply the
  branch's out of order and `prisma migrate dev` reports drift (error). Fix:
  regenerate the migration on top of main.
- **Rails**: same shape with `db/migrate/20260201000000_add_name.rb` under
  main's `20260301000000_add_email.rb`. `db:migrate` still runs it, but
  environments diverge in order and `schema.rb` hides the gap (warning by
  default). Two files with the same version or name are errors
  (`DuplicateMigrationVersionError`).
- **Flyway**: the branch adds `V2.9__x.sql`, main already has `V2_10__y.sql`.
  With Flyway's default `outOfOrder=false` the branch migration is ignored on
  databases past it and `validate` fails; teams that run with `outOfOrder=true`
  accept it, so warning by default. Both sides adding `V3__*` is an error.
- **Drizzle**: both sides ran `drizzle-kit generate` on the same journal
  (`journal-fork`): the branch snapshot was built against a schema that is no
  longer main's. Fix: merge main, union the journal entries by tag, re-index
  `idx`, fix the branch snapshot's single `prevId`.

## Install

```yaml
# .github/workflows/mainfast-migrations.yml
name: Mainfast Migrations
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
  pull-requests: write   # sticky comment + stale label
  statuses: write        # the blocking mainfast/migrations commit status
jobs:
  migrations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5.1.0
        with: { fetch-depth: 0 }
      - uses: mainfast/migrations@v0
```

Full example with comments: [`examples/mainfast-migrations.yml`](examples/mainfast-migrations.yml).
Third-party actions are pinned to full commit SHAs.

### Permissions

| Permission | Why | Without it |
|---|---|---|
| `contents: read` | fetch main and `refs/pull/<n>/head` | the check cannot run |
| `pull-requests: write` | the sticky PR comment and the stale label (both go through the issues API, which accepts this permission for PRs); also lets the plan notice count PR authors | the comment and label are skipped with a warning; summary and annotations still carry the result |
| `statuses: write` | the `mainfast/migrations` commit status | no status, so nothing blocks the merge |

On PRs from forks the token is read-only: the comment, label and status are
skipped with a warning and the job summary still carries the result.

## What actually blocks a merge

The comment and the label tell people; **only the commit status blocks**, and
only when branch protection (or a ruleset) requires it. After the first run,
add `mainfast/migrations` under *Settings > Branches > main > Require status
checks to pass* (or the ruleset equivalent).

- **`pull_request`** (mode `check`): fetches current main and the PR head,
  writes the job summary, annotates the migration files, sets outputs `ok` and
  `report`, keeps one PR comment up to date, adds/removes the stale label, sets
  `mainfast/migrations` on the head (failure on errors, success otherwise), and
  fails the step on errors (`fail-on: error | warning | never`).
- **`push` to main** (mode `recheck-open-prs`): lists open PRs into main
  (`GET /repos/{o}/{r}/pulls?state=open&base=main`), fetches every head in one
  `git fetch`, re-runs the check, and on every PR sets `mainfast/migrations`:
  **failure** on the ones that went stale (plus the comment and the label),
  **success** on the clean ones (their label is removed and our comment is
  marked resolved). The push job itself does not fail; the statuses do the
  blocking. Output `stale-prs`.

The status is **on by default**. `status: false` turns it off (comment and
label only; then nothing blocks). A stale PR becomes mergeable again when its
next push re-runs the `pull_request` check and sets success.

Inputs: `mode`, `token`, `main-branch`, `config`, `comment`, `comment-author`,
`label`, `status`, `fail-on`, `max-prs`, `license` (see [`action.yml`](action.yml)).

The sticky comment is matched only when it starts with the marker **and** was
written by the token's own login (`GET /user`, else `github-actions[bot]`;
override with `comment-author`, e.g. `<app-slug>[bot]` for an App token), so a
marker pasted into someone else's comment is never edited or followed. Branch
names, file and revision names and PR titles are escaped before they reach a
comment or the job summary, and `@mentions` in them are neutralised.

## Config

`.mainfast.yml`, section `migrations:`. It is read from **main** (the base) in
both modes, so a PR cannot switch its own check off; a new setting takes effect
once it is merged. An explicit `config:` input reads a file from the checkout
instead. Reference: [`../docs/config.md`](https://github.com/mainfast/mainfast/blob/main/docs/config.md); example:
[`.mainfast.example.yml`](https://github.com/mainfast/mainfast/blob/main/migrations/.mainfast.example.yml).

```yaml
migrations:
  frameworks: auto          # or [alembic, django, prisma, rails, flyway, drizzle]
  paths:                    # replaces autodetect for that framework
    alembic: [services/api/alembic/versions]
  ignore: [legacy/**]
  outOfOrder:               # per framework; or one value for all: error | warning | off
    rails: error
    prisma: off
  label: "mainfast: migrations stale"
```

`outOfOrder` sets the severity of an ordering conflict (`out-of-order` for
Prisma, Rails, Flyway; `journal-fork` for Drizzle). Defaults: Rails `warning`,
Flyway `warning`, Prisma `error`, Drizzle `error`. Alembic and Django ledgers
are graphs without a version order: their forks (`multiple-heads`,
`multiple-leaves`) break `upgrade` / `migrate` outright and are always errors,
so their `outOfOrder` entries (default `error`) currently change nothing.

Autodetect: Alembic is a `versions/` dir with an `env.py` beside it (or scripts
that declare `down_revision`); Django is any `migrations/NNNN_*.py` dir; the
others go by file pattern. The app label for Django is the parent directory
name.

## How it reads git

Only `git ls-tree` and `git cat-file` on refs: no checkout, nothing written to
the working tree. Main is fetched to `refs/mainfast/main` and each PR head to
`refs/mainfast/pr/<n>` from `refs/pull/<n>/head`. The merge-base must be
reachable, so check out with `fetch-depth: 0`.

## CLI

```sh
npx mainfast-migrations check --main origin/main --head HEAD          # exit 1 on errors
npx mainfast-migrations check --format md
GITHUB_TOKEN=... npx mainfast-migrations recheck --github-repo owner/name   # --no-status, --no-comment
```

## Library

```ts
import { analyze, toMarkdown } from '@mainfast/migrations';
const report = analyze({ repo: '.', mainRef: 'origin/main', headRef: 'HEAD' });
```

## Development

```sh
npm install                 # at the suite root (mainfast/)
npm run all                 # here: typecheck, tests, build
```

Tests build real temporary git repos for each framework (`test/helpers.ts`),
and the recheck tests push PR heads to a local bare "origin" as
`refs/pull/<n>/head`. GitHub calls go to a fixture `fetch`
(`@mainfast/shared/mockfetch`); the global `fetch` throws in tests
(`test/setup.ts`), so no test can reach the network.

## Limits

- Django app labels come from the directory name; a custom `AppConfig.label` is
  not read. Cross-app dependencies are not validated.
- Alembic `depends_on` is ignored; with `branch_labels` present,
  `multiple-heads` drops to a warning.
- Flyway repeatable (`R__`) and undo (`U`) migrations and Java migrations are
  ignored.
- Rails `schema.rb` version and Prisma `migration_lock.toml` are not inspected.
- `extends:` in `.mainfast.yml` is accepted but not applied yet (Intact only).
- A content conflict inside one migration file is not this tool's job (Mainfast
  Intact covers what a merge dropped).

## Report a problem

Found a bug or a false positive? [Open an issue](https://github.com/mainfast/mainfast/issues) in the main Mainfast repository. Anyone can, free, with no key. Issues are public, so leave out private code and secrets.
