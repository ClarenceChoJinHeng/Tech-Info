Server & Infrastructure Hardening
==================================

> Baseline server configuration that applies no matter what stack you're deploying. Small on their own, but each one is a common finding in any external security audit or pentest report.

* * *

## Disable directory indexing

A misconfigured server will list the contents of a folder when there's no index file in it — file names, internal structure, and sometimes files that were never meant to be public, like backups or a stray `.env`.

Confirm directory indexing is disabled on every server hosting a project you're responsible for. If you don't manage the server directly, ask the Person in Charge (PIC) to disable it and confirm back.

Apache: remove `Indexes` from the `Options` directive. Nginx: `autoindex off;` (already the default there).

Confirm it directly rather than assuming the config took effect:

```bash
curl -i https://example.com/some/folder/
```

You want a 403 or 404, not a page listing filenames.

* * *

## HTTPS everywhere

All traffic should go over HTTPS, including staging and internal tools, not just production's main domain. Redirect HTTP to HTTPS at the server or load-balancer level rather than relying on the app to handle it. Use a certificate manager (Let's Encrypt/Certbot, or your cloud provider's managed certs) so renewal happens automatically. An expired cert is an outage nobody planned for.

* * *

## Security headers

A handful of response headers raise the bar significantly for almost no engineering effort:

| Header | What it does |
| --- | --- |
| `Content-Security-Policy` | Restricts where scripts/styles/frames can load from — a strong second layer against XSS. |
| `X-Frame-Options: DENY` (or `frame-ancestors` in CSP) | Prevents the site from being embedded in a clickjacking iframe. |
| `Strict-Transport-Security` | Tells browsers to only ever connect over HTTPS. |
| `X-Content-Type-Options: nosniff` | Stops browsers from guessing content types in a way that can be exploited. |

Most frameworks ship a middleware or plugin that sets sane defaults for all of these — `helmet` for Node, a security header package for Laravel. Use that instead of hand-rolling it.

A quick way to eyeball what's actually being sent, without waiting on a full scan:

```bash
curl -I https://example.com
```

Worth rerunning after any change to a CDN or reverse proxy in front of the app — those can quietly strip or override headers the app itself is setting correctly.

* * *

## Don't leak internals in errors

Debug mode, stack traces, and verbose error pages belong in local development only, never staging or production. Return a generic message to the client and log the real details server-side, where only the team can see them.

* * *

## Turn off server version disclosure

Every stack broadcasts its version somewhere by default — a response header, an error page footer, a meta tag. That's a free head start for anyone running a scanner: they match the version against known CVEs before they've looked at anything your app actually does.

- **Apache**: `ServerTokens Prod` and `ServerSignature Off` in the main config (not just one vhost — the signature shows up on every generated error page too).
- **Nginx**: `server_tokens off;`.
- **Node/Express**: `app.disable('x-powered-by')`, or strip the header in whatever security-header middleware you're already using (see above).
- **PHP**: `expose_php = Off` in `php.ini`.
- **ASP.NET**: remove the `X-AspNet-Version` and `X-Powered-By` headers via `web.config`.

Confirm it the same way as the headers above — don't assume the config took effect:

```bash
curl -I https://example.com
```

* * *

## Remove default pages and scaffolding

Anything left over from provisioning the server or standing up the framework tells an attacker the box hasn't been touched since setup — and sometimes it's a way in on its own. Work through this before a server goes anywhere near production, and ideally before it even reaches staging:

- **Default landing pages** — nginx's "Welcome to nginx!", Apache's `it-works.html`, IIS's default site, a freshly scaffolded framework's placeholder homepage. A default page still being served is the clearest signal a box was never actually configured.
- **Installer and setup scripts** — `install.php`, `setup.php`, `phpinfo.php`, a CMS's install wizard. These are meant to run once; leaving them reachable afterward is an unnecessary risk, not a convenience.
- **Sample content and demo data** — starter themes, example plugins, seeded demo accounts or posts that ship with a CMS or boilerplate. Remove whatever you're not actually using.
- **Default credentials** — any account an installer creates automatically (`admin`/`admin`, a database's default superuser). Change the username and password, or disable the account outright, immediately after provisioning — don't wait for a "later" that never comes.
- **Well-known admin paths with no additional protection** — `/wp-admin`, `/phpmyadmin`, `/admin` sitting open at the default path. You don't have to relocate it, but it should sit behind the same access controls as any other admin surface — see [Network & Environment Protection](/Security/Network-and-Environment-Protection).

Make this part of the provisioning checklist itself, not a follow-up task someone might forget — confirm it before a server is handed off or goes live, the same way you'd confirm directory indexing above.

* * *

## Resources

- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) and [HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html) — reference for every header in the table above.
- [Mozilla HTTP Observatory](https://developer.mozilla.org/en-US/observatory) — scan a live site and get a graded report on its headers and TLS config. Worth running against staging before launch.
- [OWASP Testing Guide – Configuration and Deployment Management](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/) — the fuller checklist behind the version-disclosure and default-scaffolding items above.
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) — a deeper TLS/certificate configuration check.
- [testssl.sh](https://testssl.sh/) — the same kind of TLS check as SSL Labs, but a command-line tool you run yourself. Useful for internal hosts that a public scanner can't reach.

* * *

## Related pages

- [Network & Environment Protection](/Security/Network-and-Environment-Protection) — restricting who can reach the server, alongside hardening the server itself.
- [Secure Coding](/Security/Secure-Coding) — CSP and output escaping work together against XSS; neither one replaces the other.
