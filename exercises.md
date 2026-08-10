# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Phong  Mã học viên: 2A202601087

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử bạn deploy ứng dụng lên Cloud (Railway/Render) nhưng quên thiết lập biến môi trường `AGENT_API_KEY` trên Dashboard. 
- Nếu để giá trị mặc định `"changeme"`, ứng dụng vẫn khởi chạy bình thường. Tuy nhiên, API key của hệ thống lúc này là `"changeme"`. Các bot quét lỗ hổng tự động trên Internet sẽ dễ dàng thử key phổ biến này và gọi trái phép vào endpoint `/ask`, tiêu tốn toàn bộ chi phí token LLM của bạn mà bạn không hề hay biết.
- Với nguyên tắc Fail Fast: Ngay khi container khởi chạy, lớp `Settings()` không tìm thấy `AGENT_API_KEY` sẽ tung exception dứt khoát và dừng ứng dụng ngay lập tức (CrashLoopBackOff). Bạn sẽ lập tức thấy log lỗi trên dashboard và bổ sung biến môi trường trước khi ứng dụng tiếp nhận bất kỳ traffic thực tế nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"timestamp": "2026-08-10T05:11:15Z", "level": "info", "event": "ask_llm_completed", "user_id": "sv-test", "cost_usd": 0.00004995, "tokens": {"in": 145, "out": 47}}`

Hai việc làm được với log JSON:
1. **Truy vấn & Lọc chỉ số tự động trên Log Management (Datadog/Elasticsearch/Loki)**: Dễ dàng lọc chính xác tất cả log có `event = ask_llm_completed` và `cost_usd > 0.0001` hoặc tính tổng lượng token tiêu thụ theo từng `user_id` mà không cần viết biểu thức chính quy (Regex) phức tạp để parse chuỗi văn bản không có cấu trúc.
2. **Thiết lập cảnh báo tự động (Automated Alerting)**: Cấu hình luật cảnh báo trên Grafana/Datadog để tự động gửi thông báo về Slack/Telegram cho đội ngũ kỹ thuật khi phát hiện `level == "error"` hoặc khi `cost_usd` của một người dùng tăng vọt bất thường.

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
| 1 stage (bản đầu) | ~450 MB |
| Multi-stage | ~180 MB |

Giải thích: Phần dung lượng chênh lệch (~270 MB) là bộ trình biên dịch `gcc`, các công cụ build C/C++ (`build-essential`), thư mục wheel cache của `pip` (`/root/.cache`), cũng như các thư viện dev/headers không cần thiết cho quá trình runtime. Kỹ thuật Multi-stage build giúp loại bỏ toàn bộ các công cụ build thừa này, chỉ copy các thư viện Python đã đóng gói từ stage `builder` sang stage `runtime` tinh gọn (`python:3.11-slim`).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa 1 ký tự trong `app/main.py`: Tất cả các layer phía trên bao gồm `FROM`, `WORKDIR`, `COPY requirements.txt .`, và `RUN pip install ...` đều được dùng lại từ **Docker Build Cache** vì file `requirements.txt` không hề thay đổi. Chỉ có layer `COPY . .` và các câu lệnh phía sau nó phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi thay đổi dù chỉ 1 ký tự trong source code (`app/main.py`), layer `COPY . .` sẽ bị vô hiệu hóa cache (cache invalidation). Điều này buộc Docker phải thực thi lại lệnh `RUN pip install` từ đầu, tải lại toàn bộ danh sách thư viện Python tốn rất nhiều thời gian mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện tấn công:
  1. Kẻ tấn công phát hiện lỗ hổng Remote Code Execution (RCE) trong ứng dụng Python (ví dụ: qua deserialization hoặc command injection).
  2. Kẻ tấn công thực thi lệnh shell bên trong container. Do container chạy dưới quyền root (UID 0), kẻ tấn công có toàn quyền đọc/sửa/xóa file hệ thống trong container.
  3. Nếu tồn tại lỗ hổng container breakout hoặc container được mount các thư mục nhạy cảm từ máy host (như `/var/run/docker.sock`), quyền root trong container sẽ giúp kẻ tấn công chiếm toàn quyền kiểm soát hệ thống máy chủ host.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ở bước 2: Lệnh này chuyển quyền thực thi container sang một tài khoản không có đặc quyền (non-root user `appuser` với UID 10001). Dù kẻ tấn công RCE thành công vào container, chúng chỉ có quyền hạn cực kỳ hạn chế, bị cấm truy cập file hệ thống, không thể cài thêm phần mềm độc hại hay thực hiện các hành vi leo leo quyền trên hệ điều hành host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa trong 2 giây liên tiếp: **20 request** (gấp đôi hạn mức 10/phút).
- Cách đạt được con số đó:
  1. Người dùng gửi 10 request vào giây `59` của phút thứ 1 (ví dụ: `00:00:59`). Thuật toán đếm theo phút đồng hồ vẫn ghi nhận 10 request này thuộc về phút thứ 1 (hợp lệ).
  2. Ngay giây tiếp theo `00` của phút thứ 2 (`00:01:00`), bộ đếm đồng hồ tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa ở giây `00` này (hợp lệ cho phút thứ 2).
  => Tổng cộng trong khoảng thời gian 2 giây ngắn ngủi (từ `00:00:59` đến `00:01:00`), người dùng đã gửi thành công 20 request, gây ra đột biến lưu lượng (Spike) làm quá tải hệ thống. Thuật toán Sliding Window ra đời để khắc phục triệt để lỗ hổng ranh giới phút này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  - **Rate Limit**: Giới hạn **tần suất/tốc độ** truy cập trong khoảng thời gian ngắn (ví dụ: tối đa 10 request/phút) nhằm chống tấn công Spam, DoS/DDoS và bảo vệ server khỏi quá tải.
  - **Cost Guard**: Giới hạn **tổng chi phí/tài nguyên** tiêu tốn trong khoảng thời gian dài (ví dụ: tối đa $10.0/tháng) nhằm kiểm soát ngân sách gọi LLM API.

- Tình huống Rate Limit cho qua nhưng Cost Guard chặn: Người dùng gửi chỉ 1 request duy nhất trong phút (thỏa mãn Rate Limit < 10 req/phút), nhưng prompt của request đó chứa nội dung cực kỳ dài (gửi toàn bộ tài liệu 100 trang khiến chi phí token tiêu tốn vượt trần $10.0/tháng). Cost Guard sẽ lập tức chặn request này.
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn: Người dùng gửi dồn dập 15 request liên tiếp trong 3 giây, nhưng mỗi request chỉ hỏi 1 câu ngắn vài từ (tổng chi phí chỉ mất $0.0001, nhỏ hơn nhiều so với ngân sách $10.0). Tuy nhiên, do vượt quá tần suất 10 req/phút, Rate Limit sẽ ngay lập tức chặn lại và trả mã lỗi `429`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp chung trả về mã lỗi `503 Service Unavailable` do không nối được Redis.
3. Bộ điều phối (Orchestrator như Docker Swarm / Kubernetes) thực hiện Liveness Check, nhận được mã lỗi `503` nên kết luận cả 3 container `agent` đều đã bị treo/hỏng (unhealthy).
4. Orchestrator lập tức cưỡng chế dừng (KILL) và khởi động lại (RESTART) toàn bộ cụm 3 container.
5. Trong khi Redis vẫn chưa khắc phục xong, các container mới khởi động lên lại tiếp tục fail health check -> Orchestrator lại tiếp tục KILL và RESTART liên tục, khiến cụm app rơi vào vòng lặp tử thần (`CrashLoopBackOff`), làm ngưng trệ toàn bộ ứng dụng ngay cả với các endpoint tĩnh không phụ thuộc vào Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Nếu lưu lịch sử trong một dict Python (RAM cục bộ), bạn sẽ thấy con số `history_length` nhảy thất thường không đồng nhất giữa các lần gọi API (ví dụ: lần 1 gọi vào Instance A thấy `history_length = 1`, lần 2 rơi vào Instance B thấy `history_length = 1`, lần 3 rơi vào Instance C thấy `history_length = 1`, lần 4 lại rơi vào Instance A thấy `history_length = 2`).
- Nguyên nhân: Do 3 container `agent` chạy ở 3 process độc lập với bộ nhớ RAM tách biệt. Nginx Load Balancer phân phối request luân phiên (Round-Robin) tới các container khác nhau, dẫn đến Agent bị "mất trí nhớ" và đứt đoạn lịch sử hội thoại của người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: Lỗi `$PORT is not a valid integer` khi khởi chạy uvicorn trên Railway, dẫn đến container bị crash ngay sau khi build xong.
- Nguyên nhân tìm được: Khi xem tab **Logs** trên Railway Dashboard, phát hiện Railway mặc định tự thêm tham số `--port $PORT` vào lệnh khởi chạy. Do shell chưa expand biến môi trường `$PORT`, Uvicorn nhận chuỗi ký tự chữ `'$PORT'` thay vì một số nguyên đại diện cho cổng (ví dụ: `8000`).
- Cách khắc phục:
  1. Tạo file `railway.toml` ở gốc repository với nội dung `startCommand = "python -m app.main"` để chỉ định lệnh khởi chạy chuẩn.
  2. Sửa đoạn code đọc cổng trong `app/main.py` sử dụng `int(os.getenv("PORT", 8000))` để parse biến `$PORT` thành số nguyên an toàn trước khi truyền vào `uvicorn.run()`.
