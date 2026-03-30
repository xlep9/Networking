<img width="763" height="1028" alt="image" src="https://github.com/user-attachments/assets/7fdc970a-b488-4c3a-8784-2fdc27ca48ad" />

1. kiểm tra trong máy mình trước, không hỏi DNS ngay lập tức, mà thường kiểm tra theo thứ tự:
- Trình duyệt cache: trước đó đã từng vào trang này chưa
- DNS cache của hệ điều hành
- File hosts: nếu trong máy có khai báo tên miền trỏ sẵn tới IP

2. Nếu trong máy không có, client sẽ gửi câu hỏi DNS đến DNS resolver đã được cấu hình sẵn.
Resolver này có thể là:
- DNS của router/gateway
- DNS nội bộ trong công ty/lab
- DNS công cộng như 8.8.8.8, 1.1.1.1

3. Resolver kiểm tra cache của nó, Nếu còn trong cache, resolver trả lời ngay cho client.
4. Nếu resolver chưa biết, nó đi hỏi tiếp
- Hỏi root DNS server: root server trả về địa chỉ (a.gtld-servers.net -> 192.5.6.30) các TLD (Top-Level Domain) server ( It manages domain name suffixes like .com, .org, ...).
- Hỏi TLD server: trả về địa chỉ (ns1.example.net) authoritative name server (Máy chủ tên miền có thẩm quyền) của example.com
- Hỏi authoritative DNS server: Resolver hỏi authoritative server của example.com (ns1.example.net), Authoritative trả:
www.example.com A 93.184.216.34 (bản ghi cuối cùng mà resolver đang cần)

---
- NS record = hỏi ai quản lý
- A/AAAA record = hỏi IP là gì

tóm lại là: 

- root trả: NS của .com, IP của các NS đó
- TLD trả: NS của example.com và IP của các NS đó
- Authoritative trả: www.example.com A 93.184.216.34. Tức là authoritative trả: bản ghi cuối cùng mà resolver đang cần

---
## SRV12 
là secondary (slave) DNS server dùng để dự phòng và đồng bộ dữ liệu từ SRV11, nhằm đảm bảo hệ thống DNS luôn hoạt động khi SRV11 gặp sự cố và tăng độ tin cậy của hệ thống.

---
## File zone là gì?
- là “sổ dữ liệu” của một zone
- Ví dụ zone lab.test có thể có file:

---
Client
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.1/24
      nameservers:
        addresses:
          - 192.168.56.101   # SRV1

```
SRV1
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:   # NAT
      dhcp4: true

    enp0s8:   # Host-only
      dhcp4: false
      addresses:
        - 192.168.56.101/24
```
SRV2
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.102/24
```
---
SRV1 làm recursive resolver cho client nội bộ, làm authoritative DNS cho zone lab.test
```
# /etc/bind/named.conf.options - các máy trong ds này được phép hỏi đệ quy

acl "trusted" {
        192.168.56.101;    # srv1
        192.168.56.102;    # srv2
        192.168.56.1;  # client
};

options {
        directory "/var/cache/bind";
        recursion yes; # разрешаем рекурсивные запросы
        allow-recursion { trusted; }; # разрешаем рекурсивные запросы только от доверенных клиентов
        listen-on { 192.168.56.101; }; # приватный адрес SRV1, на котором будет работать сервис DNS
        allow-transfer { none; }; # запрет переноса зоны
        forwarders {
                8.8.8.8;
                8.8.4.4;
        }; # перенаправление запросов, неизвестных серверу на Google DNS
};

# /etc/bind/named.conf.local - khởi tạo một zone tên là lab.

zone "lab.test" {
    type master;
    file "/etc/bind/zones/db.lab.test"; # путь к файлу зоны
};

#  /etc/bind/zones/db.lab.test - ghi vào file đã khai báo zone bên trên

$TTL    604800
@       IN      SOA     srv1.lab.test. admin.lab.test. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL

     IN      NS      srv1.lab.test. ; NS - name server, khai báo rằng srv1.lab.test là máy chủ DNS chịu trách nhiệm cho zone lab.test

