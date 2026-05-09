---
name: git-workflow-best-practices
description: Git workflow best practices — branch management with `feature/` / `bugfix/` / `hotfix/` / `release/` prefixes (Vincent Driessen's Git Flow), Conventional Commits format (`type(scope): subject` with `feat` / `fix` / `docs` / `style` / `refactor` / `test` / `chore` types), atomic commits with imperative mood, pre-commit hygiene (run tests, review diff, no debug code, no secrets, no `git add .` without per-file review), remote workflow (`git pull --rebase` before push, no force push to shared branches, resolve conflicts locally, push only feature-complete code), and code review process (PR creation required, max ~400 lines per PR, self-review first, squash on merge for clean main history). Use this skill whenever the user is about to commit, push, create a branch, open or merge a pull request, write a commit message, or asks about git workflow conventions — including staging files, splitting commits, choosing a branch name, formatting a commit message, deciding whether to rebase or merge, sizing a PR, or handling conflicts. Trigger on `git commit`, `git push`, `git checkout -b`, `git rebase`, `gh pr create`, `git merge`, `git stash`, or when the user shows uncommitted/staged changes via `git status` / `git diff` and is preparing to commit. Do NOT use for pure read-only git history exploration (`git log`, `git show`, `git blame` for context), Mercurial, SVN, Perforce, or generic version-control conceptual questions unrelated to a real workflow decision.
---

# Git Workflow Best Practices

This skill captures workflow and style conventions for git work. **It does NOT cover safety rules** — those are in Claude Code's built-in system prompt and always apply (never `--force` on `main`/`master`, never bypass hooks with `--no-verify` unless explicitly requested, never amend after a failed pre-commit hook, never commit unless asked, prefer specific file paths over `git add -A`/`git add .`). This skill adds the *style* layer on top.

Two-line summary the skill enforces: **commits are small, named, intentional**; **branches and PRs are scoped to one purpose, reviewed before merge, and leave a clean history on `main`.**

## Branch Management

1. **NEVER commit directly to `main` / `master`.** Always create a feature branch first. The system prompt's safety rules forbid force-push to main; this rule prevents the situation where you'd want to.
2. **Branch naming uses a typed prefix**:
   - `feature/<short-slug>` — new functionality
   - `bugfix/<short-slug>` — non-critical bug fixes
   - `hotfix/<short-slug>` — urgent production fixes
   - `release/<version>` — release preparation
   The prefix tells anyone glancing at `git branch` what the branch is *for* without reading the slug.
3. **Branching topology** (Git Flow):
   - `feature/*` and `bugfix/*` branch from `develop`, merge back to `develop`.
   - `hotfix/*` branches from `main`, merges to both `main` AND `develop` (so `develop` doesn't lose the fix).
   - `release/*` branches from `develop`, merges to `main` (and back to `develop` if any release fixes happened).
   Projects without a `develop` branch (trunk-based) treat `main` as the integration target — feature branches still merge through PRs, just into `main`.
4. **One branch per task.** Never mix two features or a feature + an unrelated cleanup in one branch — it makes review and revert impossible to scope.

## Commit Practices

5. **Atomic commits**: one logical change per commit. If you can describe the commit with "and" between two clauses, split it.
   - **Why:** atomic commits make `git revert` surgical (revert one bug fix without losing the unrelated refactor that came in the same session), `git bisect` precise (the failing commit pinpoints one change), and code review readable (reviewer reads one intent at a time, not five interleaved).
6. **Imperative mood in subject**: "Add JWT validation" not "Added JWT validation" or "Adds JWT validation". Read it as completing the sentence "If applied, this commit will ___".
7. **Conventional Commits format**: `type(scope): subject`
   - **Type**: `feat` (new feature), `fix` (bug fix), `docs` (documentation), `style` (formatting, no logic change), `refactor` (code restructure, no behavior change), `test` (adding/fixing tests), `chore` (build, deps, tooling).
   - **Scope** (optional but recommended): the area touched — `feat(auth): add JWT validation`, `fix(api): handle null user in /me`.
   - **Subject**: ≤72 chars, imperative, no trailing period.
   - **Why:** machine-parsable. Tools like `standard-version`, `semantic-release`, `git-cliff` derive changelogs and version bumps automatically from these prefixes. Even without automation, scanning `git log --oneline` for `feat:` vs `fix:` is faster than reading prose.
8. **Body explains the why, not the what.** The diff already shows what changed. The body is for: motivation (which ticket, what user pain), tradeoffs considered, follow-up work needed.

## Before Committing

9. **Run tests before committing.** Broken code in history makes `git bisect` lie. Even if CI will catch it, a green local run prevents wasted PR cycles.
10. **Review `git diff --cached`** (the staged set) before running `git commit`. Catches: forgotten debug prints, accidentally-included unrelated files, secrets, leftover TODO comments meant for another commit.
11. **Lint and format** before staging. The repo's pre-commit hooks should enforce this; running them locally first avoids the "fix lint" follow-up commit.
12. **Strip debug code**: `print()`, `console.log()`, `debugger`, `breakpoint()`, commented-out experiments, dead code from refactors. If it's intentional logging, route it through the project's logger; if it's debugging, remove it.
13. **Never commit secrets**: API keys, passwords, tokens, private certs, `.env` files with real credentials. Use a secrets manager + `.env.example` placeholder. If a secret slips in, treat it as compromised — rotate it, then `git filter-repo` to scrub history (and force-push with explicit team coordination).

## Working with Remotes

14. **Pull with rebase** before pushing: `git pull --rebase origin <branch>`. Or set `git config pull.rebase true` once.
    - **Why:** keeps your feature branch as a linear sequence of your commits on top of the latest base, instead of a confusing merge bubble. PR review reads top-to-bottom; merge bubbles invert that.
15. **Push only feature-complete, tested code.** Half-broken pushes are a signal to teammates that something's ready when it isn't. Use `git stash` or commit-and-amend-locally for in-progress work.
16. **Force push only to your own private branch.** Even there, prefer `--force-with-lease` (which fails if someone else pushed in the meantime) over `--force`. Never force-push to `main`, `develop`, `release/*`, or any branch shared with others.
    - **Why:** `--force` overwrites the remote's history with yours. Anyone with commits on top of what you overwrote loses them silently. The system prompt's safety rule already blocks force-push to main; this rule extends "shared" to release/develop branches.
17. **Update from base daily** during multi-day work. Long-running branches diverge from `develop`/`main`, and the rebase to catch up gets harder the longer you wait. Daily `git pull --rebase origin develop` keeps conflicts small.
18. **Resolve conflicts locally before pushing.** Never push a branch with `<<<<<<< HEAD` markers; PRs in that state confuse reviewers and break CI. If a rebase conflict is too large to resolve, ask for help before continuing.

## Pull Request Process

19. **Always create a PR/MR for review.** Even solo projects benefit — the PR description forces you to articulate intent, and the diff view catches things you miss in your own editor.
20. **Keep PRs small — target ≤400 lines changed.**
    - **Why:** code review effectiveness drops sharply past ~400 lines (research from SmartBear's review studies and others). Reviewers stop reading carefully and start rubber-stamping. Split big changes into a stack of small PRs (refactor → behavior change → tests, in separate PRs that reviewers merge in order).
21. **Self-review first.** Read your own PR diff in the GitHub/GitLab UI — *not* in your editor — before requesting review. The web view exposes things the editor hides: too-large hunks, leftover whitespace changes, unrelated file modifications.
22. **Address every review comment.** Either fix it, or reply explaining why you're not. Silent dismissal is the fastest way to lose reviewer goodwill on the next PR.
23. **Squash on merge** for feature branches.
    - **Why:** keeps `main`'s history one commit per feature, readable in `git log --oneline`. The branch's intermediate "wip", "fix typo", "address review" commits don't pollute the canonical history. Long-running branches with intentionally-curated commits (e.g., a refactor done in 5 atomic steps) can use rebase-merge instead — but the default for typical feature branches is squash.
24. **Use draft PRs** for in-progress work that you want CI to run on but don't want reviewers to read yet. Promote to "ready for review" when self-review is done.

## When applying these rules

- **Be opinionated about footguns** (#5 atomic commits, #7 conventional format, #14 pull-rebase, #16 force-push scope, #20 PR size, #23 squash on merge) — these prevent muddled history, lost commits, broken bisect, and rubber-stamp reviews. Not preferences.
- **Be flexible about branching topology** (#3) — Git Flow assumes `develop` + `main` separation. Trunk-based workflows merge directly to `main`; feature branches are still scoped, just shorter-lived. Match the project's convention; don't impose Git Flow on a trunk-based repo.
- **Read existing convention first.** If the project has a `CONTRIBUTING.md`, a `.github/pull_request_template.md`, or visibly enforced commit format in `git log`, conform to it rather than introducing this skill's defaults wholesale.
- **System-prompt safety rules apply unchanged.** This skill adds style on top; it never relaxes the safety layer. If a rule here seems to conflict with Claude Code's built-in git rules, the built-in rules win.
- **Cross-skill awareness:** when the change being committed touches code, the relevant code-quality skills (`pytest-best-practices`, `sqlalchemy-best-practices`, etc.) handle the *what changed*; this skill handles the *how it lands in git*.
