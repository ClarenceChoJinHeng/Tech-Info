Self-Testing & Verification
============================

> Code review only catches what the reviewer thinks to look for. Before it gets there, try breaking your own code the way an attacker would use it. Most security bugs are things nobody tried, not things someone got wrong on purpose.

* * *

## Think like an attacker, briefly

For any input field, URL parameter, or API payload you just built, ask what happens if it isn't what you expect. Then try it before shipping:

| Try this | Looking for |
| --- | --- |
| `' OR '1'='1`, `'; DROP TABLE users;--` in text fields | Does it error out, or does it change how the query behaves? See [Secure Coding](/Security/Secure-Coding). |
| `<script>alert(1)</script>` in anything that gets displayed later | Does it show up as text, or does it execute? |
| Someone else's ID in a URL you own (`/orders/1042` → `/orders/1043`) | Can you see another user's data? That's an access-control bug (IDOR), not a query bug — the query worked exactly as written, just for the wrong person. |
| Hitting an API endpoint with no auth token, or an expired one | Does it reject you, or does it quietly hand back data anyway? |
| Very long input, empty input, the wrong data type | Does it crash with a stack trace, or fail gracefully? |

None of this needs special tooling. Your browser's devtools, or a REST client like Postman/Insomnia, or plain `curl` from a terminal, are enough to run through all of it against your own feature in a few minutes.

### Quick recipes

A couple of the table's rows as actual commands, so there's no translation step:

```bash
# No auth token at all — should be rejected, not quietly served
curl -i https://staging.example.com/api/orders/999

# Someone else's ID — swap the token for a different (real) user's
# and confirm you get 403/404, not their data
curl -i https://staging.example.com/api/orders/999 \
  -H "Authorization: Bearer <a-different-users-token>"
```

`-i` prints the response headers along with the body, so you can see the actual status code rather than guessing from what rendered on screen.

* * *

## Write the negative test, not just the happy path

A test suite that only proves the feature works for valid input hasn't proven it's safe. Anything touching auth or user-supplied data should have a test for the case that's supposed to fail.

```ts
// ✅ Proves unauthorized access is actually rejected, not just untested
it('returns 403 when requesting another user\'s order', async () => {
  const res = await request(app)
    .get('/orders/999')
    .set('Authorization', `Bearer ${otherUsersToken}`);

  expect(res.status).toBe(403);
});
```

One test per access-control boundary — wrong user, wrong role, no token, expired token — pays for itself the first time a refactor removes a check nobody noticed was load-bearing. The same idea holds for input validation: test that bad input gets rejected, not just that good input gets through.

* * *

## Let tools do the repetitive checking

Most language linters have a security-focused ruleset — ESLint security plugins, Bandit for Python, PHPStan/Psalm security rules, .NET analyzers. They catch known-bad patterns automatically, so code review doesn't have to spot them by eye every time. A tool like Semgrep does something similar across languages from one config, which is worth it if a project mixes stacks. Run a dependency audit too (`npm audit`, `composer audit`, `pip-audit`); see [Secrets & Dependency Management](/Security/Secrets-and-Dependency-Management). Check the response headers and network tab — confirm HTTPS is enforced, no secrets are leaking into API responses or client-side bundles, and errors aren't returning stack traces; see [Server & Infrastructure Hardening](/Security/Server-and-Infrastructure-Hardening).

Tools catch known patterns. They don't replace actually trying to break your own feature. Use both.

**A few habits worth keeping, regardless of which tool you reach for:**

- Test against staging or a local copy, never production. An IDOR test that actually reads someone real's data is no longer a test.
- Use a second, throwaway test account instead of your own — it's the only way to honestly check "can user A see user B's data."
- Keep a personal scratch file of payloads and commands you reuse often. You'll run the same handful of checks on every feature; there's no need to reconstruct them from memory each time.
- A finding that reproduces once and disappears on retry still means something. Don't wave it off as a fluke — log it and come back to it.

* * *

## Before opening a PR

A quick pass over what you built, focused on the security-relevant parts:

- [ ] Tried at least one malicious input on every new user-input field.
- [ ] Confirmed auth/role checks actually block access, not just that the UI hides the button.
- [ ] No secrets, tokens, or internal URLs hardcoded or logged.
- [ ] Errors shown to the user don't include stack traces or internal details.
- [ ] Any new dependency checked against the [dependency checklist](/Security/Secrets-and-Dependency-Management).

* * *

## Resources

- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) — a deliberately vulnerable web app built for practice.
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — the broadest free set of interactive vulnerability labs, organized by vulnerability class.
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) — a full methodology for testing an app's security, for when a few payloads aren't enough.
- [OWASP ZAP](https://www.zaproxy.org/) — a free, beginner-friendly automated scanner. Point it at your own staging environment, not anyone else's.
- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload) — a proxy that sits between your browser and the app so you can inspect and replay requests by hand. The free edition covers everything in this page; the paid one adds the automated scanner.
- [Semgrep](https://semgrep.dev/) — static analysis with community-maintained security rules across most common languages, useful when a project mixes stacks and a single per-language linter isn't enough.

* * *

## Related pages

- [Secure Coding](/Security/Secure-Coding) — the specific injection patterns to test for.
- [Authentication & Access Control](/Security/Authentication-and-Access-Control) — what "actually blocks access" should mean in practice.
- [Code Review](/Code-Review) — self-testing narrows what the reviewer still has to catch.
