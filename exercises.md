# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Đức Đại  Mã học viên: 2A202602725

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định 'changeme', ứng dụng vẫn khởi động trơn tru. Khi triển khai lên môi trường Cloud (production) mà ta quên cài đặt Secret, kẻ tấn công dò được API Key 'changeme' có thể thoải mái gọi API của chúng ta miễn phí, gây cạn kiệt tiền gọi LLM. Việc "chết sớm" (fail fast) giúp hệ thống tự động từ chối khởi động ngay lập tức, cảnh báo ta biết lỗi cấu hình trước khi có bất kỳ ai truy cập vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

{"event": "request_completed", "level": "info", "timestamp": "2026-09-28T16:30:00.123Z", "duration_ms": 150.5, "user_id": "user123", "cost_usd": 0.001}
Hai việc làm được:
1. Dễ dàng truy vấn và lọc dữ liệu bằng máy (VD: tìm mọi request có `duration_ms > 1000` hoặc lọc theo `user_id` cụ thể trên Elasticsearch).
2. Tự động vẽ biểu đồ phân tích (Dashboard) thống kê tổng chi phí `cost_usd` hay thời gian phản hồi trung bình theo thời gian thực mà không cần dùng regex để bóc tách text.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~ 1.1 GB |
| Multi-stage | ~ 180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch khổng lồ đó là những công cụ chỉ phục vụ quá trình build (như GCC, build-essential), mã nguồn tải về tạm thời của các thư viện (cache pip) và các file không cần thiết khác. Phiên bản multi-stage chỉ copy thư mục đã được dịch sẵn (virtual environment hoặc site-packages) sang một base image tối giản (slim/alpine) để chạy, bỏ lại toàn bộ rác phía sau.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Các layer COPY requirements.txt và RUN pip install sẽ lấy thẳng từ bộ nhớ đệm (cache) vì nội dung file requirements.txt không thay đổi. Chỉ từ layer COPY app/ app/ trở đi mới phải chạy lại.
Nếu đặt COPY . . lên trước RUN pip install, mỗi khi ta sửa 1 ký tự trong code Python, layer COPY đó sẽ bị tính là mất cache, kéo theo lệnh RUN pip install đằng sau cũng mất cache và phải tải lại toàn bộ thư viện từ đầu rất tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Nếu app có lỗ hổng (VD: Remote Code Execution do Deserialize), kẻ tấn công chạy được mã độc trên hệ thống. Vì container chạy quyền root, mã độc cũng chạy với quyền root trong container. Nếu Docker cấu hình lỏng lẻo, kẻ tấn công có thể lợi dụng quyền root này để xâm nhập ra máy ảo host (VM escape) và phá hoại hệ thống.
Lệnh USER appuser cắt đứt chuỗi này ngay sau khi bị RCE: Kẻ tấn công chỉ chiếm được quyền của tài khoản `appuser` (quyền thấp), không thể sửa đổi file hệ thống hay thực hiện các kỹ thuật leo thang đặc quyền ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Cách thực hiện: Căn lúc 00:59 giây, gửi 10 request (lúc này chưa hết phút 1 nên vẫn được tính vào hạn mức 10/phút). Đúng 1 giây sau, đồng hồ chuyển sang 01:00, hạn mức bị reset lại từ đầu, người dùng lập tức gửi tiếp 10 request nữa.
Cửa sổ trượt (sliding window) ngăn chặn mánh khóe này vì nó luôn tính tổng request trong đúng 60 giây gần nhất thay vì phụ thuộc vào đồng hồ hệ thống.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate Limit kiểm soát TẦN SUẤT (nhằm chống DDoS, chống sập server do request dồn dập). Cost Guard kiểm soát TỔNG NGÂN SÁCH (nhằm chống cạn tiền, giới hạn chi phí).
- Rate cho qua nhưng Cost chặn: Người dùng thỉnh thoảng mới gửi 1 câu hỏi (không vi phạm rate limit), nhưng họ đã xài app suốt 1 tháng và vượt mốc 10 USD -> Cost Guard chặn (lỗi 402).
- Cost cho qua nhưng Rate chặn: Người dùng vừa tạo tài khoản, chưa tốn xu nào (qua Cost Guard), nhưng dùng tool cào dữ liệu bắn 100 câu hỏi vào API cùng 1 lúc -> Rate Limit chặn (lỗi 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis đứt kết nối. Endpoint /health (đã bị gộp chung kiểm tra Redis) báo lỗi.
2. Hệ thống quản lý (Kubernetes/Render) quét /health thấy lỗi, tưởng rằng bản thân container (App) bị treo/chết.
3. Hệ thống thẳng tay kill container hiện tại và khởi động container mới.
4. Container mới lên, Redis vẫn đang đứt -> /health lại báo lỗi -> Lại bị kill tiếp. Tạo thành vòng lặp CrashLoopBackOff.
Nếu tách ra, /health (App sống) vẫn xanh, /ready (Redis chết) báo đỏ -> Hệ thống chỉ tạm thời ngắt traffic đi vào container chứ không kill nó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu lịch sử trong dict (stateful), vì Load Balancer phân phối request ngẫu nhiên tới 3 container khác nhau (A, B, C), mỗi container lại giữ 1 bản lịch sử cục bộ riêng. Do đó con số `history_length` sẽ nhảy lung tung (trồi sụt thất thường) tùy thuộc vào việc request rơi vào container nào. Nếu dùng Redis (stateless), dữ liệu tập trung ở 1 nơi, nên `history_length` ở container nào cũng sẽ lấy được đoạn hội thoại dài và tăng dần đều.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi: Health check failed: timeout after 10m khi triển khai trên Render.
Nguyên nhân: Mở tab Logs trên Render xem thì phát hiện dòng chữ ValidationError - ứng dụng sập ngay lúc khởi động do thiếu biến môi trường `AGENT_API_KEY`. (Render không tự động lấy biến từ file .env local của máy tính).
Cách sửa: Vào mục Settings > Environment Variables của Web Service trên trang web Render, điền thủ công key `AGENT_API_KEY` và giá trị tương ứng vào, rồi chọn Manual Deploy để hệ thống build lại. Hết lỗi.
