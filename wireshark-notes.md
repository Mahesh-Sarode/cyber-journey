## Lab: First Wireshark Capture

**Category:** Networking / Packet Analysis
**Environment:** Kali Linux VM (VirtualBox), Wireshark on eth0
**Objective:** Capture my own HTTP traffic and identify handshake, request, and headers

### Commands
​```bash
sudo wireshark &
ping -c 4 google.com
curl http://example.com
​```

### Filters Used
- `icmp` – ping packets
- `dns` – domain lookups
- `http.request` – HTTP requests (found HEAD and GET)
- `tcp.flags.syn==1` – SYN and SYN-ACK of the handshake
- `tcp.port == 54444` – one full connection

### Findings
- TCP 3-way handshake: SYN → SYN-ACK → ACK, about 47 ms RTT
- `User-Agent: curl/8.22.0` is sent to every site
- HEAD and GET differ by 1 byte (method name length)
- Traffic went over IPv6 to Cloudflare
- Troubleshot a failed `curl` to neverssl.com (port 80 timeout, SYN retransmissions) using `nc -zv` and a retransmission filter

### Key Takeaway
Plain HTTP is readable by anyone on the network, and the User-Agent is
client-controlled so it can be spoofed (`curl -A`). Servers that leak
version info (e.g. `Server: Apache/2.4.68`) help attackers fingerprint
targets; Cloudflare hides it.
