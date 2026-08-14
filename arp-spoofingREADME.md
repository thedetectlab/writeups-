# Anatomy of a SQL Injection

*how one unescaped quote breaks a query, and the fix that actually holds*

## The setup

Take a login form. Nothing fancy — username and password, checked against a database. The backend query looks like this:

```sql
SELECT * FROM users WHERE username = '$username' AND password = '$password';
```

`$username` and `$password` are dropped straight into the string, no questions asked. This works fine for as long as every user types exactly what the form expects. It stops working the moment someone doesn't.

## Breaking it with one character

Enter this as the username:

```
' OR '1'='1
```

The query the database actually receives is:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
```

Read it the way the database parser reads it. The single quote closes the string that was supposed to hold the username. `OR '1'='1'` is now a second condition on the WHERE clause — and it's always true. Depending on operator precedence and what the password field does, this can be enough to return every row in the table, including the first admin account it finds.

That's the entire trick. Not a clever exploit chain, not obfuscated shellcode — just a quote character landing somewhere the developer assumed it never would.

## Why the naive fixes don't hold

**Blocklisting quotes.** Strip `'` from input and the login form above is safe. But `UNION SELECT`, comments (`--`, `#`), and encoded variants (`%27`, double-encoding, unicode lookalikes) route around a blocklist that only knows one character. Blocklists fail because they enumerate what you thought of, not what's possible.

**Escaping manually.** Calling `addslashes()` or similar on every input before concatenation reduces the attack surface but doesn't close it. Escaping rules vary by database and by context (a numeric field doesn't need quotes at all, so escaping quotes does nothing there). Miss one code path — one report generator, one admin panel, one "temporary" debug query — and the hole is back.

**Regex validation.** Restricting input to `[a-zA-Z0-9]+` blocks the classic payload, but it also breaks every legitimate user with an apostrophe in their name, and it still doesn't address the actual problem: input is being treated as code.

## The fix that actually holds: parameterized queries

The real issue is that the query string and the data were mixed together before the database ever saw either one. Parameterized queries keep them apart:

```python
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password = %s",
    (username, password)
)
```

Here, `%s` is a placeholder the database driver fills in *after* the query has already been parsed and compiled. The driver sends the query structure and the values as separate things over the wire. There's no string concatenation for an attacker to interrupt, because the value never becomes part of the SQL text in the first place. `' OR '1'='1` submitted this way is treated as a literal string being compared against a username column — a username that (almost certainly) doesn't exist, so the login just fails.

This is why parameterized queries (or an ORM that generates them under the hood) are the standard answer, not a "best practice" footnote. Escaping and validation can still help as defense in depth, but they're patching a design that puts data and code in the same string. Parameterization removes that design entirely.

## Quick checklist

- Never build SQL by concatenating or interpolating raw input, even for "internal" or "trusted" endpoints.
- Use parameterized queries / prepared statements for every query that includes variable input — reads and writes.
- If using an ORM, confirm it's actually parameterizing, not silently falling back to string formatting for certain query types.
- Apply least privilege on the DB account the app uses, so a successful injection still can't touch tables it has no business reading.
- Log and alert on queries that throw SQL syntax errors — a spike in malformed queries against production is often the first sign someone's probing.
