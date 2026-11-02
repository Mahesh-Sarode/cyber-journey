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

  ---

## Lab: HTTP Header Spoofing with curl + Wireshark

**Category:** Networking / Packet Analysis
**Environment:** Kali Linux VM, Wireshark on eth0
**Objective:** Change the headers curl sends and prove it in a packet capture

### Payload
```bash
curl http://example.com
curl -A "Mozilla/5.0(Windows NT 10.0; Win64; x64)" http://example.com
curl -H "X-TEST: hello-TEST" http://example.com
```

### Breakdown
- `-A` sets the User-Agent header
- `-H` adds any custom header (name and value)
- Wireshark filters used: `http.request`, `http contains "hello-TEST"`
- `contains` is case-sensitive
- TCP payload grew from 75 to 104 bytes with the fake User-Agent (+29)
- The custom header added exactly 20 bytes (149 to 169 byte frames)

### Result
- Plain request showed `User-Agent: curl/8.22.0`
- Spoofed request showed a fake Windows browser string
- Custom header `X-TEST: hello-TEST` appeared in the packet
- Capture saved as `useragent-spoof.pcapng`

### Key Takeaway
Every HTTP header is client-controlled text. Servers can't verify the
User-Agent, so blocking tools like curl or sqlmap by User-Agent is weak
security. Never trust client headers for access decisions.  
