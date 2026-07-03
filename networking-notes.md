# Networking Notes

## Public vs Private IP
Public: visible to internet, assigned by ISP
Private: hidden (192.168.x.x, 10.x.x.x, 172.16.x.x)
Router converts private to public using NAT

## Subnetting
/24 = 255.255.255.0 = 254 usable hosts
IP = street address, Port = apartment number

## OSI Model
7. Application  4. Transport
6. Presentation 3. Network
5. Session      2. Data Link
                1. Physical

## TCP vs UDP
TCP: reliable, 3-way handshake, ordered
UDP: fast, connectionless, fire and forget

## DNS 4 steps
Resolver → Root Server → TLD → Authoritative Name Server

## Common Ports
80=HTTP  443=HTTPS  22=SSH  21=FTP  3389=RDP
