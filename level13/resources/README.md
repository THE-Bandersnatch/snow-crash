# Level 13 - Binary UID Check Bypass with GDB

## 🎯 Objective
Use a debugger to bypass a UID verification in a binary.

## 🧠 My Thought Process

### Step 1: "What does the binary do?"
```bash
./level13
# UID 2013 started us but we we expect 4242
```

The binary checks my UID (2013 = level13) and wants 4242.

```bash
strings level13
# UID %d started us but we we expect %d
# your token is %s
```

**Key insight:** The token is embedded in the binary! It just won't show it unless UID matches.

### Step 2: "Who is UID 4242?"
```bash
grep 4242 /etc/passwd
# (no result)
```

There's no user with UID 4242 - it's a fictional requirement!

### Step 3: "Can I become UID 4242?"
Options:
1. Create a user with UID 4242? → Need root
2. Modify /etc/passwd? → Need root
3. **Trick the binary!** → Use a debugger

### Step 4: "Understanding the binary's logic"
The binary probably does:
```c
uid_t uid = getuid();
if (uid == 4242) {
    printf("your token is %s\n", token);
} else {
    printf("UID %d started us but we expect 4242\n", uid);
}
```

**Attack idea:** Use GDB to modify the return value of `getuid()`!

### Step 5: "Setting up GDB"
```bash
gdb -q ./level13
```

**Strategy:**
1. Set a breakpoint on `getuid`
2. Let it run until breakpoint
3. Let `getuid` return
4. Modify the return value (in `$eax` register)
5. Continue execution

### Step 6: "The GDB exploit"
```
(gdb) break getuid
Breakpoint 1 at 0x80484b0

(gdb) run
Breakpoint 1, 0xb7ee4cc0 in getuid ()

(gdb) finish
Run till exit from #0  0xb7ee4cc0 in getuid ()
0x0804859a in main ()

(gdb) set $eax = 4242
(gdb) continue

your token is 2A31L79asukciNyi8uppkEuSx
```

## ✅ Solution
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

**Token:** `2A31L79asukciNyi8uppkEuSx`

## 📚 Concepts to Learn

### 1. CPU Registers (x86)
Registers are super-fast storage in the CPU:

| Register | Purpose |
|----------|---------|
| EAX | Accumulator, **function return values** |
| EBX | Base pointer |
| ECX | Counter |
| EDX | Data |
| ESP | Stack pointer |
| EBP | Base pointer |
| EIP | Instruction pointer (next instruction) |

**Key insight:** Function return values go in `EAX` (32-bit) or `RAX` (64-bit).

### 2. Function Call Convention
When a function returns:
```
getuid() returns 2013
         │
         ▼
    EAX = 2013
         │
         ▼
    Caller reads EAX
```

By modifying `EAX` after `getuid` returns, we change what the program sees!

### 3. GDB Commands Used
| Command | Purpose |
|---------|---------|
| `break getuid` | Pause when getuid is called |
| `run` | Start the program |
| `finish` | Execute until current function returns |
| `set $eax = 4242` | Modify register value |
| `continue` | Resume execution |
| `quit` | Exit GDB |

### 4. The Attack Visualization
```
Normal Execution:
─────────────────────────────────────────
main()
  │
  ├─► getuid() ──► returns 2013
  │                   │
  │   uid = 2013 ◄────┘
  │
  ├─► if (uid == 4242) → FALSE
  │
  └─► print "UID 2013 but expect 4242"

With GDB Modification:
─────────────────────────────────────────
main()
  │
  ├─► getuid() ──► returns 2013
  │                   │
  │   [GDB modifies EAX to 4242]
  │                   │
  │   uid = 4242 ◄────┘
  │
  ├─► if (uid == 4242) → TRUE
  │
  └─► print "your token is ..."
```

### 5. Why This Works
GDB has full control over the debugged process:
- Can read/write memory
- Can read/write registers
- Can modify execution flow
- Can call functions

This is why:
- Debuggers are powerful security research tools
- Anti-debugging exists in malware
- Sensitive checks shouldn't rely on client-side code

### 6. Alternative Approaches
**Using ltrace to find the token directly:**
```bash
ltrace ./level13 2>&1 | grep token
```

**Disassembly to find embedded string:**
```bash
objdump -s -j .rodata level13
# or
strings level13 | grep -i token
```

**Binary patching:**
```bash
# Find the comparison and change 4242 to our UID
# (More permanent but requires binary modification)
```

## 🔧 Tools Used
- `gdb` - GNU Debugger
- `strings` - Extract readable text
- `grep` - Search passwd file
- Understanding of x86 calling conventions

## 🛡️ How to Prevent This

### Code Countermeasures:
1. **Don't embed secrets in binaries:**
   ```c
   // WRONG
   char *token = "supersecret";
   
   // RIGHT - fetch from secure storage
   char *token = getenv("SECURE_TOKEN");
   ```

2. **Use multiple checks:**
   ```c
   // Multiple getuid() calls
   if (getuid() != 4242) exit(1);
   // ... more code ...
   if (getuid() != 4242) exit(1);  // Check again
   ```

3. **Integrity verification:**
   ```c
   // Check that code hasn't been modified
   // (Anti-tampering, but can be bypassed)
   ```

### Anti-Debugging Techniques:
1. **ptrace detection:**
   ```c
   if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0)
       exit(1);  // Being debugged!
   ```

2. **Timing checks:**
   ```c
   time_t start = time(NULL);
   // ... sensitive code ...
   if (time(NULL) - start > 1)
       exit(1);  // Too slow, likely debugging
   ```

3. **Signal handling tricks**

**Note:** All anti-debugging can be bypassed by a determined attacker. Security should not rely solely on these techniques.

