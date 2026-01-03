# Level 02 - Network Traffic Analysis (PCAP)

## 🎯 Objective
Find the password to access the `flag02` account by analyzing network traffic.

## 🧠 My Thought Process

### Step 1: "What do we have here?"
In my home directory, I found `level02.pcap` - a packet capture file!

```bash
ls -la
# -rw-r--r-- 1 flag02 level02 8302 level02.pcap

file level02.pcap
# level02.pcap: tcpdump capture file
```

**What is a PCAP file?**
A PCAP (Packet Capture) file contains recorded network traffic. It's like a recording of everything that went through a network interface.

### Step 2: "Let me download and analyze it"
I transferred the file to my local machine for analysis:

```bash
scp -P 4242 level02@IP:~/level02.pcap .
```

### Step 3: "What's in this traffic?"
Using Python with Scapy to analyze the packets:

```python
from scapy.all import rdpcap, TCP, Raw

packets = rdpcap('level02.pcap')
for pkt in packets:
    if TCP in pkt and Raw in pkt:
        print(f"{pkt[TCP].sport}->{pkt[TCP].dport}: {repr(pkt[Raw].load)}")
```

**What I discovered:**
- Traffic on port 12121 (non-standard, likely telnet-like service)
- A login session with "wwwbugs login:" prompt
- Password being typed character by character!

### Step 4: "Reconstructing the password"
The capture showed each keystroke as a separate packet:

```
Pkt 44: 'f'
Pkt 46: 't'
Pkt 48: '_'
Pkt 50: 'w'
Pkt 52: 'a'
Pkt 54: 'n'
Pkt 56: 'd'
Pkt 58: 'r'
Pkt 60: '\x7f'  ← BACKSPACE!
Pkt 62: '\x7f'  ← BACKSPACE!
Pkt 64: '\x7f'  ← BACKSPACE!
Pkt 66: 'N'
Pkt 68: 'D'
Pkt 70: 'R'
... (more characters)
```

**The tricky part:** The user made typos and pressed backspace (0x7f)!

### Step 5: "Simulating the typing"
I had to replay the keystrokes accounting for backspaces:

```
f → t → _ → w → a → n → d → r    = "ft_wandr"
[DEL] [DEL] [DEL]                  = "ft_wa" (deleted 'r', 'd', 'n')
N → D → R → e → l                  = "ft_waNDRel"
[DEL]                               = "ft_waNDRe" (deleted 'l')
L → 0 → L                          = "ft_waNDReL0L"
```

**Final password:** `ft_waNDReL0L`

## ✅ Solution
- **Network capture:** Telnet-style login session
- **Reconstructed password:** `ft_waNDReL0L`
- **Token:** `kooda2puivaav1idi4f57q8iq`

## 📚 Concepts to Learn

### 1. Network Packet Capture
PCAP files are used for:
- Network troubleshooting
- Security analysis
- Forensic investigations
- Intrusion detection

**Tools:** Wireshark, tcpdump, tshark, Scapy

### 2. Why Telnet is Dangerous
Telnet transmits EVERYTHING in plaintext:
- Usernames
- Passwords  
- Commands
- Output

**Anyone on the network can capture and read this traffic!**

### 3. The Backspace Character (0x7f)
In ASCII:
- `0x7f` (127) = DEL (Delete)
- `0x08` (8) = BS (Backspace)

When analyzing keyboard input, you must account for editing characters!

### 4. Network Protocol Layers
```
Application Layer: Telnet data ("Password: ")
Transport Layer:   TCP (port 12121 → 39247)
Network Layer:     IP (10.1.1.2 → 10.1.1.1)
Link Layer:        Ethernet frames
```

### 5. TCP Session Reconstruction
When analyzing captures:
1. Identify the TCP stream (source/dest IP:port pairs)
2. Separate client → server from server → client
3. Reassemble data in sequence order
4. Handle retransmissions and out-of-order packets

## 🔧 Tools Used
- `scp` - Secure file copy
- `Scapy` - Python packet manipulation library
- `Wireshark` - GUI packet analyzer (alternative)
- `tshark` - CLI packet analyzer (alternative)

## 🛡️ How to Prevent This
1. **Use SSH instead of Telnet** - All traffic is encrypted
2. **Use HTTPS instead of HTTP** - Encrypts web traffic
3. **Implement network segmentation** - Limit who can sniff traffic
4. **Use VPNs** - Encrypt all traffic over untrusted networks
5. **Monitor for rogue devices** - Detect unauthorized packet capture
