Secrets & Dependency Management
================================

> Two of the most common ways a project gets breached have nothing to do with a clever exploit: a credential sitting in git history, or a known vulnerability in a dependency nobody updated.

* * *

## Secrets never go in git

This is covered in more depth in [Coding Standards → General Principles → Secrets and configuration](/Coding-Standards/General-Principles#secrets-and-configuration), but the short version applies here too. API keys, database passwords, signing keys, and client credentials belong in environment variables or a secrets manager, never in the repo. Commit a `.env.example` with dummy values so people can see what config the app needs, and never commit the real `.env`.

If a secret does get committed, rotate it. Deleting the line afterward doesn't remove it from git history. Anything shipped in a client-side bundle, web or mobile, should be treated as public, no matter how it's obfuscated.

"Rotate it" means revoke the old one too, not just add a new one. Swapping the key in `.env` while the old key is still valid at the provider isn't a fix — it just means there are now two working keys instead of one.

A scanner catches what a code reviewer skimming a diff won't. Run it before a repo goes public or before a client audit, and set it up as a pre-commit hook if you want it to stop a secret before it ever reaches history rather than finding it after:

```bash
gitleaks detect --source . -v
```

* * *

## Keep dependencies patched

Every dependency is code you're responsible for even though you didn't write it. Left unpatched, it's a vulnerability with your project's name on it.

Run the ecosystem's audit tool regularly, not just when something breaks — `npm audit`, `composer audit`, `pip-audit`, `dotnet list package --vulnerable`. Where the platform supports it, turn on automated update tooling like Dependabot or Renovate, so patches show up as PRs instead of relying on someone remembering to check. A dependency upgrade for a known CVE should be its own priority, not something that waits for the next feature sprint, especially for anything internet-facing. Before adding a dependency in the first place, run through the checklist in [General Principles → Adding a dependency](/Coding-Standards/General-Principles#adding-a-dependency).

A vulnerability audit only flags known CVEs — it says nothing about a package that's simply three years behind but has no reported issues yet. Worth running `npm outdated` / `composer outdated` (or the equivalent for your stack) on its own occasionally, since "nothing's flagged" and "nothing needs updating" aren't the same thing.

* * *

## Resources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP Top 10 — A06: Vulnerable and Outdated Components](https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/)
- [GitHub Dependabot documentation](https://docs.github.com/en/code-security/dependabot) — automated dependency update PRs and security alerts.
- [Gitleaks](https://github.com/gitleaks/gitleaks) — a free scanner for secrets accidentally committed to a repo. Good to run before a repo goes public or a client audit happens.

* * *

## Related pages

- [Coding Standards → General Principles](/Coding-Standards/General-Principles) — secrets handling and dependency vetting from the day-to-day coding-standards angle.
- [Authentication & Access Control](/Security/Authentication-and-Access-Control) — API keys and service accounts follow the same least-privilege rule as user accounts.
