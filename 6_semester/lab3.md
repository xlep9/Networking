Client — Firewall — Server

- Subnet ngoài (Client ↔ Firewall-eth0): 10.10.10.0/24
- Subnet trong (Firewall-eth1 ↔ Server): 10.20.20.0/24

- Client: 10.10.10.10/24

- Firewall (máy chạy iptables)
- eth0 (ngoài): 10.10.10.1/24
- eth1 (trong): 10.20.20.1/24

- Server: 10.20.20.10/24

- Gateway:
- Client default gateway → 10.10.10.1
- Server default gateway → 10.20.20.1
