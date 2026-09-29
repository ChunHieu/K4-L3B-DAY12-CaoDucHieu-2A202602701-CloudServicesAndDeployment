# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Các câu trả lời dưới đây dựa trên kết quả em trực tiếp quan sát
> khi cài đặt, chạy test, build container và deploy service trong bài lab.
>
> Họ và tên: **Cao Đức Hiếu**
> Mã học viên: **2A202602701**

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc này hữu ích.

> Khi tạo service mới trên Render, em có thể quên khai báo `AGENT_API_KEY`. Fail fast
> làm lần deploy đó lỗi ngay trong log khởi động, trước khi service nhận traffic. Nếu dùng
> khóa mặc định `changeme`, health check vẫn có thể xanh và người ngoài đoán được khóa để
> gọi `/ask`, làm phát sinh chi phí. Lỗi cấu hình sớm vì vậy dễ phát hiện và an toàn hơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Một dòng log JSON thu được khi gọi `/ask`:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-09-29T04:56:36.055682+00:00","user_id":"reflection-test","tokens_in":6,"tokens_out":38,"cost_usd":0.0000237}
```

> Với log này em có thể lọc/đếm các event `ask_completed` theo `user_id`, đồng thời
> tổng hợp `cost_usd` hoặc số token để theo dõi ngân sách và đặt cảnh báo. Một chuỗi
> `print("đã trả lời xong")` không có trường dữ liệu ổn định để hệ thống log truy vấn,
> nhóm hoặc tính tổng tự động.

---

### Câu 3 — Kích thước image (CP2)

| Bản | Dung lượng |
|-----|-----------:|
| 1 stage, base `python:3.11` đầy đủ | khoảng 1 GB; lần build đo thử bị dừng khi đang tải các layer base lớn |
| Multi-stage, runtime `python:3.11-slim` | 271 MB |

> Lệnh `docker images day12-agent:prod` cho thấy image cuối là 271 MB. Khi thử build
> lại starter một stage, riêng các layer nén của base đầy đủ đã phải tải khoảng 334 MB
> (trong đó có layer 236 MB), nên quá trình tải quá chậm và em đã dừng. Phần chênh lệch
> chủ yếu là hệ điều hành/toolchain của image Python đầy đủ và các thành phần chỉ cần
> trong lúc build. Runtime slim chỉ nhận dependency đã cài cùng source nên nhỏ hơn rõ rệt.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

> Dockerfile copy `requirements.txt` và chạy `pip install` trước khi copy `app/` và
> `utils/`. Vì vậy khi chỉ sửa một ký tự trong `app/main.py`, layer base, layer copy
> requirements và layer cài dependency vẫn lấy từ cache; chỉ các layer copy source trở
> đi phải chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source làm
> invalid cache và Docker phải cài lại toàn bộ dependency, khiến build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước hết chiếm quyền
> trong process của container. Khi process chạy root, chúng có quyền sửa file hệ thống
> trong container và có thể lợi dụng volume, socket Docker hoặc lỗi runtime để tác động
> tới host với quyền cao. Lệnh `USER appuser` cắt chuỗi ở bước sau khi chiếm process:
> mã độc chỉ có UID không đặc quyền, nên phạm vi đọc/ghi và hậu quả bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

> Với bộ đếm theo phút đồng hồ và hạn mức 10/phút, user có thể gửi 10 request ở
> 10:00:59 rồi thêm 10 request ngay tại 10:01:00. Như vậy có tối đa 20 request trong
> khoảng 2 giây nhưng mỗi phút lịch vẫn chỉ ghi 10. Sliding window 60 giây nhìn lại đúng
> 60 giây gần nhất nên loạt thứ hai bị chặn sau khi quota của loạt đầu đã dùng hết.

---

### Câu 7 — Rate limit và cost guard (CP3)

> Rate limit bảo vệ tốc độ/số request trong 60 giây, còn cost guard bảo vệ tổng tiền
> theo user trong tháng UTC. Một user gửi ít request nhưng prompt rất lớn có thể không
> vượt rate limit nhưng vượt ngân sách và bị cost guard trả 402. Ngược lại, user gửi
> nhiều câu rất ngắn trong vài giây khi ngân sách còn nhiều sẽ bị rate limiter trả 429,
> dù cost guard vẫn cho qua nếu chỉ xét chi phí.

---

### Câu 8 — `/health` khác `/ready` (CP4)

> Nếu gộp hai endpoint và cho health check phụ thuộc Redis, khi Redis mất kết nối thì
> cả ba container cùng trả 503. Orchestrator coi cả ba process bị hỏng và lần lượt restart
> chúng, dù code ứng dụng vẫn sống. Redis vẫn chưa phục hồi nên container mới tiếp tục
> fail, tạo vòng lặp restart và làm mất toàn bộ khả năng phục vụ. Khi tách riêng, `/health`
> vẫn 200 nên container không bị restart; `/ready` trả 503 để load balancer tạm ngừng gửi
> request. Redis phục hồi thì readiness tự xanh và traffic quay lại.

---

### Câu 9 — Stateless (CP4)

> Compose hiện map cố định `8000:8000`, nên scale trực tiếp ba replica trên cùng host sẽ
> xung đột cổng nếu chưa đặt Nginx phía trước. Em kiểm chứng tính stateless bằng test tạo
> hai `ConversationStore` độc lập dùng chung một Redis: instance B đọc được message do
> instance A ghi, và request thứ hai thấy `history_length=2`. Với dict Python, mỗi replica
> có lịch sử riêng; request bị phân phối luân phiên sẽ cho độ dài nhảy không đều như
> `0, 0, 0, 2, 2, 2` thay vì tăng thống nhất `0, 2, 4, ...`.

---

### Câu 10 — Deploy thật (CP5)

> Khi kiểm tra service Render, một lần `pytest tests/test_cp5.py` báo
> `ConnectError: getaddrinfo failed` cho URL `onrender.com`, dù dashboard báo Live. Em
> đối chiếu bằng cách mở `/health` trên trình duyệt và dùng `curl -v`; cả `/health` và
> `/ready` đều trả 200, nên xác định đây là lỗi DNS tạm thời ở môi trường chạy test chứ
> không phải app hoặc `REDIS_URL`. Em chạy lại test từ terminal máy sau khi DNS hoạt động;
> kết quả CP5 là 9 passed, 4 test local fallback được skip vì em deploy cloud thật.
