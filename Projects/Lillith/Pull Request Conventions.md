---

sidebar_position: 1

---

# Pull Requests

  

### 1. Branch Naming Conventions

  

The branch name **must** include the GitHub ticket number and a short, descriptive title.

  

- Format `${feat|bug|chore|docs|refactor|test}/${GITHUB_TICKET_NUMBER}-description`

- Example: `feat/123-add-login-flow`

- Avoid overly long or vague branch names.

- **Prefix must align with commit types (see section 4).**

  

### 2. PR Title

  

The PR title is linted and enforced to be in the Conventional Commits standard.

The only thing that is not enforced is the ticket number. Make sure to include it, so that our changelog will be with appropriate linking.

  

- Format: `${type}(${scope}): ${subject} #${TICKET-NUMBER}`

- Example: feat(auth): add login feature to frontend #123

- **Align title with commit purpose** (feat, fix, chore, etc.).

  

- **Each PR must follow the** [**Conventional Commits**](https://www.conventionalcommits.org/en/v1.0.0/) **standard** to ensure auto-generated Release Notes.

- **Format:**

  

| **Type**            | **Example PR Title**                                 | **When to use**                                                |

| :------------------ | :--------------------------------------------------- | :------------------------------------------------------------- |

| feat                | feat(auth): add login functionality with JWT support | Adding new features                                            |

| fix                 | fix(storage): handle null pointer error              | Bug fixes                                                      |

| chore               | chore: update dependencies                           | Maintenance tasks, build updates                               |

| docs                | docs: update README for new setup instructions       | Documentation only                                             |

| refactor            | refactor(video): improve stream handling             | Code changes without affecting behaviour                       |

| test                | test(api): add unit tests for auth service           | Adding or updating tests                                       |

| **breaking change** | feat(api)!: migrate to new API.                      | Changes that break backward compatibility (major version bump) |

  

**⚠️ Important Rules:**

  

- Use **imperative mood** (e.g., Add, Fix, Change — not Added, Fixed).

- **One logical change per PR** — avoid mixed PRs.

- If the PR introduces breaking changes, **add footer**:

  `BREAKING CHANGE: description of the breaking change`

  

More example area available here:

  

- [**Conventional Commits**](https://www.conventionalcommits.org/en/v1.0.0/)

- [**conventional commits a better way**](https://medium.com/neudesic-innovation/conventional-commits-a-better-way-78d6785c2e08)

  

### 3. Ticket Reference

  

Every PR **must reference the related GitHub ticket** in both **title and description**.

We mentioned above what is the format for the PR title, but in the PR description, which follows this [Pull Request Template](https://github.com/ClickIDF/1337-jupiter-monorepo/blob/dev/.github/pull_request_template.md), we need to link the issue in this line:

  

```diff

<!-- replace the issue number with the ... -->

  

- Please link to the issue: Closes #...

+ Please link to the issue: Closes #123

```

  

This will link the PR to the issue and close the issue automatically when the PR is merged.

  

If this PR does not have a linked issue (which should not be the case for day-to-day PRs), set above line to be the following:

  

```diff

<!-- replace the Closes #... with Null -->

  

- Please link to the issue: Closes #...

+ Please link to the issue: Null

```

  

### 4. Commit Messages (Conventional Commits Standard)

  

Commits are no longer enforced in feature branches, but for CHANGELOG creation reasons, they are enforced when merging `dev` to `main`.

This is why we merge PRs to branch `dev` using squash, and merge to `main` using merge commits.

  

### Pre-commit Best Practices

  

Before committing your changes, especially when working with TypeScript code, it's essential to run type checking to ensure a clean deployment:

  

```bash

npm run lint                    # Lint code

npm run format                  # Format code

npm run typecheck

```

  

**Why this matters:**

  

- Prevents TypeScript compilation errors from reaching production

- Ensures type safety across the entire codebase

- Avoids CI/CD pipeline failures due to type errors

- Maintains code quality standards

  

**When to run:**

  

- Before every commit (especially for TypeScript changes)

- After making significant changes to type definitions

- Before creating a pull request

- When working with feature flags or API integrations

  

**Pro tip:** If you encounter TypeScript errors during development, fix all of them before committing. A clean `npm run typecheck` ensures your changes won't break the deployment pipeline.

  

## 5. Reviewers

  

- Each PR **must be reviewed by a peer** before merging:

- **Sadir reviews Miluim**, and **Miluim reviews Sadir** — depending on PR creator.

- For changes to DevOps files (Github Workflows, Terraform, and deployment scripts), the PR must be reviewed by a member of the Jupiter DevOps code owners team.

  

## 6. Example of a Full Pull Request Flow

  

| **Step**        | **Example**                                             |

| :-------------- | :------------------------------------------------------ |

| **Branch name** | feat/123-add-login-flow                                 |

| **PR Title**    | feat(auth): add login feature to frontend #123          |

| **PR Body**     | Closes #123 This PR adds a login flow with JWT support. |

  

## 7. Summary of PR Types for Release Please

  

| **PR Type** | **Description**                              | **Appears in Release Notes as...** |

| :---------- | :------------------------------------------- | :--------------------------------- |

| feat        | New feature                                  | **Features**                       |

| fix         | Bug fix                                      | **Bug Fixes**                      |

| chore       | Maintenance, CI, build                       | _Not included by default_          |

| docs        | Documentation updates                        | _Not included by default_          |

| refactor    | Code refactor without behavior change        | _Not included by default_          |

| test        | Adding or updating tests                     | _Not included by default_          |

| Ci          | for CI/CD pipeline changes                   | _Not included by default_          |

| perf        | for performance improvements                 | _Not included by default_          |

| Build       | for build system changes                     | _Not included by default_          |

| revert      | for reverting a commit due to issues or bugs |                                    |

| wip         | work in progress, not finished yet           |                                    |

| hotfix      | for critical fix for production environment  |                                    |