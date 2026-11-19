Secure Coding
=============

> Most breaches start with unvalidated input, not clever exploits. SQL Injection and XSS are the two you'll run into most often, and the fix for both is the same rule: never let untrusted input turn into code or markup.

* * *

## SQL Injection

This happens when user input gets pasted directly into a query string. Once that happens, the attacker is writing part of your SQL.

```php
// ❌ Vulnerable — input becomes part of the query
$user = DB::select("SELECT * FROM users WHERE username = '$username' AND password = '$password'");
// password = "'; DROP TABLE users;" turns into a second, attacker-controlled statement
```

```php
// ✅ Parameterized — input is always treated as data, never as SQL
$user = DB::select(
    "SELECT * FROM users WHERE username = :username AND password = :password",
    ['username' => $username, 'password' => $password]
);
```

Use parameterized queries or prepared statements. Every mainstream database library supports them, so there's no reason to build queries by hand. An ORM (Eloquent, Sequelize, Entity Framework, Prisma) does this for you by default. The same rule applies outside SQL too — shell commands, NoSQL queries, LDAP lookups all break the same way if you build them by concatenating strings.

### An ORM isn't automatically safe

The ORM's normal methods — `where`, `find`, `.create()` — parameterize for you. The problem is the raw-SQL escape hatch that every ORM also provides. If you build that with string interpolation, you're back to square one.

```php
// ❌ Vulnerable — raw SQL escape hatch, input concatenated straight in
$users = DB::table('users')->whereRaw("username = '$username'")->get();
```

```php
// ✅ Same escape hatch, input passed as a bound parameter instead
$users = DB::table('users')->whereRaw('username = ?', [$username])->get();
```

```js
// ❌ Vulnerable — Prisma's raw query helper, string-interpolated
const users = await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE username = '${username}'`);

// ✅ Tagged-template raw query — Prisma parameterizes the interpolated values for you
const users = await prisma.$queryRaw`SELECT * FROM users WHERE username = ${username}`;
```

Use the query builder for anything driven by user input. Save raw SQL for cases the ORM can't express, like complex reporting or vendor-specific functions. And when you do write raw SQL, bind the parameters — `whereRaw('col = ?', [value])`, a `$queryRaw` tagged template, EF Core's `FromSqlInterpolated`. The dangerous version is the one built with string concatenation: `whereRaw("col = '$value'")`, `$queryRawUnsafe`, `FromSqlRaw` fed a concatenated string. In review, any raw-query call is worth a second look at how the value got in there.

**Quick check:** before a release, grep the codebase for the raw-query escape hatches your ORM provides, and skim what's actually in them:

```bash
grep -rn "whereRaw\|queryRawUnsafe\|FromSqlRaw" src/
```

It won't catch everything (a taint-tracking tool does that better), but it takes thirty seconds and often turns up the one query everyone forgot about.

### User-controlled column or table names

Parameters only work for values. If user input picks a column name, table name, or sort direction — a common pattern for "sort by any field" features — you can't bind that. Identifiers don't work as parameter placeholders.

```php
// ❌ Vulnerable — column name built from user input, no way to bind it as a parameter
$sortColumn = $request->input('sort'); // attacker sends "id; DROP TABLE users;--"
$users = DB::table('users')->orderByRaw("$sortColumn")->get();
```

```php
// ✅ Validate against an allowlist of real column names before using it
$allowedSorts = ['name', 'created_at', 'email'];
$sortColumn = in_array($request->input('sort'), $allowedSorts, true)
    ? $request->input('sort')
    : 'created_at';

$users = DB::table('users')->orderBy($sortColumn)->get();
```

The fix is an allowlist, not sanitization. Check the input against a fixed list of columns you control, and fall back to a safe default if it doesn't match. Don't try to sanitize an arbitrary string into a safe identifier — it's harder to get right than it looks. The same applies to dynamic filters and dynamically chosen tables, anywhere input picks what to query instead of what value to match.

* * *

## Cross-Site Scripting (XSS)

XSS happens when untrusted input gets rendered as HTML or JavaScript instead of plain text. Whatever the attacker typed then runs in someone else's browser.

```html
<!-- ❌ Renders raw HTML — an attacker's <script> tag executes -->
<div v-html="userInput"></div>

<!-- ✅ Escaped by default — displays as text, doesn't execute -->
<p>{{ userInput }}</p>
```

Your framework's default templating already escapes output. Vue's `{{ }}`, React's JSX, and Blade's `{{ }}` all do this automatically. The risk shows up when someone reaches for the raw-HTML escape hatch — `v-html`, `dangerouslySetInnerHTML`, `{!! !!}` — without sanitizing the input first. Validate input on the way in, and escape it on the way out. Validation catches obviously bad input, but escaping is what actually stops the attack, so don't skip it just because the input was validated. If you need to render user-supplied HTML, like output from a rich-text editor, run it through a sanitizer library instead of writing your own filter.

Most linters can flag the dangerous sink for you before review even starts — React's `react/no-danger` rule and Vue's `vue/no-v-html` rule both catch it at the point it's written, which is a lot cheaper than catching it in a PR.

**A few habits worth keeping:**

- Keep a short list of your project's raw-query and raw-HTML escape hatches somewhere reviewers can see it — a comment in the code style guide, a line in the PR template, whatever gets read. It turns "did anyone check this" into "yes, here's where we check it."
- New sort/filter parameter? Write the allowlist first, before the feature works end to end. It's a five-minute addition now and an easy one to forget once the happy path is working.
- If a linter rule for dangerous sinks isn't already on, turning it on is a bigger win than any one code review comment — it catches the mistake every time, not just when someone happens to notice.

* * *

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — the standard reference for common web vulnerability classes, including injection and XSS.
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PortSwigger Web Security Academy — SQL injection](https://portswigger.net/web-security/sql-injection) and [Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) — free interactive labs where you exploit real vulnerable apps.
- [CodeQL](https://codeql.github.com/) — semantic code analysis that traces untrusted input from where it enters to where it's used, catching injection paths a plain text search would miss.

* * *

## Related pages

- [Code Review](/Code-Review) — this is exactly the kind of issue a second pair of eyes should catch before merge.
- [Self-Testing & Verification](/Security/Self-Testing-and-Verification) — how to try these payloads against your own feature before it reaches review.
- [Secrets & Dependency Management](/Security/Secrets-and-Dependency-Management) — sanitizer and validation libraries still need to be kept up to date.
