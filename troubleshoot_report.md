# Bài 5 – Chẩn đoán và Xử lý Xung đột Cổng Mạng (Address Already in Use)

## 1. Tình huống

Khi khởi chạy dịch vụ `spring-app.service` (cổng 8082), hệ thống báo lỗi:

```
ERROR: Address already in use (Bind failed) - Port 8082
```

Nguyên nhân: Một tiến trình khác đang chiếm dụng cổng 8082.

---

## 2. Bước 1 – Giả lập cổng bị chiếm dụng

```bash
# Chạy tiến trình Python HTTP Server chiếm cổng 8082 ở chế độ nền
python3 -m http.server 8082 > /dev/null 2>&1 &
```

**Kết quả:** Tiến trình Python được chạy ẩn dưới nền, chiếm giữ cổng 8082.

---

## 3. Bước 2 – Chẩn đoán: Tìm tiến trình chiếm cổng 8082

### Cách 1: Sử dụng `ss`

```bash
sudo ss -tlnp | grep 8082
```

**Kết quả:**

```
LISTEN  0  5  0.0.0.0:8082  0.0.0.0:*  users:(("python3",pid=12345,fd=3))
```

### Cách 2: Sử dụng `lsof`

```bash
sudo lsof -i :8082
```

**Kết quả:**

```
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3  12345  root   3u  IPv4 123456      0t0  TCP *:8082 (LISTEN)
```

### Cách 3: Sử dụng `netstat`

```bash
sudo netstat -tlnp | grep 8082
```

**Kết quả:**

```
tcp  0  0  0.0.0.0:8082  0.0.0.0:*  LISTEN  12345/python3
```

### Thông tin tiến trình tìm được

| Thông tin | Giá trị |
|-----------|---------|
| PID | 12345 |
| Tên tiến trình | python3 |
| Lệnh đầy đủ | python3 -m http.server 8082 |
| Cổng chiếm dụng | 8082 |
| Trạng thái | LISTEN |

> ⚠️ **Lưu ý:** PID thực tế sẽ khác khi chạy trên máy. Thay `12345` bằng PID thực tế từ kết quả lệnh `ss` hoặc `lsof`.

---

## 4. Bước 3 – Xử lý: Kết thúc tiến trình chiếm cổng

```bash
# Thử kill nhẹ nhàng trước (SIGTERM)
kill 12345

# Kiểm tra lại xem tiến trình còn chạy không
sudo ss -tlnp | grep 8082

# Nếu tiến trình vẫn còn, dùng kill -9 (SIGKILL) - buộc dừng ngay lập tức
kill -9 12345

# Xác nhận cổng đã được giải phóng
sudo ss -tlnp | grep 8082
# (không có kết quả trả về = cổng đã trống)
```

**Giải thích sự khác biệt giữa `kill` và `kill -9`:**

| Lệnh | Tín hiệu | Mô tả |
|-------|-----------|-------|
| `kill <PID>` | SIGTERM (15) | Yêu cầu tiến trình tự kết thúc một cách sạch sẽ (graceful shutdown) |
| `kill -9 <PID>` | SIGKILL (9) | Buộc kernel dừng tiến trình ngay lập tức, không cho cleanup |

> 💡 Luôn thử `kill` trước. Chỉ dùng `kill -9` khi tiến trình không phản hồi.

---

## 5. Bước 4 – Khởi động lại dịch vụ Spring Boot

```bash
# Khởi động lại dịch vụ spring-app
sudo systemctl restart spring-app.service

# Kiểm tra trạng thái dịch vụ
sudo systemctl status spring-app.service
```

**Kết quả mong đợi:**

```
● spring-app.service - Spring Boot Application (app.jar) on port 8082
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; ...)
     Active: active (running) since ...
   Main PID: 54321 (java)
```

---

## 6. Bước 5 – Xác nhận kết quả cuối cùng

```bash
sudo ss -tlnp | grep 8082
```

**Kết quả mong đợi:**

```
LISTEN  0  100  *:8082  *:*  users:(("java",pid=54321,fd=XX))
```

> ✅ **Kết luận:** Cổng 8082 hiện đang được lắng nghe bởi tiến trình `java` (app.jar), không phải tiến trình giả lập `python3` ban đầu. Vấn đề xung đột cổng đã được giải quyết thành công.

---

## 7. Tóm tắt quy trình chẩn đoán

```
Phát hiện lỗi "Address already in use"
        │
        ▼
Chẩn đoán bằng ss/lsof/netstat → Tìm PID
        │
        ▼
Ghi nhận thông tin tiến trình chiếm cổng
        │
        ▼
kill <PID> → Nếu không hiệu quả → kill -9 <PID>
        │
        ▼
Xác nhận cổng đã giải phóng (ss -tlnp)
        │
        ▼
Khởi động lại dịch vụ (systemctl restart)
        │
        ▼
Xác nhận dịch vụ hoạt động bình thường ✅
```
