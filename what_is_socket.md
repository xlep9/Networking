| Thành phần | Vai trò                                                                  |
| ---------- | ------------------------------------------------------------------------ |
| IP         | xác định máy                                                             |
| Port       | xác định chương trình                                                    |
| Protocol   | luật giao tiếp                                                           |
| Socket     | object trong kernel dùng để thực hiện giao tiếp theo IP/port/protocol đó |

ví dụ:
socket(AF_INET, SOCK_STREAM, 0);

OS tạo ra:
TCP socket object (chưa có IP/port)

Sau đó: bind(127.0.0.1, 4444) -> Socket lúc này mới “gắn địa chỉ”.

Sau đó: listen() / connect() -> Socket chuyển sang trạng thái hoạt động.

IP/port/protocol là “địa chỉ & luật”

Socket là “thực thể đang dùng địa chỉ đó để giao tiếp”
