# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phan Hoàng Vũ  Mã học viên: 2A202602450

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy service lên môi trường Cloud (như Render hoặc Railway), lập trình viên rất dễ quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. Nếu ta đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường và probe báo healthy, nhưng vô tình mở toang cổng API cho bất kỳ ai hoặc bot tự động quét trên Internet sử dụng khóa `"changeme"` để gọi LLM miễn phí. Hậu quả là hóa đơn dịch vụ tăng vọt mà ta không hề hay biết cho tới cuối tháng. Ngược lại, khi không có giá trị mặc định, cơ chế Fail Fast khiến app dừng ngay lập tức (`ValidationError`) trong bước deploy, buộc ta phát hiện ra sự thiếu sót và set secret bảo mật ngay khi còn đang theo dõi deploy log.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T14:52:05.123456+00:00", "user_id": "sv-test", "tokens_in": 5, "tokens_out": 12, "cost_usd": 0.0001}`
>
> Hai việc làm được với log JSON mà `print` văn bản không làm được:
> 1. **Tổng hợp và tính toán số liệu tự động (Aggregation & Metrics):** Các hệ thống giám sát tập trung (Datadog, Grafana Loki, CloudWatch) có thể tự động bóc tách các trường JSON để tính toán tức thời: ví dụ tính tổng chi phí theo từng user `SUM(cost_usd) GROUP BY user_id`, hoặc thống kê lượng token trung bình tiêu thụ trong 1 giờ qua.
> 2. **Thiết lập cảnh báo tự động (Alerting):** Hệ thống có thể đặt bộ lọc trực tiếp trên dữ liệu có cấu trúc, ví dụ tự động kích hoạt cảnh báo gửi về Slack/PagerDuty khi `cost_usd > 0.5` trong 1 request hoặc khi số event có `level == "error"` vượt quá 5 lần/phút mà không phải viết regex xử lý chuỗi văn bản không ổn định.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch gần 750 MB bao gồm:
> 1. Sự khác biệt giữa base image đầy đủ (`python:3.11`) và bản rút gọn (`python:3.11-slim`): image gốc chứa toàn bộ các gói hệ thống Linux, các trình biên dịch C/C++ (`gcc`, `g++`, `make`), công cụ build và các file header không cần thiết cho môi trường chạy ứng dụng.
> 2. Quá trình multi-stage build: Stage `builder` chứa toàn bộ pip cache, wheel build files và các dependency tạm phục vụ cài đặt; còn stage `runtime` chỉ copy thành phẩm cuối cùng sang `/usr/local`, giúp image sạch và nhẹ tối đa, đẩy nhanh tốc độ pull/deploy trên cloud.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile tối ưu hiện tại: các layer khởi tạo môi trường (`FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`) đều được tái sử dụng từ cache (`CACHED`). Chỉ các layer từ `COPY . .` trở đi (và `RUN chown`, `USER`, `CMD`) mới phải chạy lại, quá trình build chỉ mất 1–2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Docker tính toán cache theo từng layer từ trên xuống dưới. Mỗi khi ta sửa dù chỉ 1 ký tự trong `app/main.py`, layer `COPY . .` sẽ bị invalid cache, kéo theo toàn bộ các layer phía sau nó (bao gồm cả `RUN pip install`) bị ép chạy lại từ đầu. Kết quả là mỗi lần sửa code, Docker phải tải và cài lại toàn bộ thư viện từ Internet, làm thời gian build kéo dài thêm vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện leo thang:
> 1. Kẻ tấn công phát hiện một lỗ hổng trong code (ví dụ: Remote Code Execution qua lỗ hổng Command Injection hoặc Deserialization không an toàn).
> 2. Kẻ tấn công khai thác lỗ hổng để thực thi shell payload bên trong container.
> 3. Vì container mặc định chạy dưới quyền root (UID 0), kẻ tấn công chiếm quyền root trong container, có thể chỉnh sửa file hệ thống và cài thêm các công cụ exploit.
> 4. Kẻ tấn công lợi dụng một lỗ hổng container escape (như cgroup v1 release_agent, lỗ hổng nhân Linux, hoặc việc mount docker socket `/var/run/docker.sock`) để thoát khỏi ranh giới container.
> 5. Do UID 0 trong container mặc định ánh xạ với UID 0 trên máy host, kẻ tấn công chiếm toàn quyền root cao nhất trên máy chủ vật lý/host.
>
> Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi ngay tại Bước 3: kẻ tấn công sau khi khai thác chỉ có quyền của user thường, không thể can thiệp file nhạy cảm, không có quyền can thiệp kernel hay thiết bị, khiến nỗ lực container escape bị vô hiệu hóa.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 requests** trong 2 giây liên tiếp.
> Cách đạt được: Người dùng căn thời gian gửi 10 request vào giây cuối cùng của phút thứ nhất (lúc `10:00:59`), hệ thống ghi nhận đủ quota 10 request của phút 10:00. Ngay 1 giây sau, khi đồng hồ chuyển sang `10:01:00`, bộ đếm theo phút được reset về 0; người dùng lập tức gửi tiếp 10 request nữa lúc `10:01:01`. Cả hai đợt đều hợp lệ theo cách đếm phút đồng hồ, nhưng thực tế đã có 20 request được gửi dồn dập chỉ trong 2 giây. Cửa sổ trượt 60 giây giải quyết triệt để lỗ hổng này vì nó luôn tính tổng request trong khoảng `[now - 60s, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau:
> - **Rate limit:** Giới hạn **tần suất số lượng request** theo thời gian ngắn (ví dụ: 10 request/phút) để ngăn server bị cạn kiệt CPU/RAM và chống nghẽn đường truyền.
> - **Cost guard:** Giới hạn **tổng chi phí tài chính (USD)** tích lũy trong chu kỳ dài (ví dụ: 10 USD/tháng) để tránh thâm hụt ngân sách tiền túi.
>
> Tình huống:
> - *Rate limit cho qua nhưng Cost guard chặn:* Một user gọi API rất chậm rãi, chỉ 1 request sau mỗi 5 phút (hoàn toàn thỏa mãn < 10 req/phút). Nhưng user đó gửi một prompt khổng lồ kèm tài liệu 100.000 tokens trị giá 0.5 USD, trong khi ngân sách tháng của user đã tiêu hết 9.8 / 10.0 USD. Cost guard phát hiện vượt trần ngân sách nên ném lỗi 402 Payment Required.
> - *Cost guard cho qua nhưng Rate limit chặn:* Một user mới dùng đầu tháng còn nguyên 10.0 USD ngân sách, nhưng viết script gửi 30 câu hỏi liên tiếp trong vòng 3 giây. Dù 30 câu hỏi ngắn chỉ tốn 0.003 USD (chưa ảnh hưởng ngân sách tháng), Rate limit vẫn phải chặn ngay lập tức (lỗi 429) để bảo vệ server khỏi bị treo.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Redis gặp sự cố mạng tạm thời hoặc khởi động lại trong vòng 30 giây.
> 2. Bộ điều phối (Docker Compose, Kubernetes hoặc Cloud Platform) gửi probe kiểm tra định kỳ (liveness) vào endpoint gộp.
> 3. Vì Redis mất kết nối, endpoint của cả 3 container agent đều trả về mã lỗi 503 hoặc timeout.
> 4. Bộ điều phối nhận diện rằng cả 3 container đều "đã chết" (unhealthy) và tự động phát lệnh `SIGKILL` để restart toàn bộ 3 container cùng một lúc.
> 5. Cả 3 container mới khởi động lại và tiếp tục gọi kiểm tra Redis, lúc này Redis vẫn chưa kịp phục hồi xong, dẫn đến container tiếp tục bị fail probe và rơi vào vòng xoáy restart liên hoàn (CrashLoopBackOff).
> 6. Khi Redis đã hoạt động trở lại sau 30 giây, toàn bộ 3 container đều đang bị khởi động lại dở dang, khiến toàn bộ hệ thống bị sập hoàn toàn (outage) từ một sự cố phụ thuộc tạm thời.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Khi lưu trong Redis (Stateless): `history_length` tăng đều đặn qua từng lượt gọi: `0 -> 2 -> 4 -> 6...` bất kể request được bộ cân bằng tải phân phối đến container nào trong cụm 3 instance.
> - Nếu lưu trong một `dict` Python của RAM (Stateful): Vì mỗi container chạy trong một tiến trình riêng và có vùng nhớ RAM độc lập, khi Load Balancer phân phối request theo cơ chế Round Robin hoặc ngẫu nhiên, ta sẽ thấy `history_length` nhảy hỗn loạn và không nhất quán. Ví dụ: gọi lần 1 vào Container A (`history_length = 0`), gọi lần 2 rơi vào Container B (`history_length` vẫn là `0` vì B không có dữ liệu của A), gọi lần 3 lại vào Container A (`history_length = 2`), gọi lần 4 vào Container C (`history_length = 0`). Agent sẽ rơi vào tình trạng "mất trí nhớ" chập chờn tùy theo container nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải:** Lỗi xung đột cổng `Bind for 0.0.0.0:8000 failed: port is already allocated` khi thử nghiệm scale ngang trên máy cục bộ hoặc khi deploy lên cloud mà không đọc biến môi trường dynamic port.
> - **Thông báo lỗi:** `driver failed programming external connectivity on endpoint ...: Bind for 0.0.0.0:8000 failed: port is already allocated`
> - **Cách tìm ra nguyên nhân:** Đọc log khởi tạo của Docker Compose và nhận thấy trong file `docker-compose.yml`, service `agent` đang gán cố định cổng host `8000:8000`. Khi tạo 3 replica, container đầu tiên đã chiếm cổng 8000 của host, 2 container tiếp theo cố gắng bind vào cùng một cổng mạng nên bị hệ điều hành từ chối.
> - **Cách sửa:**
>   1. Trong `Dockerfile`, cho phép uvicorn đọc cổng động do môi trường cấp bằng lệnh `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để khi deploy lên Render/Railway (nơi platform tự gán `$PORT`), app tự động bind đúng cổng được chỉ định.
>   2. Với Docker Compose đa container, bỏ việc bind cổng host trực tiếp của từng replica mà đưa tất cả đứng sau một Reverse Proxy (như Nginx) để tiếp nhận duy nhất cổng 80/8000 từ bên ngoài và cân bằng tải nội bộ tới các container agent.
