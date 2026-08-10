# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Minh  Mã học viên: 2A202601972

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy ứng dụng lên Railway/Render, nếu bạn quên thiết lập biến môi trường `AGENT_API_KEY`, việc không có giá trị mặc định sẽ khiến pydantic ném lỗi `ValidationError` và làm ứng dụng crash ngay lập tức ở bước khởi động. Lỗi này hiện ngay trên log deploy của developer khi vừa push code. Nếu để mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động thành công và phục vụ request, kẻ tấn công có thể mò ra key mặc định `"changeme"` để gọi API miễn phí rút kiệt tài khoản LLM của bạn mà bạn chỉ phát hiện ra khi nhận hóa đơn tài chính cuối tháng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log JSON thu được: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T11:28:00+00:00", "user_id": "sv-test", "cost_usd": 0.0001, "tokens_in": 15, "tokens_out": 25}`
> Hai việc làm được với log JSON mà print thường không làm được:
> 1. Lọc và truy vấn chính xác (Queryability & Filtering): Các hệ thống gom log như Datadog/ELK/CloudWatch có thể trích xuất trường `cost_usd` hoặc `user_id` để đếm tổng chi phí theo từng user trong ngày hoặc lọc ra các lỗi theo `level="error"`.
> 2. Tự động cảnh báo (Automated Alerting): Có thể thiết lập rule giám sát tự động kích hoạt alert (gửi Telegram/Slack) khi tổng `cost_usd` vượt quá ngưỡng quy định hoặc tỷ lệ sự kiện lỗi gia tăng.

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
| Multi-stage | ~270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~800MB) bao gồm bộ công cụ biên dịch (GCC, g++, make), các file header C/C++ (`python3-dev`, `build-essential`), bộ nhớ đệm pip build cache, và các thư viện hệ thống thừa không cần thiết ở môi trường runtime. Stage builder đã biên dịch và cài đặt thư viện vào `/install`, sau đó stage runtime chỉ copy thư viện đã đóng gói sang base image `python:3.11-slim`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa 1 ký tự trong `app/main.py` và build lại: các layer `COPY requirements.txt` và `RUN pip install` đứng trước được dùng lại hoàn toàn từ Docker cache (vì file `requirements.txt` không đổi). Layer `COPY app ./app` và các layer phía sau bị invalidate cache và phải chạy lại.
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa bất kỳ ký tự nào trong code, Docker sẽ làm mất cache từ layer `COPY . .` trở đi, dẫn đến việc Docker phải tải và cài đặt lại toàn bộ thư viện pip từ đầu, làm tăng thời gian build từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Mặc định container chạy dưới quyền root. Nếu code Python có lỗ hổng (như Command Injection hoặc Remote Code Execution), kẻ tấn công có thể thực thi lệnh trong container với quyền root. Nếu kẻ tấn công khai thác tiếp lỗ hổng container escape (hoặc volume mount nhầm socket docker), họ sẽ chiếm toàn bộ quyền kiểm soát tối cao root trên hệ thống máy host.
> Lệnh `USER appuser` cắt đứt chuỗi tấn công bằng cách hạ quyền tiến trình xuống user thường (`uid=10001`). Dù kẻ tấn công chiếm được shell trong container, họ chỉ có quyền hạn chế của `appuser`, không thể sửa file hệ thống hoặc leo thang chiếm máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
> Giải thích: Nếu dùng đếm theo phút cố định (reset lúc 00 giây): Người dùng gửi 10 request ở 2 giây cuối của phút thứ nhất (10:00:58 - 10:00:59), sau đó lúc 10:01:00 quota đếm lại về 0, họ gửi tiếp 10 request ở 2 giây đầu của phút thứ hai (10:01:00 - 10:01:01). Cả hai phút họ đều không vi phạm hạn mức 10 req/phút, nhưng thực tế họ đã gửi 20 request trong khoảng 2-3 giây liên tiếp.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request theo đơn vị thời gian ngắn (ví dụ: tối đa 10 request/phút) để chống DDoS/spam làm treo server. Cost guard giới hạn tổng chi phí tài chính (USD) theo chu kỳ dài hơn (theo tháng) để bảo vệ ngân sách API LLM.
> Tình huống Rate limit cho qua nhưng Cost guard chặn: User gửi 1 request duy nhất trong ngày nhưng prompt cực dài (vài trăm ngàn tokens) tiêu tốn 15 USD, vượt quá ngân sách tháng 10 USD. Rate limit thấy 1 req/phút nên cho qua, nhưng Cost guard phát hiện vượt ngân sách nên chặn (HTTP 402).
> Tình huống Cost guard cho qua nhưng Rate limit chặn: User gửi 15 request liên tục trong vòng 5 giây, mỗi request ngắn 5 token (chi phí vô cùng nhỏ chỉ 0.0001 USD). Cost guard kiểm tra ngân sách vẫn còn nhiều nên cho qua, nhưng Rate limit phát hiện vượt quá 10 req/phút nên chặn ngay (HTTP 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Endpoint liveness probe gộp trả về lỗi HTTP 503 do Redis không phản hồi.
> 2. Container Orchestrator (Docker/Kubernetes) cho rằng process ứng dụng đã chết nên phát tín hiệu kill và restart cả 3 container agent.
> 3. Cả 3 container bị restart cùng lúc và không container nào sẵn sàng phục vụ.
> 4. Khi Redis hồi phục sau 30s, cả 3 container vẫn đang trong quá trình boot up ➔ Toàn bộ ứng dụng bị sập hoàn toàn (Cascade Failure), biến một sự cố nhỏ ở dependency thành sự cố ngưng trệ toàn hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Do Load Balancer phân phối các request đến 3 instance container theo cơ chế Round-Robin/Least Connections, câu hỏi thứ 1 rơi vào Container A (RAM A lưu history len = 2), câu hỏi thứ 2 rơi vào Container B (RAM B rỗng, history len = 0), câu hỏi thứ 3 rơi vào Container C (history len = 0), câu hỏi thứ 4 rơi vào Container A (history len = 4)...
> Kết quả: `history_length` trong phản hồi sẽ nhảy nhót ngẫu nhiên không ổn định (0, 2, 0, 4...), agent bị "mất trí nhớ" bất thường vì dữ liệu không được chia sẻ giữa các instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi: Log báo `INFO: Uvicorn running on http://0.0.0.0:8080` nhưng container liên tục bị báo Unhealthy và restart.
> Nguyên nhân: Railway tự động gán biến môi trường `PORT=8080` nên Uvicorn lắng nghe ở cổng 8080. Tuy nhiên câu lệnh `HEALTHCHECK` trong Dockerfile lại ghi cứng truy cập cổng `8000` (`http://127.0.0.1:8000/health`). Do đó healthcheck bị từ chối kết nối (`ConnectionRefusedError`).
> Cách sửa: Cập nhật câu lệnh `HEALTHCHECK` trong `Dockerfile` đọc cổng động từ biến môi trường `os.getenv('PORT', '8000')` để tự động khớp với cổng mà Uvicorn đang chạy.

