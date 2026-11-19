Authentication & Access Control
================================

Multi-Factor Authentication (MFA) means proving who you are with two or more independent factors. A stolen password alone isn't enough to get in.

* * *

## MFA is a default

Every project we build, internal or client-facing, ships with MFA on user authentication unless a client's written spec says otherwise. If it does get excluded, document why and agree on something to cover the gap — shorter session lifetimes, a stricter lockout policy, IP restrictions.

**Approved methods** (use at least one):

| Method | Notes |
| --- | --- |
| Authenticator app (TOTP) | Google Authenticator, Microsoft Authenticator, or any standard TOTP app. Preferred — works offline, no SMS/email dependency. |
| Email one-time code | Fine when an authenticator app isn't practical for the user base. |

SMS-based MFA isn't on the approved list. It's vulnerable to SIM-swapping, so it shouldn't be the only option for anything security-sensitive.

If you're implementing TOTP yourself rather than using a provider that already has it built in (Auth0, Cognito, Firebase Auth, Laravel Fortify), a well-maintained library does the actual code generation and verification correctly — `otplib` for Node, `pyotp` for Python, `pragmarx/google2fa` for Laravel. Writing the TOTP algorithm from scratch is a good way to get the clock-drift handling subtly wrong.

* * *

## Least privilege

MFA answers "is this really the user?" Least privilege answers the next question: what should they be allowed to do once they're in?

Give accounts and API keys the minimum access their job needs, not admin because it's easier to set up. Check periodically who has admin or owner access on cloud consoles, CI/CD, and production databases — access tends to build up over time and rarely gets cleaned up on its own. When someone leaves the project or the company, revoke their access immediately, not next sprint.

**A quick check worth running yourself:** log in as a low-privilege user and try hitting an admin-only route directly with its URL, not through a link the UI never shows you.

```bash
curl -i https://staging.example.com/api/admin/users \
  -H "Authorization: Bearer <a-regular-users-token>"
```

If that returns real data instead of a 403, the access control lives in the frontend and not the API — which means it isn't access control, it's just a hidden button.

* * *

## Password policy

Favor a longer minimum length (12+ characters) over strict composition rules. Length resists cracking better than forcing a symbol and a number into every password. Never store passwords in plain text or with reversible encryption — hash them with bcrypt or argon2. Rate-limit or lock out accounts after repeated failed logins, and actually test that lockout yourself: run a handful of wrong passwords against a test account in a row and confirm the account throttles rather than quietly accepting a hundredth guess.

* * *

## Resources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [OWASP Access Control Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html)
- [NIST SP 800-63B — Digital Identity Guidelines: Authentication](https://pages.nist.gov/800-63-3/sp800-63b.html) — the source most "length over complexity" password guidance comes from.
- [PortSwigger Web Security Academy — Access control vulnerabilities](https://portswigger.net/web-security/access-control) — interactive labs for the "wrong user's data" bug least privilege is meant to prevent.

* * *

## Related pages

- [Network & Environment Protection](/Security/Network-and-Environment-Protection) — restricting where logins can come from, on top of how they authenticate.
- [Secrets & Dependency Management](/Security/Secrets-and-Dependency-Management) — API keys and service credentials follow the same least-privilege rule.
- [Self-Testing & Verification](/Security/Self-Testing-and-Verification) — how to verify your access controls actually block what they're supposed to.
