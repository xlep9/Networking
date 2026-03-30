## SRV2
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      optional: true
      dhcp4-overrides:
        use-dns: false
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.103/24
      nameservers:
        addresses:
          - 192.168.56.254
          - 8.8.8.8
```
- sudo nano /etc/bind/named.conf.options
```
options {
    directory "/var/cache/bind";
    recursion no;
    listen-on { 127.0.0.1; 192.168.56.103; };
    listen-on-v6 { none; };
    allow-query { any; };
};
```
- sudo nano /etc/bind/named.conf.local
```
zone "my.lab.test" {
    type master;
    file "/etc/bind/zones/db.my.lab.test";
};
```
- sudo nano /etc/bind/zones/db.my.lab.test
```
$TTL 604800
@   IN  SOA srv2.lab.test. admin.lab.test. (
        2026032901  ; Serial
        604800      ; Refresh
        86400       ; Retry
        2419200     ; Expire
        604800      ; Negative Cache TTL
)

@   IN  NS  srv2.lab.test.

www IN  A   192.168.56.150
app IN  A   192.168.56.151
mail IN A   192.168.56.152
```
- test
```
dig @192.168.56.103 my.lab.test SOA
dig @192.168.56.103 my.lab.test NS
dig @192.168.56.103 www.my.lab.test A
```
### Bật DNSSEC
- Sinh KSK và ZSK
```
cd /etc/bind/keys

# Tạo ZSK (Key Signing Key - flag 256)
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE my.lab.test

# Tạo KSK (Key Signing Key - flag 257)
sudo dnssec-keygen -f KSK -a RSASHA256 -b 2048 -n ZONE my.lab.test

# Liệt kê các key đã tạo
ls Kmy.lab.test*.key
```
- Tạo file zone tạm có chứa DNSKEY records
```
sudo bash -c 'cat > /tmp/db.my.lab.test.withkeys << "EOF"
$TTL 604800
@   IN  SOA srv2.lab.test. admin.lab.test. (
        2026032903  ; Serial (tăng lên 03)
        604800
        86400
        2419200
        604800
)

; DNSKEY records - ZSK (flag 256) and KSK (flag 257)
$INCLUDE /etc/bind/keys/Kmy.lab.test.+008+00669.key
$INCLUDE /etc/bind/keys/Kmy.lab.test.+008+29947.key

; NS record
@   IN  NS  srv2.lab.test.

; A records
www IN  A   192.168.56.150
app IN  A   192.168.56.151
mail IN A   192.168.56.152
EOF'
```
- Ký zone con
```
cd /etc/bind/keys
sudo dnssec-signzone -S \
    -o my.lab.test \
    -f /etc/bind/zones/db.my.lab.test.signed \
    -N INCREMENT \
    -k Kmy.lab.test.+008+29947.key \
    /etc/bind/zones/db.my.lab.test \
    Kmy.lab.test.+008+00669.key
# Option -S sẽ tự động thêm DNSKEY records vào zone
```
- Chuyển BIND sang dùng file đã ký
```
sudo nano /etc/bind/named.conf.local

zone "my.lab.test" {
    type master;
    file "/etc/bind/zones/db.my.lab.test.signed";
};
```
- Tạo DS record để gửi lên zone cha
```
cd /etc/bind/keys
sudo dnssec-dsfromkey -2 Kmy.lab.test*.key | sudo tee /etc/bind/zones/dsset-my.lab.test

# Xem nội dung DS record
cat /etc/bind/zones/dsset-my.lab.test
```
- Khởi động lại BIND và kiểm tra
```
sudo named-checkconf
sudo systemctl restart bind9
sudo systemctl status bind9
```
- Kiểm tra DNSSEC trên zone con
```
# Kiểm tra DNSKEY (phải có cả KSK và ZSK)
dig @192.168.56.103 my.lab.test DNSKEY +dnssec

# Kiểm tra RRSIG trên bản ghi A
dig @192.168.56.103 www.my.lab.test A +dnssec
```
---
## SRV11
Vai trò: giữ dữ liệu gốc của zone cha, cho SRV12 đồng bộ, và khai báo delegation của my.lab.test sang SRV2.

- /etc/netplan/srv11.yaml
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      optional: true
      dhcp4-overrides:
        use-dns: false

    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.101/24
      nameservers:
        addresses:
          - 192.168.56.254
          - 8.8.8.8
```
- Tạo thư mục cần thiết: sudo mkdir -p /etc/bind/zones - 
sudo mkdir -p /etc/bind/keys
---
## SRV12

---
## Gateway
Vai trò: nhận truy vấn từ client, chuyển toàn bộ cây lab.test sang SRV11/SRV12, còn truy vấn ngoài Internet
thì dùng forwarders công cộng.

- sudo nano /etc/netplan/Gateway.yaml
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      optional: true
    enp0s8:
      dhcp4: false
      addresses:
      - 192.168.56.254/24
```
- sudo nano /etc/bind/named.conf.options
```
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { trusted; };

    listen-on { 127.0.0.1; 192.168.56.254; };
    listen-on-v6 { none; };

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;
    
    // Tạm thời comment trust anchor cho đến khi có KSK thật
    // managed-keys {
    //     lab.test. initial-key 257 3 8 "BASE64_KSK_CUA_LAB.TEST";
    // };
};
```
- sudo nano /etc/bind/named.conf.local
```
zone "lab.test" {
    type forward;        # Chuyển tiếp toàn bộ zone này
    forward only;        # Chỉ dùng forwarders, không tự hỏi Internet
    forwarders {         # Gửi tới authoritative servers nội bộ
        192.168.56.101;  # Primary của lab.test
        192.168.56.102;  # Secondary của lab.test
    };
};
```
---




