## 0. Client — Firewall — Server

- Subnet Client ↔ Firewall-eth0: 10.10.10.0/24
- Subnet Firewall-eth1 ↔ Server: 10.10.20.0/24

- Client: 10.10.10.10/24 - intnet1

- Firewall
- eth0: 10.10.10.1/24 - intnet1
- eth1: 10.10.20.1/24 - intnet2

- Server: 10.10.20.10/24 - intnet2

- Gateway:
- Client default gateway → 10.10.10.1
- Server default gateway → 10.10.20.1

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
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t mangle -F

# Nếu một packet không match bất kỳ rule nào trong chain INPUT, thì cuối cùng bị DROP.
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# cho phép các gói tin thuộc các kết nối đã được thiết lập hoặc liên quan đi qua
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT


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
Ktra
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
## 1. Заблокировать все входящие Telnet-соединения с адреса 10.10.10.10
firewall (continue)

```
# block telnet from specific IP (10.10.10.10)
iptables -A INPUT -p tcp -s 10.10.10.10 --dport 23 -j DROP

# allow telnet from others
iptables -A INPUT -p tcp --dport 23 -j ACCEPT

# Ktra
sudo iptables -L -n -v

# Kiểm tra Telnet server đang chạy
ss -tlnp | grep 23

```
CLient 
```
# Test từ máy 10.10.10.10
telnet 10.10.10.1 23 # Trying 10.10.10.1...

# Test từ máy khác 10.10.10.11
telnet 10.10.10.1 23 # Connected to 10.10.10.1

# Test nhanh bằng nmap
nmap -p 23 SERVER_IP

```

firewall (Kiểm tra counter rule)
```
sudo iptables -L -n -v
# thấy số packet đã match rule (pkts) tăng lên
```

<img width="1279" height="875" alt="image" src="https://github.com/user-attachments/assets/0e4f2447-d0b0-49b8-ae66-959919e17f3e" />

---
## 2. Установить блокировку для входящих запросов TCP не открывающих новое соединение и не принадлежащих никакому из установленных соединений

firewall (continue)
```

# Chặn TCP “NEW nhưng không SYN”
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
sudo iptables -A INPUT -p tcp -m conntrack --ctstate NEW ! --syn -j DROP

# ALLOW client connect to firewall
iptables -A INPUT -s 10.10.10.10 -j ACCEPT

# Xem rule có vào chưa (lấy line-number để lát xem counter)
sudo iptables -L INPUT -n -v --line-numbers

```
<img width="1340" height="784" alt="image" src="https://github.com/user-attachments/assets/ae7ea471-b818-4383-b96b-dcff110bc3ab" />

---
## 3. Ограничить количество параллельных соединений к серверу SSH для одного адреса - не более 3 соединений одновременно

firewall (continue)
```

# connlimit: chặn nếu >3 kết nối SSH đồng thời từ 1 IP tới server
sudo iptables -A FORWARD -d 10.10.20.10 -p tcp --dport 22 \
  -m conntrack --ctstate NEW \
  -m connlimit --connlimit-above 3 --connlimit-mask 32 \
  -j REJECT --reject-with tcp-reset

# cho phép tạo kết nối SSH mới tới server
sudo iptables -A FORWARD -d 10.10.20.10 -p tcp --dport 22 \
  -m conntrack --ctstate NEW \
  -j ACCEPT

# xem rule
sudo iptables -L FORWARD -n -v --line-numbers
```

Server
```
sudo systemctl enable ssh    # Bật tự động khởi động cùng máy
sudo systemctl start ssh     # Khởi chạy SSH ngay lập tức
sudo ss -lntp | grep ':22' # Ktra

```
Client
```
# thiết lập kết nối SSH không cần mật khẩu 

#Tạo cặp khóa SSH
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy khóa công khai lên server
ssh-copy-id xlep@10.10.20.10

# Kiểm tra kết nối không cần mật khẩu
ssh -o BatchMode=yes xlep@10.10.20.10 "echo OK"

```
Test

- Trên Client tạo 3 kết nối giữ 300s
```
for i in 1 2 3; do
  nohup ssh -n -o BatchMode=yes xlep@10.10.20.10 "sleep 300" >/dev/null 2>&1 &
done
sleep 2
ss -tn | grep "10.10.20.10:22"
```
- Thử kết nối thứ 4 (phải fail)
```
ssh -o BatchMode=yes xlep@10.10.20.10 "sleep 300"
```
- Trên Firewall (thấy dòng REJECT (connlimit) tăng pkts/bytes.)
```
sudo iptables -L FORWARD -n -v --line-numbers
```
- Trên Client clear all
```
pkill -f "ssh .*sleep 300" || true
```

---
## 4. Необходимо сделать так, чтобы ICMP пакеты типа echo- request принимались не более одного раза в секунду

firewall (continue)

```
# cho phép 1 gói ping mỗi giây
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT
# vượt quá giới hạn sẽ bị drop
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```
Từ client
```
# Vì ping mặc định 1 giây / 1 lần, nên tất cả đều được accept.
ping 10.10.10.1

# gửi 5 ping mỗi giây nên chỉ 1 ping/giây được trả lời các ping khác bị DROP
ping -i 0.2 192.168.1.10

# Kiểm tra packet bị drop trên server (xem cột pkts)
sudo iptables -L -v -n

```






