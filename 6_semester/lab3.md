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

# Bật vĩnh viễn (sau reboot vẫn là 1)
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system

# Ktra
sysctl net.ipv4.ip_forward

# Reset iptables
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F

# policy mặc định
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT


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

ping 10.10.10.1

# server ping firewall:

ping 10.10.20.1

# client ping server
ping 10.10.20.10

```

<img width="1749" height="1068" alt="Screenshot from 2026-03-03 10-22-16" src="https://github.com/user-attachments/assets/d2e4153e-90d9-469b-9b52-cace05d1dab1" />

---
firewall (continue) - telnet

```
# rule hặn tất cả các kết nối TCP từ địa chỉ IP 10.10.10.10 (client) đến cổng 23 (Telnet) cua firewall
sudo iptables -A INPUT -p tcp -s 10.10.10.10 --dport 23 -j DROP
# Ktra
sudo iptables -L -n -v
# delete this rule
sudo iptables -D INPUT -p tcp -s 10.10.10.10 --dport 23 -j DROP

```
<img width="1279" height="875" alt="image" src="https://github.com/user-attachments/assets/0e4f2447-d0b0-49b8-ae66-959919e17f3e" />

---
firewall (continue)

```
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

iptables -A INPUT -i lo -j ACCEPT

iptables -A INPUT   -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

sudo iptables -L -n -v

# Chặn TCP “NEW nhưng không SYN”
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
sudo iptables -A INPUT -p tcp -m conntrack --ctstate NEW ! --syn -j DROP
# Xem rule có vào chưa (lấy line-number để lát xem counter)
sudo iptables -L INPUT -n -v --line-numbers


```



