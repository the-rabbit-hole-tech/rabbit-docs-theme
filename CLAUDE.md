# CLAUDE.md - docs-theme

Working agreement for this repository. It was scaffolded from `Bugs5382/project-template`;
the governance below is shared across all repos created that way.

## Enforced by hooks (run `bash .claude/hooks/install.sh` once per clone)

- Conventional Commits on commits, issue titles, and PR titles.
- No AI tells in commits/issues/PRs/comments/source; no emoji in source or commit messages (emoji
  are allowed in Markdown docs and CI workflow files).
- Pre-push: the ecosystem's format/lint/test gate must pass (Go: gofmt/vet/golangci-lint/test;
  npm: lint/test scripts; Python: ruff/pytest).

## Conventions

- Branching: never commit to `main`. Work on a feature/working branch; open a PR.
- Commits: Conventional Commits (`type(scope): description`). The operator (@Bugs5382) is the
  author of record on every commit.
- Voice: human-authored. No attribution trailers (`Co-Authored-By`, `Generated with`), no robot
  glyphs/emoji, no session framing.
- Local design notes live in a non-tracked `plan/` folder; delete a note when its work is done.
- GitHub Actions: a job id must be a plain identifier (a letter or `_`, then alphanumerics/`-`/`_`);
  put emoji and display text in the job's `name:`, never the job key. The Actionlint check
  (`.github/workflows/action-lint.yaml`) enforces this, so a malformed workflow fails at PR time
  instead of silently at startup on `main`.

## CI and Actions minutes

This is a public repo, so every job runs on GitHub-hosted runners (`runs-on: ubuntu-latest`) and
pays for its own runner start and checkout. The workflows are shaped to start as few jobs as possible:

- **Drafts run nothing.** PR workflows skip draft PRs and run on `ready_for_review`, `opened`,
  `synchronize` and `reopened`. Open a PR as a draft, run the full checks locally, push once they
  pass, and mark it ready when the work is finished. That starts one CI run. After it is ready,
  push only real fixes, batched into one push.
- **One job for the small checks.** PR Title, PR Body, PR Hygiene, the gitleaks secret scan and the
  categorizing-label check are steps of one `✅ PR Checks` job (`job-pr-checks.yaml`). Every step
  runs even when an earlier one fails, so the log shows every failure. It is the only workflow that
  reacts to `edited`: a title or body fix reruns it, not the build.
- **Pull requests only.** Test, Actionlint, GoLic and the licence check run on pull requests, not
  on push to `main`. The squash merge lands the tree the PR run already tested. Only the release
  workflows (Release Manager on `main`, Release and Publish on a published release, which calls
  Test through `workflow_call`) and Label Sync run outside a PR.
- **No no-op jobs.** The licence check is npm only; this repo has no `go.mod`.
- **Every job has a `timeout-minutes`** (10 for small checks, 15 to 30 for the licence check, Test
  and releases), so a hung job stops long before GitHub's 360-minute default. The publish
  workflow's `test` job calls a reusable workflow and cannot take one; Test's own job carries it.
- **Pinned prebuilt tools.** actionlint and gitleaks install their pinned release binaries for the
  runner's architecture and check them against the published SHA-256. To bump one, change the
  version and both checksums in its workflow.
- **Required checks:** if the ruleset ever lists required checks, use `✅ PR Checks`; it replaces
  `PR Title`, `PR Body`, `PR Hygiene`, `Gitleaks (secret scan)`, `Autolabel` and `Categorizing label
  present`.

## Engineering discipline

- Root-cause before fixing: confirm the actual cause with evidence before changing code; do not
  patch symptoms.
- Map every reference before removing a feature: trace its wiring across the tree first, preserve
  adjacent behavior that only looks related, and defer-and-flag an entangled piece rather than
  guessing it.
- Verify with evidence, not assertions: run the real check for what changed (lint, a full
  template/build render, `actionlint` for workflows) before calling it done. Green CI is necessary,
  not sufficient.
