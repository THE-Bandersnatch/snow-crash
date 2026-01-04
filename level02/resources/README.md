# Level 02 - Network Traffic Analysis (PCAP)

## Objective
Find the password to access the `flag02` account by analyzing network traffic.

## My Notes

Found `level02.pcap` in the home directory. PCAP file = packet capture, recorded network traffic.

```bash
file level02.pcap
# level02.pcap: tcpdump capture file
```

Downloaded it to my local machine to analyze:
```bash
scp -P 4242 level02@IP:~/level02.pcap .
```

Used Python with Scapy to extract the traffic:
```python
from scapy.all import rdpcap, TCP, Raw

packets = rdpcap('level02.pcap')
for pkt in packets:
    if TCP in pkt and Raw in pkt:
        print(f"{pkt[TCP].sport}->{pkt[TCP].dport}: {repr(pkt[Raw].load)}")
```

Found traffic on port 12121 (non-standard port, probably telnet-like service). There's a login session with "wwwbugs login:" prompt, and the password is being typed character by character - each keystroke is a separate packet.

The tricky part: the user made typos and used backspace (0x7f). Had to reconstruct the actual password by simulating the typing sequence:

```
'f' → 't' → '_' → 'w' → 'a' → 'n' → 'd' → 'r'  = "ft_wandr"
'\x7f' (backspace, 3 times)                      = "ft_wa" (deleted 'r', 'd', 'n')
'N' → 'D' → 'R' → 'e' → 'l'                      = "ft_waNDRel"
'\x7f' (backspace)                                = "ft_waNDRe" (deleted 'l')
'L' → '0' → 'L'                                  = "ft_waNDReL0L"
```

Final password: `ft_waNDReL0L`

```bash
su flag02
# Password: ft_waNDReL0L
getflag
```

## Solution
- Password: `ft_waNDReL0L`
- Token: `kooda2puivaav1idi4f57q8iq`

## Key Takeaways

**PCAP analysis:** Packet captures record all network traffic. Useful for troubleshooting, security analysis, forensics. Tools: Wireshark, tcpdump, Scapy.

**Why telnet is dangerous:** Everything is transmitted in plaintext - usernames, passwords, commands, output. Anyone on the network can capture and read it. Use SSH instead (encrypted).

**Backspace character (0x7f):** When analyzing keyboard input from network captures, need to account for editing characters like backspace. 0x7f = DEL, 0x08 = BS. The actual password might differ from what you see if there were typos and corrections.

**Network protocol layers:** Application layer (telnet data), transport (TCP), network (IP), link (Ethernet). Understanding layers helps with analysis.

## Tools Used
- `scp` - Copy file from VM
- `Scapy` - Python packet analysis
- Wireshark/tshark also work
