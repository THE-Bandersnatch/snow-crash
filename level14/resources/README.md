# Level 14 - getflag Binary Reverse Engineering

## Objective
Reverse engineer the `getflag` binary itself to obtain the final token.

## My Notes

Final level - nothing in home directory. Need to attack `getflag` itself.

Analyzed getflag:
```bash
strings /bin/getflag
# You should not reverse this
# LD_PRELOAD
# ptrace
```

Multiple anti-debugging measures: detects ptrace (debugger), checks for LD_PRELOAD (library injection), examines /proc/self/maps.

From disassembly, the binary:
1. Checks ptrace - if being debugged, exits
2. Checks LD_PRELOAD environment variable
3. Calls getuid()
4. Uses switch statement to look up token based on UID
5. Prints token (encrypted tokens embedded in binary)

The ptrace check: `ptrace(PTRACE_TRACEME, 0, 0, 0)` - process can only have one tracer. If already being debugged, fails.

Bypass strategy: Set breakpoint AFTER ptrace call, modify return value to make it look successful. Then set breakpoint after getuid, modify EAX to flag14's UID (3014).

Found breakpoint addresses from disassembly:
- After ptrace: 0x0804898e
- After getuid: 0x08048b02

```bash
gdb -q /bin/getflag << 'EOF'
break *0x0804898e
break *0x08048b02
run
set $eax = 0        # Fake ptrace success
continue
set $eax = 3014     # Set UID to flag14
continue
quit
EOF
```

Output: `your token is 7QiHafiNa3HVozsaXkawuYrTstxbpABHD8CPnHJ`

Token: `7QiHafiNa3HVozsaXkawuYrTstxbpABHD8CPnHJ`

## Key Takeaways

**Anti-debugging techniques:** ptrace detection, LD_PRELOAD checks, /proc/self/maps analysis, timing checks. All can be bypassed by determined attacker.

**ptrace() in depth:** PTRACE_TRACEME requests to be traced by parent. If already traced, returns -1. Used for anti-debugging. It's a cat-and-mouse game.

**Two-stage bypass:** First bypass anti-debugging (fake ptrace success), then fake UID (set EAX to flag14's UID).

**Finding breakpoint addresses:** Use objdump -d to find addresses after function calls.

**Client-side security limitations:** Any code running on attacker's machine is vulnerable. Client-side checks can always be bypassed. Secrets in binaries are never safe.

## Tools Used
- `gdb` - GNU Debugger
- `objdump -d` - Disassembler
- `strings` - Extract text
- `cat /etc/passwd` - Find flag14's UID

## Lessons Learned

**For defenders:**
- Client-side checks can always be bypassed
- Secrets in binaries are never safe - use HSMs, server-side validation
- Defense in depth - multiple layers, server-side is ultimate authority, assume client is compromised
- Obfuscation ≠ Security - makes analysis harder, not impossible

Completed all 15 levels! Learned cryptography basics, password cracking, network analysis, environment manipulation, command injection, race conditions, symbolic links, binary reverse engineering, and anti-debugging bypass.
