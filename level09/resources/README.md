# Level 09 - Custom Encoding Reverse Engineering

## Objective
Reverse engineer a custom encoding algorithm to decode the token.

## My Notes

Found a setuid binary and an unreadable token file. Ran the binary - it needs one argument:

```bash
./level09 test
# tftu
```

The output is different from input. Tested more inputs to find the pattern:
```bash
./level09 "aaaa"
# abcd

./level09 "0000"
# 0123

./level09 "hello"
# hfnos
```

Looking at the transformations:
- "aaaa" → "abcd" (a+0, a+1, a+2, a+3)
- "0000" → "0123" (0+0, 0+1, 0+2, 0+3)
- "test" → "tftu" (t+0, e+1, s+2, t+3)

Each character is shifted by its position index. Encoding formula: `encoded[i] = original[i] + i`

To decode: `original[i] = encoded[i] - i`

The binary is setuid and can read the token file. Got the encoded bytes from the token file. Wrote a Python script to decode:

```python
# Encoded bytes from token file
encoded = [102, 52, 107, 109, 109, 54, 112, 124, 61, 94, 96, 112, 116, 68, 66, 68, 117, 123]

decoded = ""
for i, byte in enumerate(encoded):
    decoded += chr(byte - i)

print(decoded)  # f3iji1ju5yuevaus41q1afiuq
```

Used the decoded password to login:
```bash
su flag09  # Password: f3iji1ju5yuevaus41q1afiuq
getflag
```

Token: `s5cAJpM8ev6XHw998pRWG728z`

## Key Takeaways

**Reverse engineering process:** Black-box testing (provide inputs, observe outputs), pattern recognition, hypothesis formation, verification, implementation of reverse algorithm.

**Position-based encoding:** Each character shifted by position index. Not encryption - trivially reversible once pattern is understood.

**ASCII values:** Characters stored as numbers. Python `ord()` and `chr()` convert between characters and ASCII values.

**Working with binary data:** Characters might not be printable, use hex/decimal representation, be careful with string vs bytes in Python.

## Tools Used
- The binary itself for testing
- Python for decode script
- `xxd` or `hexdump` to view raw bytes

## Prevention
- Obscurity is not security - encoding like this is easily broken
- Use real encryption (AES, ChaCha20, etc.)
- Protect keys, not algorithms (Kerckhoffs's principle)
- Static analysis defeats obscurity
