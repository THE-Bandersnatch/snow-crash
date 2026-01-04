# Level 01 - Password Hash Cracking

## Objective
Find the password to access the `flag01` account.

## My Notes

Home directory is empty again. Checked `/etc/passwd` for flag01:

```bash
cat /etc/passwd | grep flag01
```

Got: `flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash`

Wait, there's an actual hash in the passwd file instead of an `x`. That's unusual. Normally `/etc/passwd` has `x` in the password field, meaning the hash is in `/etc/shadow` (which only root can read). But here the hash is directly in `/etc/passwd`, which is world-readable. Bad practice.

The hash `42hDRfypTqqnw` is 13 characters. First two characters (`42`) look like a salt. This format matches DES crypt - the old Unix password hashing. DES crypt is weak because it only uses the first 8 chars of the password, has a small keyspace, and no iteration cost, so it cracks fast.

Used John the Ripper to crack it:
```bash
echo 'flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash' > hash.txt
john --format=descrypt hash.txt
```

Cracked almost instantly: `abcdefg`

Logged in:
```bash
su flag01
# Password: abcdefg
getflag
```

## Solution
- Hash: `42hDRfypTqqnw`
- Password: `abcdefg`
- Token: `f2av5il02puano7naaf6adaaf`

## Key Takeaways

**Password storage history:** Old systems stored DES hashes directly in `/etc/passwd` (world-readable). Modern systems use `/etc/shadow` with restricted permissions. Even better systems use bcrypt/argon2 with high iteration counts.

**DES crypt weaknesses:** Only uses first 8 characters, small keyspace, fast to compute (which means fast to crack). Modern algorithms like bcrypt are deliberately slow to resist brute force.

**Why it cracked instantly:** "abcdefg" is in every password dictionary, and DES has no computational cost, so john can try millions of hashes per second. Modern algorithms would slow this down significantly.

**Lesson:** Never store password hashes in world-readable files. Use `/etc/shadow` with 600 permissions, and use modern hashing algorithms.

## Tools Used
- `cat`, `grep` - Read passwd file
- `john` - John the Ripper password cracker
- `su` - Switch user