- One concern per branch/PR, even tiny ones — it keeps reviews and the drafted changelog clean.
- When PRs interact, state an explicit merge ORDER rather than opening them and walking away:
  anything a *tag* triggers needs its inputs on `main` first; a new *gate* (check) needs the
  violations it catches fixed first; a workflow that builds from committed content needs that
  content merged first.
- Semver framing: `breaking` only means breaking against a *released* version; removing something
  that was never shipped is not a breaking change.
- Cross-repo reconciliation goes through a neutral drop-zone outside both repos — never a repo
  inside a repo, and no accidental gitlinks/submodules.

## Workflow

Issue (from a template; free-form issues are disabled) -> for sequential / multi-step work, a parent
issue with ordered **sub-issues** -> put it on the active **milestone** -> branch
`<type>/<issue#>-<slug>` -> code (comments cite the issue) -> PR with a Conventional Commit title
(the autolabeler sets the category label from the title), the template body, and a **closing
summary** before merge -> **squash** merge. The operator (@Bugs5382) is the assignee.

On merge, release-drafter drafts the next notes by label and `CHANGELOG.md` updates on `main` via the
changelog action -- **nothing tags automatically**. When the first push to main resolves the version,
rename the milestone to that version. The maintainer then **manually publishes the GitHub Release**,
which creates the tag with the finalized changelog (and triggers the publish where the repo ships a
package).

Keep public artifacts (issues, PRs, commit messages) free of references to local-only design notes.

## Releasing

On every push to `main` the **Release Manager** workflow (`.github/workflows/job-version-bump.yaml`)
runs: release-drafter anticipates the next version, the manifest version is bumped (package
ecosystems only), `CHANGELOG.md` is updated via the changelog action, and a
`chore(pre-release): vX [skip ci]` commit is pushed back to `main`. **Nothing tags automatically.**
The maintainer then publishes the GitHub Release by hand, which creates the `vX.Y.Z` tag with the
finalized changelog and triggers the publish workflow where the repo ships a package.

The Release Manager pushes to `main` through the release GitHub App, which the branch ruleset
(`governance/rulesets/branch-default.json`) lists as a bypass actor — without that bypass the
`[skip ci]` commit would be rejected by the PR-required rule.

**GitHub Action repos release differently** — a composite/JS action has no package manifest and is
**not** published to a registry. The maintainer manually tags `vX.Y.Z`, publishes the GitHub
Release, and repoints the floating major tag `@vN` (e.g. `git tag -fa v1 -m "v1 -> v1.2.3" v1.2.3 &&
git push origin v1 --force`) so consumers pinned to `owner/repo@vN` pick up the release; publishing
to the GitHub Marketplace is an optional manual step. The **first release is `v1.0.0` by hand** —
release-drafter would otherwise draft `v0.1.0` on the first run. See `ecosystems/action/README.md`
for the full action-release sequence.

### Docs site and first-release gotchas

- A tag-gated GitHub Pages deploy (publishing only on `vX.Y.Z`) fails at the Deploy step unless the
  auto-created `github-pages` environment allows tag refs. Once, alongside enabling Pages
  (Settings -> Pages -> Source = GitHub Actions), add a tag policy, then re-run the failed Deploy
  job (no need to re-cut the tag):
  `gh api -X POST repos/the-rabbit-hole-tech/rabbit-docs-theme/environments/github-pages/deployment-branch-policies -f name='v*' -f type=tag`
- Docusaurus MDX 3: avoid the `## Heading {#custom-id}` explicit-id syntax (it fails to compile);
  rely on the auto-generated slugs.

## Local preview

`preview/` is a gitignored throwaway Docusaurus site that consumes this theme, for
eyeballing changes locally. It is never committed.

- Run it: `cd preview && npm start -- --port 3001`
- The theme is installed as a packed copy, so after editing the theme, refresh that
  copy: `npm run build && npm pack --pack-destination preview/ && (cd preview && npm install ./the-rabbit-hole-tech-docs-theme-*.tgz)`
- If `preview/` is missing, recreate it: `npx create-docusaurus@latest preview classic --typescript`, then apply the consumption recipe from `README.md` (plugin + `customCss` + `recommendedThemeConfig`).
