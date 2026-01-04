# Level 00 - Caesar Cipher / Hidden File Discovery

## 🎯 Objective
Find the password to access the `flag00` account.

## My Notes

Started level00. Connected as `level00` user - home directory is empty. Need to find the password for `flag00` somewhere on the system.

Searched for files owned by flag00:
```bash
find / -user flag00 2>/dev/null
```

The `2>/dev/null` redirects stderr to hide permission denied errors - makes output way cleaner.

Found `/usr/sbin/john`. That's unusual - it's in a system binary directory but the name "john" reminds me of John the Ripper (password cracker). Could be a hint.

Checked the file:
```bash
file /usr/sbin/john
cat /usr/sbin/john
```

Got: `cdiiddwpgswtgt`

This looks encoded. All lowercase letters, no numbers or special chars. Probably a simple substitution cipher. Caesar cipher seems like a good first guess.

Tried all 25 rotations with a simple bash loop:
```bash
for i in {1..25}; do
  echo "ROT$i: $(echo 'cdiiddwpgswtgt' | tr 'a-z' "$(echo {a..z} | tr -d ' ' | cut -c$((i+1))-26)$(echo {a..z} | tr -d ' ' | cut -c1-$i)")"
done
```

ROT11 gave `nottoohardhere` - that's clearly English! "not too hard here" makes sense.

Used it to login:
```bash
su flag00
# Password: nottoohardhere
getflag
```

## ✅ Solution
- **Encoded password:** `cdiiddwpgswtgt`
- **Decoded password (ROT11):** `nottoohardhere`
- **Token:** `x24ti5gi3x0ol2eh4esiuxias`

## Key Takeaways

**File ownership and find:** Using `find -user` is a basic recon technique to locate files belonging to specific users. Simple but effective.

**Caesar cipher (ROT-N):** Each letter is shifted by N positions in the alphabet. ROT13 is common, but any rotation works. Formula: `encrypted[i] = (original[i] + N) mod 26`. Easy to brute force since there are only 25 possible rotations.

**Security through obscurity doesn't work:** Simple encoding like this isn't encryption. Anyone who recognizes the pattern can decode it. Real security needs proper crypto.

**Context clues matter:** The filename "john" was a hint pointing to password/crypto stuff. Pay attention to naming - sometimes developers leave hints.

## 🔧 Tools Used
- `find` - File search utility
- `cat` - Read file contents
- `tr` - Character translation (for cipher decoding)
- `su` - Switch user
