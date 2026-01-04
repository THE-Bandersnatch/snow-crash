# Level 13 - Binary UID Check Bypass with GDB

## Objective
Use a debugger to bypass a UID verification in a binary.

## My Notes

Ran the binary:
```bash
./level13
# UID 2013 started us but we we expect 4242
```

It checks my UID (2013 = level13) and wants 4242. Checked /etc/passwd - no user with UID 4242. It's a fictional requirement. The token is embedded in the binary, it just won't show it unless UID matches.

The binary probably does:
```c
uid_t uid = getuid();
if (uid == 4242) {
    printf("your token is %s\n", token);
} else {
    printf("UID %d started us but we expect 4242\n", uid);
}
```

Attack: Use GDB to modify the return value of `getuid()`. Function return values go in EAX register (32-bit) or RAX (64-bit). After `getuid()` returns, I can modify EAX before the program uses it.

```bash
gdb -q ./level13 << 'EOF'
break getuid
run
finish
set $eax = 4242
continue
quit
EOF
```

Output: `your token is 2A31L79asukciNyi8uppkEuSx`

Token: `2A31L79asukciNyi8uppkEuSx`

## Key Takeaways

**CPU registers (x86):** EAX holds function return values. By modifying EAX after function returns, we change what the program sees.

**Function call convention:** When function returns, value goes in EAX, caller reads EAX. We intercept this.

**GDB commands:** `break` sets breakpoint, `run` starts program, `finish` executes until function returns, `set $eax` modifies register, `continue` resumes execution.

**Why this works:** GDB has full control over debugged process - can read/write memory and registers, modify execution flow. This is why anti-debugging exists in malware.

**Alternative approaches:** Use ltrace to find token directly, disassembly to find embedded strings, binary patching.

## Tools Used
- `gdb` - GNU Debugger
- `strings` - Extract readable text
- Understanding of x86 calling conventions

## Prevention
- Don't embed secrets in binaries - fetch from secure storage
- Use multiple checks (call getuid() multiple times)
- Anti-debugging techniques (ptrace detection, timing checks) - but all can be bypassed by determined attacker
- Security should not rely solely on client-side checks
