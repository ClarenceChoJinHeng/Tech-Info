Commits and Naming
==================

> How to write commit messages, and how to name branches and tags. Short page, but it's the one people will notice when you get it wrong.

* * *

## 1. Commit messages

A commit message is a note to the next person who reads the history. Six months from now that person is probably you, trying to work out why a line exists.

### The format

We encourage **[Conventional Commits](https://www.conventionalcommits.org/)**. It isn't enforced, but please use it. It makes the history easy to scan and it's what most of the industry uses.

```
<type>(<optional scope>): <what this commit does>

<optional body: why you made the change>
```

```
fix(booking): stop double booking on the same time slot

Two requests arriving at the same moment could both pass the
availability check. The slot is now locked while the booking saves.
```

Write the message in your editor with `git commit`, not on the command line with `git commit -m "..."`. The `-m` flag makes it awkward to add a body, so messages end up as one line with no *why*. See [Git Basics](/Git-Guide/Git-Basics#settings-we-recommend) to set which editor Git opens.

### Types

| Type | Use it when | Example |
| --- | --- | --- |
| `feat` | Adding something new for the user | `feat: add SMS appointment reminders` |
| `fix` | Fixing a bug | `fix: show correct clinic name in reminder SMS` |
| `refactor` | Changing code without changing what it does | `refactor: move pricing logic into PricingService` |
| `style` | Formatting only, no logic change | `style: run prettier on booking module` |
| `test` | Adding or fixing tests | `test: cover past-date validation` |
| `docs` | Documentation, README, comments | `docs: add local setup steps to README` |
| `chore` | Dependencies, config, housekeeping | `chore: upgrade axios to 1.7` |
| `perf` | Making something faster | `perf: cache clinic list for 5 minutes` |
| `ci` | Pipeline and build config | `ci: run tests on pull requests` |

The scope in brackets is optional. Use it when the project has clear areas, like `(auth)`, `(booking)`, `(payments)`.

If a change breaks something for whoever uses it (an API field renamed, an endpoint removed), add `!` after the type: `feat(api)!: rename patientId to customerId`.

### The rules

- **Keep the first line under about 50 characters.** Platforms cut off long lines in lists and history views.
- **Say what the commit does**, as if you're finishing the sentence "This commit will...". So `add`, `fix`, `remove`, not `added`, `fixing`, `removes`.
- **Say what and where, not just which file.** The diff already shows the file.
- **Add a body when the reason isn't obvious.** Leave a blank line after the first line, then explain *why*.
- **One logical change per commit.** Don't mix a bug fix with reformatting half the file. Reviewers can't see the fix among 300 lines of whitespace changes.

### Good and bad

| ❌ Bad | Why | ✅ Good |
| --- | --- | --- |
| `update` | Update what? | `fix: prevent booking dates in the past` |
| `Update SCSS file` | Says the file, not the purpose | `fix: header overlapping menu on mobile home page` |
| `fixed bug` | Which bug? | `fix: reminder SMS sent twice for same appointment` |
| `wip` | Fine on your own branch, not what should land in `main` | Tidy up before the PR, see [Advanced Git](/Git-Guide/Advanced-Git) |
| `feat: add reminders, fix login, update packages` | Three changes in one commit | Three separate commits |
| `Changes as per comments` | Means nothing in history | `fix: validate phone number format before sending SMS` |

* * *

## 2. Branch names

```
<type>/<short-description>
```

| Type | Branched from | Example |
| --- | --- | --- |
| `feature/` | `main` | `feature/sms-reminders` |
| `bugfix/` | `release/staging` or `release/uat` | `bugfix/past-date-booking` |
| `hotfix/` | `release/production` | `hotfix/payment-timeout` |

The four long-lived branches have fixed names that never change: `main`, `release/staging`, `release/uat`, `release/production`.

### The rules

- **Lowercase only.** Some operating systems treat `Feature/Login` and `feature/login` as the same thing and some don't, which causes strange problems.
- **Hyphens between words.** No spaces, underscores or camelCase.
- **Keep it short.** Around 3 or 4 words after the slash, under 30 characters in total if you can.
- **Describe the work, not the person.** The branch outlives your involvement.

| ❌ Bad | Why | ✅ Good |
| --- | --- | --- |
| `Feature/SMS_Reminders` | Capitals and underscores | `feature/sms-reminders` |
| `ali-branch` | Says who, not what | `feature/clinic-search` |
| `fix` | No type prefix, no description | `bugfix/wrong-clinic-name` |
| `feature/add-the-new-sms-reminder-feature-for-clinics` | Far too long | `feature/sms-reminders` |
| `test123` | Means nothing to anyone else | `feature/try-new-calendar-lib` |

Branches cost nothing. If you're not sure whether something needs its own branch, create one. You can always merge it or delete it later.

* * *

## 3. Tag names

We tag every promotion into `release/uat` and `release/production`, using the date:

```
v<year>.<month>.<day>-<environment>
```

| Tag | Meaning |
| --- | --- |
| `v2026.10.25-uat` | The build sent to client UAT on 25 October 2026 |
| `v2026.10.25-prod` | That same build, now live (even if it went live a few days later) |
| `v2026.10.25.2-uat` | A second UAT build on the same day |
| `v2026.10.29-prod` | A hotfix that went live on 29 October |

### The rules

- **Year, month, day, in that order**, with two-digit month and day (`v2026.03.07`, not `v2026.3.7`). This makes tags sort correctly in every tool, which day-first dates don't.
- **UAT sets the version.** When the client approves a UAT build, production gets the same version with `-prod`. That's how you can tell which UAT build is live.
- **A new UAT build gets a new date.** If you fix a bug in UAT and send a new build, tag it with today's date.
- **Hotfixes get the date of the hotfix**, with `-prod`.
- **Always use annotated tags** (`git tag -a`), with a message saying what's in the build.
- **Never move or delete a tag once it's pushed.** If you tagged the wrong commit, tell your lead instead of fixing it quietly.

```bash
git tag -a v2026.10.25-uat -m "Sprint 14: SMS reminders, clinic search"
git push origin v2026.10.25-uat
```

This style is called calendar versioning, or CalVer. Ubuntu (`24.04`) and JetBrains (`2024.2`) use it too. It works well for us because we ship client projects on a schedule, not libraries that other people install by version number.

* * *

## Further reading

1.  **Conventional Commits**
    [https://www.conventionalcommits.org/](https://www.conventionalcommits.org/)
2.  **How to Write a Git Commit Message**, Chris Beams
    [https://cbea.ms/git-commit/](https://cbea.ms/git-commit/)
3.  **Calendar Versioning**
    [https://calver.org/](https://calver.org/)