srv1.lab.test.        IN      A      192.168.56.101 ; khai báo rằng tên miền srv1.lab.test trỏ đến địa chỉ IP 192.168.56.101
srv2.lab.test.        IN      A      192.168.56.102 

```
---
SRV2 - DNS phụ (my.lab.test + DNSSEC) - Muốn hỏi về domain my.lab.test thì hãy hỏi server srv2.lab.test
```
# Tạo zone my.lab.test (SRV2)

# khai bao
# sudo nano /etc/bind/named.conf.local
zone "my.lab.test" {
    type master;
    file "/etc/bind/zones/db.my.lab.test";
};

# File zone
# sudo nano /etc/bind/zones/db.my.lab.test
$TTL 604800
@   IN  SOA srv2.my.lab.test. admin.my.lab.test. (
        3
        604800
        86400
        2419200
        604800
)

    IN  NS  srv2.lab.test.

srv2.lab.test. IN A 192.168.56.102 ; Server DNS srv2.lab.test nằm ở IP 192.168.56.102
; Delegation for my.lab.test (Uỷ quyền cho SRV2)
my.lab.test.     IN      NS      srv2.lab.test. ; Muốn biết thông tin của my.lab.test thì hãy hỏi server srv2.lab.test
```
---
Cấu hình DNSSEC (trên SRV2)

```
# Tạo thư mục key
sudo mkdir -p /etc/bind/keys
cd /etc/bind/keys

# Tạo KSK
dnssec-keygen -f KSK -a RSASHA1 -b 2048 -n ZONE my.lab.test
# Tạo ZSK
dnssec-keygen -a RSASHA1 -b 2048 -n ZONE my.lab.test
# Chèn key vào zone
cat Kmy.lab.test*.key >> /etc/bind/zones/db.my.lab.test
# Ký zone
cd /etc/bind/zones
dnssec-signzone -o my.lab.test -e +3mo -N INCREMENT -K /etc/bind/keys db.my.lab.test
# Bật DNSSEC
sudo nano /etc/bind/named.conf.options
dnssec-enable yes;
dnssec-validation yes;
dnssec-lookaside auto;
# Trỏ sang file signed
zone "my.lab.test" {
    type master;
    file "/etc/bind/zones/db.my.lab.test.signed";
};
# Thêm DS record vào SRV1
cat /etc/bind/zones/dsset-my.lab.test
# Copy nội dung → dán vào zone lab.test trên SRV1
sudo nano /etc/bind/zones/db.lab.test
# Restart dịch vụ
sudo systemctl restart bind9
```
```
Trên CLIENT
Test domain
dig @192.168.56.101 srv1.lab.test
dig @192.168.56.102 www.my.lab.test
Test DNSSEC
dig my.lab.test +dnssec
```
---
## DNSSEC
- private key = dùng để ký
- public key = dùng để kiểm tra chữ ký
- DNSKEY: chứa public key của zone (chứa public ZSK và public KSK)
- ZSK (Zone Signing Key - Khóa ký Zone): Một cặp khóa (Public/Private) được dùng để ký lên các RRSET dữ liệu thông thường (như bản ghi A, MX, CNAME) trong một Zone
- KSK (Key Signing Key - Khóa ký khóa): Một cặp khóa (Public/Private) có mức độ tin cậy cao hơn. Nhiệm vụ chính của KSK là ký lên bản ghi DNSKEY (chứa ZSK).
- RRSET là tập hợp các bản ghi có cùng:
cùng tên,
cùng loại,
cùng class,
ví dụ: lab.test.   IN NS   srv11.lab.test.
lab.test.   IN NS   srv12.lab.test. Chúng hợp lại thành một RRSET NS của lab.test
chain of trust = chuỗi “ai xác nhận ai” để resolver dám tin dữ liệu cuối cùng
- RRSIG (Resource Record Signature - Chữ ký bản ghi): Đây là chữ ký số của một RRSET, được tạo ra bởi Private Key của ZSK. RRSIG cho trình phân giải (Resolver) biết bản ghi này là "thật", không bị thay đổi.












