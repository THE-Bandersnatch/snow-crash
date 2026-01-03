# Level 01 - Password Hash Cracking

## 🎯 Objective
Find the password to access the `flag01` account.

## 🧠 My Thought Process

### Step 1: "Let me check the usual suspects"
After finding nothing special in my home directory, I checked the classic authentication files:

```bash
cat /etc/passwd | grep flag01
```

Result: `flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash`

### Step 2: "Wait... there's a hash IN the passwd file?!"
Normally, `/etc/passwd` contains an `x` in the second field, indicating the actual password hash is in `/etc/shadow`. But here, I see `42hDRfypTqqnw` - **an actual password hash!**

**Why is this bad?**
- `/etc/passwd` is world-readable (everyone can see it)
- `/etc/shadow` has restricted permissions (only root can read)
- Storing hashes in passwd exposes them to all users

### Step 3: "What type of hash is this?"
The format `42hDRfypTqqnw` tells me:
- It's 13 characters long
- First 2 characters (`42`) are the salt
- This is a **DES crypt hash** - the oldest Unix password hashing algorithm

**Why DES is weak:**
- Only uses first 8 characters of password
- Small keyspace (easily brute-forced)
- No iterations (fast to compute = fast to crack)

### Step 4: "Time to crack it!"
I used John the Ripper, the industry-standard password cracker:

```bash
echo 'flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash' > hash.txt
john --format=descrypt hash.txt
```

Result (almost instant!): `abcdefg`

### Step 5: "Verify and capture the flag"
```bash
su flag01           # Enter password: abcdefg
getflag             # Get the token
```

## ✅ Solution
- **Hash found:** `42hDRfypTqqnw`
- **Cracked password:** `abcdefg`
- **Token:** `f2av5il02puano7naaf6adaaf`

## 📚 Concepts to Learn

### 1. Unix Password Storage Evolution
| Era | Method | Security |
|-----|--------|----------|
| Ancient | Plain text in /etc/passwd | Terrible |
| Old | DES hash in /etc/passwd | Bad |
| Modern | SHA-512 hash in /etc/shadow | Good |
| Current | bcrypt/argon2 | Excellent |

### 2. The /etc/passwd File Format
```
username:password:UID:GID:comment:home:shell
flag01  :42hDRfy..:3001:3001:      :/home/flag01:/bin/bash
```
The `x` in modern systems means "check /etc/shadow instead."

### 3. DES Crypt Algorithm
- **Salt:** First 2 characters (randomizes the hash)
- **Limitation:** Only uses first 8 characters of password
- **Speed:** Can compute millions of hashes per second
- **Status:** OBSOLETE - Do not use!

### 4. Password Cracking Techniques
1. **Dictionary attack:** Try common words
2. **Brute force:** Try all combinations
3. **Rainbow tables:** Pre-computed hash lookup
4. **Hybrid:** Dictionary + mutations (p@ssw0rd)

### 5. Why "abcdefg" Was Found Instantly
- It's in every password dictionary
- DES has no computational cost (no stretching)
- Modern algorithms like bcrypt deliberately slow down hashing

## 🔧 Tools Used
- `cat` - Read file contents
- `grep` - Filter output
- `john` - John the Ripper password cracker
- `su` - Switch user

## 🛡️ How to Prevent This
1. Never store password hashes in /etc/passwd
2. Use /etc/shadow with proper permissions (600)
3. Use modern algorithms (bcrypt, argon2)
4. Enforce strong password policies
5. Implement account lockout after failed attempts
