# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Thành Đạt  Mã học viên: 2A202602721

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu đặt giá trị mặc định là `"changeme"`, khi deploy lên production mà DevOps quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường. Kẻ tấn công trên mạng chỉ cần gửi key `"changeme"` là vượt qua lớp xác thực và gọi thoải mái vào endpoint `/ask`, làm lộ dữ liệu và đốt sạch tiền token LLM. Việc app "chết sớm" (fail-fast) buộc quy trình deploy phải dừng ngay lập tức, ngăn ngừa việc đưa một dịch vụ không có bảo mật lên Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05, "timestamp": "2026-09-29T05:02:42.501234+00:00", "level": "info"}`
> 
> Hai việc làm được:
> 1. Log aggregator (như Datadog, ELK, CloudWatch) tự động phân tích (parse) các trường dữ liệu có cấu trúc, cho phép tìm kiếm và lọc chính xác theo `user_id`, tính tổng token hoặc chi phí `cost_usd` theo từng người dùng.
> 2. Thiết lập hệ thống giám sát và cảnh báo tự động: dashboard có thể vẽ biểu đồ chi phí LLM theo thời gian thực hoặc kích hoạt alert khi mức tiêu thụ token/request tăng đột biến.

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
| 1 stage (bản đầu) | ~1.01 GB |
| Multi-stage | ~195 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~800 MB) gồm:
> - Base image ban đầu là `python:3.11` đầy đủ chứa trình biên dịch C/C++ (`gcc`, `g++`, `make`), công cụ build, và các header files hệ thống.
> - Các file tạm trong quá trình cài đặt: bộ nhớ đệm của pip (`~/.cache/pip`), các file wheel trung gian sau khi compile.
> - Bản multi-stage chuyển sang `python:3.11-slim` và chỉ copy thư mục virtualenv `/opt/venv` đã build xong sang stage runtime, loại bỏ hoàn toàn build tools và cache thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với thứ tự hiện tại, các layer cài đặt thư viện (`COPY requirements.txt .` và `RUN pip install ...`) được dùng lại hoàn toàn từ cache (`CACHED`). Chỉ layer `COPY --chown=appuser . .` và các lệnh phía sau mới phải chạy lại.
> Nếu đặt `COPY . .` trước `RUN pip install`, mỗi khi sửa dù chỉ 1 ký tự trong source code, checksum của layer `COPY . .` sẽ đổi làm vô hiệu hóa toàn bộ cache của các layer phía sau. Docker sẽ bị buộc phải tải và cài đặt lại toàn bộ thư viện từ `requirements.txt` mỗi lần build, khiến thời gian build kéo dài từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
> 1. Code Python có lỗ hổng (RCE hoặc command injection) cho phép kẻ tấn công thực thi shell command tùy ý.
> 2. Vì tiến trình chạy bằng `root` trong container (UID 0), kẻ tấn công khai thác tiếp lỗ hổng kernel của host hoặc kỹ thuật container escape (như mount socket `/var/run/docker.sock`, khai thác `/proc`, hoặc lỗi runc) để thoát khỏi container.
> 3. Khi thoát ra host, vì UID trong container là 0 nên process trên host cũng sở hữu đặc quyền root, chiếm trọn quyền kiểm soát máy chủ.
> Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: process chạy với UID không đặc quyền (>1000). Kẻ tấn công chỉ có quyền của user thường, không thể truy cập tài nguyên bảo vệ của hệ điều hành và không đủ thẩm quyền để thực hiện container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
> Cách thực hiện:
> - Người dùng gửi 10 request vào giây `10:00:59` (cuối phút thứ nhất). Hạn mức 10 req của phút đó được sử dụng hết.
> - Đúng 1 giây sau, lúc `10:01:00`, bộ đếm theo phút đồng hồ tự động reset về 0. Người dùng gửi tiếp 10 request nữa.
> - Tổng cộng có 20 request được gửi trong khoảng thời gian từ `10:00:59` đến `10:01:01` (chỉ 2 giây) mà không bị chặn, gây spike gấp đôi hạn mức lên hệ thống backend.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Điểm khác biệt:
> - Rate limiter kiểm soát tần suất số lượng request (request count/minute) để bảo vệ server khỏi nghẽn mạng và quá tải tài nguyên (DDoS/traffic spike).
> - Cost guard kiểm soát tổng chi phí tài chính (USD budget/month) dựa trên số lượng token LLM thực tế tiêu thụ.
> 
> Tình huống:
> - Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 1 request duy nhất trong ngày nhưng kèm file văn bản dài chứa 150.000 tokens tốn ~$3.00, trong khi hạn mức tháng chỉ còn $0.50. Rate limit cho qua (1 req < 10 req/min), nhưng Cost guard chặn và trả về 402.
> - Cost guard cho qua nhưng Rate limit chặn: User gửi liên tiếp 15 request cực ngắn (mỗi câu chỉ 2 từ, tốn $0.0001) trong 5 giây. Chi phí phát sinh không đáng kể nên Cost guard cho phép, nhưng Rate limit phát hiện vượt quá 10 req/min và chặn lại với mã 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis gặp sự cố mất kết nối trong 30 giây.
> 2. Cả 3 container gọi kiểm tra Redis qua endpoint gộp chung và đồng loạt trả về lỗi 503.
> 3. Orchestrator (Docker/Kubernetes) thấy liveness probe (`/health`) thất bại nên phán đoán cả 3 container ứng dụng đã bị chết (hung/deadlock).
> 4. Orchestrator lập tức cưỡng chế restart/kill cả 3 container.
> 5. Tất cả các request đang xử lý dở bị ngắt đột ngột (502 Bad Gateway tới người dùng).
> 6. Khi 3 container mới khởi động lại, Redis vẫn đang sập trong 30 giây đó, khiến container mới lại tiếp tục fail liveness probe và rơi vào chu kỳ Restart Loop (CrashLoopBackOff), làm tê liệt hoàn toàn hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong dict Python (in-memory RAM của từng process):
> Vì có 3 instance container chạy song song sau load balancer, mỗi request mới sẽ được chia ngẫu nhiên (round-robin) vào instance 1, 2 hoặc 3.
> Người dùng sẽ thấy `history_length` nhảy hỗn loạn và không liên tục: ví dụ request 1 vào container 1 (length=0), request 2 vào container 2 (length=0), request 3 vào container 1 (length=2), request 4 vào container 3 (length=0). Agent bị "mất trí nhớ" và không thể theo dõi liền mạch ngữ cảnh hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi:
> Endpoint `/ready` trả về `HTTP 500 Internal Server Error`.
> 
> Cách tìm nguyên nhân:
> Sử dụng lệnh `railway logs --service agent` để kiểm tra log chi tiết của container, thấy traceback:
> `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings`
> `agent_api_key: Field required`
> Phát hiện nguyên nhân là class `Settings` yêu cầu `agent_api_key` nhưng trên service `agent` của Railway chưa được khai báo biến môi trường này.
> 
> Cách sửa:
> Sử dụng Railway CLI để cấu hình đầy đủ biến môi trường:
> `railway variable set --service agent AGENT_API_KEY="..." REDIS_URL='${{Redis.REDIS_URL}}' ...`
> Sau khi biến môi trường được cập nhật, Railway tự động redeploy và `/ready` trả về `200 OK {"status":"ready","redis":true}`.
