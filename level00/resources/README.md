# Level 00 - Caesar Cipher / Hidden File Discovery

## 🎯 Objective
Find the password to access the `flag00` account.

## 📝 My Notes - How I Solved This

Okay so this is level00, my first challenge. Connected as `level00` user and... nothing in the home directory. Typical CTF stuff - gotta find clues somewhere else on the system.

### First Thoughts - "Where to look?"

I need to access `flag00`, so let me search for files owned by that user. This is usually a good starting point in these challenges.

```bash
find / -user flag00 2>/dev/null
```

*(Note: The `2>/dev/null` part hides all those annoying "Permission denied" errors - learned this the hard way after scrolling through pages of errors on my first CTF)*

### Found Something Suspicious

The search found: `/usr/sbin/john`

Hmm, this is weird:
- It's in `/usr/sbin/` which is for system binaries - not a typical place for a flag file
- The name "john" is interesting... **John the Ripper** is that famous password cracker tool
- Might be a hint?

Let me check what's in it:
```bash
file /usr/sbin/john   # Check file type - probably just a text file
cat /usr/sbin/john    # Read it
```

Got this string: `cdiiddwpgswtgt`

### "This Definitely Looks Encoded"

So this string is all lowercase letters - no numbers, no special characters, just letters. This screams "simple cipher" to me. Maybe a Caesar cipher? The filename "john" might be a hint that this is related to password/crypto stuff.

My thinking:
- All letters, no symbols → probably a simple substitution cipher
- Caesar cipher (ROT cipher) is the most basic one
- Gotta try different rotation values

### Trying All Rotations

Let me brute force all 25 possible Caesar shifts (ROT1 through ROT25):

```bash
for i in {1..25}; do
  echo "ROT$i: $(echo 'cdiiddwpgswtgt' | tr 'a-z' "$(echo {a..z} | tr -d ' ' | cut -c$((i+1))-26)$(echo {a..z} | tr -d ' ' | cut -c1-$i)")"
done
```

*(This command looks messy but basically it shifts the alphabet by i positions and translates the string)*

**Bingo! ROT11 gave me:** `nottoohardhere` 

That's clearly English - "not too hard here"! Got it! 

### Getting the Flag

```bash
su flag00           # Password: nottoohardhere
getflag             # Get the token
```

## ✅ Solution
- **Encoded password:** `cdiiddwpgswtgt`
- **Decoded password (ROT11):** `nottoohardhere`
- **Token:** `x24ti5gi3x0ol2eh4esiuxias`

## 📚 Concepts to Learn

### 1. File Ownership and Permissions
Every file in Linux has an owner. Using `find -user` helps locate files belonging to specific users - a critical reconnaissance technique.

### 2. Caesar Cipher (ROT-N)
A substitution cipher where each letter is shifted by N positions. ROT13 is most common, but any rotation is possible. Named after Julius Caesar who used it for military messages.

**Formula:** `encrypted[i] = (original[i] + N) mod 26`

### 3. Security Through Obscurity ≠ Security
Hiding data using simple encoding is NOT encryption. Anyone who recognizes the pattern can decode it. Real security requires proper cryptographic algorithms.

### 4. Contextual Clues
The filename "john" was a hint toward password cracking. Always pay attention to naming conventions - developers often leave breadcrumbs.

## 🔧 Tools Used
- `find` - File search utility
- `cat` - Read file contents
- `tr` - Character translation (for cipher decoding)
- `su` - Switch user
