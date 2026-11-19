Network & Environment Protection
=================================

> Production isn't the only environment that matters. Staging, admin panels, and internal tools usually hold real or near-real data, and they get far less scrutiny than production does.

* * *

## VPN access for internal systems

Anything we build for internal use — CMS admin panels, internal dashboards, ops tooling — should sit behind the company VPN rather than being reachable from the open internet. Remote admin or user access to these systems goes through the VPN, not a public URL.

Where practical, go a step further and restrict the system to only accept connections from the VPN's egress IP, denying everything else. Apache:

```apacheconf
<Directory /var/www/html/cms>
  Options Indexes FollowSymLinks
  AllowOverride None
  Require ip 203.0.113.10   # replace with your VPN's egress IP
  Require all denied
</Directory>
```

Nginx:

```nginx
location / {
  allow 203.0.113.10;   # replace with your VPN's egress IP
  deny all;
}
```

Keep the actual project-specific IPs, VPN endpoints, and access lists in that project's own runbook or README, not here. A wiki example won't get updated when the real IP changes.

**Quick check:** confirm the restriction actually works by trying to reach the system from outside the VPN — off the network, phone data, whatever isn't the allowlisted IP:

```bash
curl -i https://internal.example.com
```

You should get a connection refused or a 403, not the app. If you get the app, the config isn't doing what you think it's doing.

* * *

## Staging is not exempt

Staging needs the same protection as production: either behind the VPN above, or with strong password protection (HTTP basic auth in front of the app, for example) when a VPN isn't practical for a given client or workflow.

Don't treat staging as throwaway. It's often where real client data lands first during testing, and it tends to stay up and reachable long after a project has wrapped.

* * *

## Client-specific requirements

If a client needs a different IP, access method, or network setup, that's fine. Document it, get it reviewed by whoever owns security decisions on the project, and write down the reasoning. Any exception needs a compensating control noted alongside it.

**Worth checking periodically:** run a port scan against a project's own server to see what's actually reachable, not just what you meant to expose. `nmap -Pn your-server-ip` will show you every open port, and it's not unusual to find one nobody remembers opening.

An IP allowlist is also easy to leave stale — a contractor's home IP, an old office that moved, a VPN that got decommissioned. It's worth a look every so often to confirm every entry still belongs there.

* * *

## Resources

- [OWASP Cheat Sheet: Attack Surface Analysis](https://cheatsheetseries.owasp.org/cheatsheets/Attack_Surface_Analysis_Cheat_Sheet.html) — the principle behind restricting what's internet-facing versus VPN-only.
- [Apache `mod_authz_host` docs](https://httpd.apache.org/docs/2.4/mod/mod_authz_host.html) — the `Require ip` directive used above.
- [Nginx `ngx_http_access_module` docs](https://nginx.org/en/docs/http/ngx_http_access_module.html) — the `allow` / `deny` directives used above.
- [Nmap](https://nmap.org/) — the standard tool for checking what ports a server actually has open from the outside.

* * *

## Related pages

- [Authentication & Access Control](/Security/Authentication-and-Access-Control) — restricting how someone authenticates, on top of where they connect from.
- [Server & Infrastructure Hardening](/Security/Server-and-Infrastructure-Hardening) — hardening the server itself once network access is locked down.
