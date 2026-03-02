Client — Firewall — Server

- Subnet Client ↔ Firewall-eth0: 10.10.10.0/24
- Subnet Firewall-eth1 ↔ Server: 10.20.20.0/24

- Client: 10.10.10.10/24 - intnet1

- Firewall
- eth0: 10.10.10.1/24 - intnet1
- eth1: 10.20.20.1/24 - intnet2

- Server: 10.20.20.10/24 - intnet2

- Gateway:
- Client default gateway → 10.10.10.1
- Server default gateway → 10.20.20.1

---
ip a

---

firewall

```
sudo nano /etc/netplan/firewall.yaml

network:
  version: 2
  ethernets:
    enp0s3:   # intnet1
      addresses: [10.10.10.1/24]
    enp0s8:   # intnet2
      addresses: [10.10.20.1/24]

sudo netplan apply

sudo sysctl -w net.ipv4.ip_forward=1

cat /proc/sys/net/ipv4/ip_forward # result: 1

```
---
Server
```
sudo nano /etc/netplan/server.yaml

network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.10.20.10/24]
      routes:
        - to: default
          via: 10.10.20.1

sudo netplan apply

```
---
Client / Client2 / Client3 / Client4

```
# Client (10.10.10.10)

sudo nano /etc/netplan/client1.yaml

network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.10.10.10/24]
      routes:
        - to: default
          via: 10.10.10.1

sudo netplan apply

```
---
KT
```
# client ping firewall:

ping -c 2 10.10.10.1

# server ping firewall:

ping -c 2 10.10.20.1

```










