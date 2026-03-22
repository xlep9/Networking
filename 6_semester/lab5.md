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
```
$TTL 604800
@   IN  SOA srv11.lab.test. admin.lab.test. (
        2026032201
        604800
        86400
        2419200
        604800 )

@       IN  NS  srv11.lab.test.
@       IN  NS  srv12.lab.test.

srv11   IN  A   192.168.56.101
srv12   IN  A   192.168.56.102
srv2    IN  A   192.168.56.103

my      IN  NS  srv2.lab.test.
```

















