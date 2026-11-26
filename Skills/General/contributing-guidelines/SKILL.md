---
name: contributing-guidelines
description: General contribution workflow for any repository - Conventional Commits for commit messages and PR titles, small focused PRs (about 400 lines max), an issue before a PR, tests and docs with every change, green CI before review, and a lightweight RFC for major changes. Use whenever writing a commit message, naming or describing a PR, opening or splitting a PR, or deciding whether a change needs an issue or an RFC. Trigger on phrases like "commit", "push", "open a PR", "conventional commits", "contributing guidelines", "follow CONTRIBUTING.md", "split this PR", or "write an RFC".
---

# Contributing Guidelines

A pragmatic workflow that keeps reviews fast: agree on the approach early, keep changes small, and make every commit and PR self-explanatory.

## Precedence

1. The user's explicit instructions.
2. The repository's own `CONTRIBUTING.md`, PR template, or `CLAUDE.md`.
3. This skill.

Use this skill to fill the gaps, never to override the first two.

---

## 1. Issue Before PR

Except for critical hotfixes, a change should have an open issue where the approach was agreed before code was written. This avoids building something that conflicts with the project's direction.

- Before opening a PR, check whether there is a related issue and link it in the description.
- If there is none, tell the user and offer to draft one. Don't open issues without being asked.

---

## 2. Small, Focused PRs

- One logical change per PR.
- Aim for under ~400 changed lines. Generated files, lock files and schema snapshots don't count.
- If a feature is larger, split it into incremental PRs that each build and pass tests on their own, and propose the split to the user before opening them.
- Inside a PR, prefer one commit per logical step over one giant commit.

---

## 3. Conventional Commits

Use the [Conventional Commits](https://www.conventionalcommits.org/) format for **commit messages and PR titles** (squash merges use the PR title as the commit message).

**Format:** `<type>(<scope>): <description>`

| Type       | Use for                                                     |
| ---------- | ----------------------------------------------------------- |
| `feat`     | A new feature                                               |
| `fix`      | A bug fix                                                   |
| `docs`     | Documentation only                                          |
| `style`    | Formatting, whitespace, missing semicolons (no logic change) |
| `refactor` | Code change that neither fixes a bug nor adds a feature     |
| `test`     | Adding or updating tests                                    |
| `chore`    | Routine tasks, dependency bumps, tooling                    |
| `build`    | Build system or external dependencies                       |
| `ci`       | CI configuration and scripts                                |
| `perf`     | Performance improvements                                    |
| `revert`   | Reverting a previous commit                                 |

**Rules**
- `scope` is optional: the module, feature or area touched (`auth`, `api`, `deps`).
- `description` is imperative, lowercase, with no trailing period, and short enough that the whole first line stays under ~72 characters ("add login screen", not "Added login screen.").
- Use the body for the *why* when it isn't obvious from the title.

**Breaking changes:** append `!` after the type/scope, or add a `BREAKING CHANGE:` footer explaining what breaks and how to migrate. Any type can be breaking.

**Examples**
- `feat(auth): add login screen`
- `fix(api): return 404 when ride is missing`
- `docs: correct spelling of CHANGELOG`
- `feat(api)!: send an email to the customer when a product is shipped`
- `feat!: drop support for Node 6` with footer `BREAKING CHANGE: use JavaScript features not available in Node 6.`

---

## 4. Leave It Better Than You Found It

- Include tests for the change.
- Update the README or relevant docs when behavior or an API changes.
- Make sure CI is green before asking for review. Run the project's build, lint and tests locally first when possible.

---

## 5. RFC for Major Changes

A regular issue is enough for normal features and bug fixes. Use an RFC (Request for Comments) for a new architectural pattern, a major dependency swap, or a cross-cutting capability:

1. Open an issue proposing the idea.
2. Once the problem is agreed to be worth solving, copy the repo's RFC template (often `docs/rfcs/0000-template.md`). If there is none, use: Summary, Motivation, Detailed design, Alternatives considered, Risks and migration, Open questions.
3. Name it with the next number and a slug, e.g. `docs/rfcs/0001-new-caching-strategy.md`.
4. Open a PR containing only the RFC document, and refine it in review.
5. Start implementing only after the RFC PR is merged.

---

## PR Description Checklist

- [ ] Title follows Conventional Commits
- [ ] Linked issue (or the reason there is none, e.g. hotfix)
- [ ] What changed and why
- [ ] How it was tested
- [ ] Tests included
- [ ] Docs updated if behavior or an API changed
- [ ] Breaking changes called out with migration notes
- [ ] CI green
