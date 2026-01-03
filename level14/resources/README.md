# Level 14 - getflag Binary Reverse Engineering

## 🎯 Objective
Reverse engineer the `getflag` binary itself to obtain the final token.

## 🧠 My Thought Process

### Step 1: "There's nothing in my home directory"
```bash
ls -la
# Only default dotfiles, no level14 binary or token
```

This is the final level. The challenge is different - we need to attack `getflag` itself!

### Step 2: "Analyzing getflag"
```bash
file /bin/getflag
# ELF 32-bit LSB executable, not stripped

strings /bin/getflag | head -20
# You should not reverse this
# LD_PRELOAD
# Injection Linked lib detected exit..
# /proc/self/maps
# ptrace
```

**Multiple anti-debugging measures detected!**
1. Detects `ptrace` (debugger)
2. Checks for `LD_PRELOAD` (library injection)
3. Examines `/proc/self/maps` (memory map analysis)

### Step 3: "Understanding getflag's logic"
From disassembly (`objdump -d /bin/getflag`):

```c
// Pseudocode
int main() {
    // Anti-debugging check
    if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0) {
        puts("You should not reverse this");
        exit(1);
    }
    
    // LD_PRELOAD check
    if (getenv("LD_PRELOAD") != NULL) {
        exit(1);
    }
    
    // Memory maps check for injected libraries
    // ...
    
    // Get user ID and show corresponding token
    uid_t uid = getuid();
    switch(uid) {
        case 3000: printf("%s", decrypt(token00)); break;
        case 3001: printf("%s", decrypt(token01)); break;
        // ...
        case 3014: printf("%s", decrypt(token14)); break;
        default: printf("Nope, no token for you\n");
    }
}
```

### Step 4: "The ptrace anti-debugging"
```c
if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0)
```

**How it works:**
- A process can only have ONE tracer (debugger)
- If already being debugged, `ptrace(TRACEME)` fails
- Binary detects this and exits

**The bypass:**
- Set breakpoint AFTER ptrace call
- Modify the return value to make it look successful

### Step 5: "Multi-stage GDB bypass"
```
(gdb) break *0x0804898e      # Right after ptrace call
(gdb) break *0x08048b02      # Right after getuid call
(gdb) run

Breakpoint 1 (after ptrace)
(gdb) set $eax = 0           # Make ptrace appear successful
(gdb) continue

Breakpoint 2 (after getuid)  
(gdb) set $eax = 3014        # Set UID to flag14
(gdb) continue

# Token printed!
```

### Step 6: "Why UID 3014?"
```bash
cat /etc/passwd | grep flag14
# flag14:x:3014:3014:...
```

flag14's UID is 3014. We need to make `getuid()` return 3014.

### Step 7: "The final exploit"
```bash
gdb -q /bin/getflag << 'EOF'
break *0x0804898e
break *0x08048b02
run
set $eax = 0
continue
set $eax = 3014
continue
quit
EOF
```

Output: `your token is 7QiHafiNa3HVozsaXkawuYrTstxbpABHD8CPnHJ`

## ✅ Solution
```bash
gdb -q /bin/getflag << 'EOF'
break *0x0804898e
break *0x08048b02
run
set $eax = 0
continue
set $eax = 3014
continue
quit
EOF
```

**Token:** `7QiHafiNa3HVozsaXkawuYrTstxbpABHD8CPnHJ`

## 📚 Concepts to Learn

### 1. Anti-Debugging Techniques

| Technique | How It Works | Bypass |
|-----------|--------------|--------|
| ptrace check | Only one tracer allowed | Fake return value |
| LD_PRELOAD | Check env variable | Don't set it, use GDB |
| /proc/self/maps | Look for injected libs | Don't inject libraries |
| Timing checks | Debuggers are slow | Hardware breakpoints |
| Signal tricks | Debuggers change signals | Handle signals properly |

### 2. ptrace() in Depth
```c
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

**PTRACE_TRACEME:**
- Process requests to be traced by parent
- If already traced, returns -1 (EPERM)
- Used for anti-debugging

**The cat-and-mouse game:**
- Debugger uses ptrace to trace
- Binary uses ptrace to detect debugger
- Attacker patches ptrace response
- Developer adds more checks...

### 3. The getflag Binary Structure
```
┌────────────────────────────────────────┐
│            getflag binary              │
├────────────────────────────────────────┤
│  1. Anti-debug checks                  │
│     - ptrace()                         │
│     - LD_PRELOAD check                 │
│     - /proc/self/maps check            │
├────────────────────────────────────────┤
│  2. getuid() call                      │
│     - Gets real user ID                │
├────────────────────────────────────────┤
│  3. UID → Token lookup                 │
│     - Switch statement                 │
│     - Encrypted tokens in binary       │
│     - Decrypted on demand              │
├────────────────────────────────────────┤
│  4. Print token                        │
└────────────────────────────────────────┘
```

### 4. Finding the Right Breakpoint Addresses
Using disassembly:
```bash
objdump -d /bin/getflag | grep -A5 ptrace
#  8048989:  call   8048540 <ptrace@plt>
#  804898e:  test   eax,eax          ← After ptrace
#  8048990:  jns    80489a8

objdump -d /bin/getflag | grep -A5 getuid
#  8048afd:  call   80484b0 <getuid@plt>
#  8048b02:  mov    DWORD PTR [esp+0x18],eax  ← After getuid
```

### 5. The Two-Stage Bypass
```
Stage 1: Bypass anti-debugging
─────────────────────────────────────
Breakpoint at 0x0804898e (after ptrace)
EAX = -1 (error, being debugged)
        ↓
Set $eax = 0 (fake success)
        ↓
Continue (anti-debug bypassed!)

Stage 2: Fake UID
─────────────────────────────────────
Breakpoint at 0x08048b02 (after getuid)
EAX = 2014 (level14's real UID)
        ↓
Set $eax = 3014 (flag14's UID)
        ↓
Continue (token printed!)
```

### 6. Why This Level Is Special
This level teaches:
- Real-world reverse engineering
- Anti-debugging bypass techniques
- The limitations of client-side security
- That any code running on attacker's machine is vulnerable

## 🔧 Tools Used
- `gdb` - GNU Debugger
- `objdump -d` - Disassembler
- `strings` - Extract text from binary
- `cat /etc/passwd` - Find flag14's UID
- Understanding of x86 assembly

## 🛡️ Lessons Learned

### For Attackers:
1. Anti-debugging is just speed bumps, not walls
2. Understanding assembly is essential
3. Patience and methodical analysis wins
4. Multiple bypass techniques needed

### For Defenders:
1. **Client-side checks can always be bypassed**
2. **Secrets in binaries are never safe:**
   - Use HSMs for key storage
   - Implement server-side validation
   - Never embed secrets in distributed code
   
3. **Defense in depth:**
   - Multiple layers of protection
   - Server-side is the ultimate authority
   - Assume client is compromised

4. **Obfuscation ≠ Security:**
   - Makes analysis harder, not impossible
   - Increases attacker's cost
   - Should not be the only protection

## 🎉 Congratulations!
You've completed all 15 levels of Snow Crash! You've learned:
- Cryptography basics (ROT cipher)
- Password cracking (DES crypt)
- Network analysis (PCAP)
- Environment manipulation (PATH, env vars)
- Command injection (various languages)
- Race conditions (TOCTOU)
- Symbolic link attacks
- Binary reverse engineering
- Anti-debugging bypass

