# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng câu hỏi bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Việt Anh  Mã học viên: 2A202601107

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên production, nếu để giá trị mặc định `"changeme"` và quên cấu hình `AGENT_API_KEY` trên Cloud dashboard, ứng dụng vẫn sẽ khởi động bình thường. Tuy nhiên, bất kỳ ai dò ra API key mặc định `"changeme"` cũng có thể gọi API của bạn hoàn toàn miễn phí, làm rò rỉ dữ liệu hoặc đốt sạch chi phí LLM trước khi bạn phát hiện ra. Việc "fail fast" không cho phép giá trị mặc định giúp báo lỗi `ValidationError` ngay lúc khởi động, bắt buộc dev/ops phải cấu hình key chính xác trước khi ứng dụng đi vào hoạt động.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T11:45:00+00:00", "user_id": "sv-test", "cost_usd": 0.00001995, "tokens_in": 1, "tokens_out": 33}`

Hai việc làm được với log JSON:
1. Các hệ thống quản lý log (như Datadog, CloudWatch, ELK) có thể tự động parse các trường structured data (`user_id`, `cost_usd`, `tokens_in`) để tính tổng chi phí theo ngày hoặc vẽ biểu đồ thời gian thực.
2. Dễ dàng thiết lập quy tắc cảnh báo (Alerting) tự động khi tỉ lệ lỗi tăng cao hoặc lọc danh sách người dùng tiêu tốn chi phí nhiều nhất.

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
| Multi-stage | ~215 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800MB) bao gồm các công cụ biên dịch (GCC, build-essential), header files, pip cache và các file tạm được sinh ra trong quá trình cài đặt thư viện ở stage `builder`. Nhờ multi-stage build, stage `runtime` chỉ copy phần kết quả thư viện đã biên dịch sang base image `slim`, giúp loại bỏ hoàn toàn các công cụ thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 dòng code trong `app/main.py`:
- Layer `COPY requirements.txt` và `RUN pip install` được dùng lại hoàn toàn từ cache.
- Layer `COPY app ./app` và các lệnh phía sau phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ dòng code nào, layer `COPY . .` bị thay đổi làm vô hiệu hóa cache của toàn bộ các lệnh phía sau, bắt buộc Docker phải tải và cài đặt lại toàn bộ thư viện Python qua `pip install`, làm kéo dài thời gian build từ vài giây lên nhiều phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong code Python -> thực thi lệnh và chiếm quyền điều khiển container. Nếu container chạy bằng root, kẻ tấn công có quyền root trong container và có thể khai thác lỗ hổng hổng container escape để lấy quyền root trên máy host.
Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay trong container: kẻ tấn công chỉ thu được quyền của một user thông thường không có đặc quyền (`appuser`), không thể đọc/ghi file hệ thống hoặc thực hiện hành vi leo leo quyền lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được: Người dùng gửi 10 request ở giây 10:00:59 (cuối phút thứ 1) và tiếp tục gửi 10 request ở giây 10:01:01 (đầu phút thứ 2). Cả hai đợt đều nằm trong hạn mức 10/phút của từng phút riêng biệt, nhưng tính trong cửa sổ trượt 2 giây thì người dùng đã gửi tới 20 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau: Rate limit giới hạn *tần suất/số lượng request* trong một khoảng thời gian ngắn (ví dụ 10 req/phút). Cost guard giới hạn *tổng chi phí tài chính* (ngân sách) theo tháng.
- Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 2 request/phút (dưới hạn mức 10 req/phút), nhưng mỗi request truyền vào văn bản cực lớn (100k tokens) dẫn tới chi phí vượt quá ngân sách tháng.
- Cost guard cho qua nhưng Rate limit chặn: Đầu tháng user chưa dùng hết ngân sách ($0 / $10), nhưng gửi 15 request liên tục trong 3 giây. Cost guard chưa ném lỗi nhưng Rate limit chặn ngay lỗi 429 để bảo vệ hệ thống khỏi bị quá tải.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối trong 30 giây.
2. Endpoint `/health` kiểm tra Redis và trả về mã lỗi 503/500.
3. Orchestrator thấy `/health` báo lỗi nên đánh giá container đã hỏng (unhealthy) và tiến hành **restart** toàn bộ 3 container `agent`.
4. Cả 3 container bị dừng cùng lúc để khởi động lại. Khi Redis có kết nối lại, không còn container nào phục vụ request, biến một sự cố tạm thời của Redis thành sự cố sập toàn bộ hệ thống (Cascading Failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python:
Mỗi request gửi tới sẽ được Load Balancer điều phối ngẫu nhiên vào 1 trong 3 container (A, B hoặc C).
Giá trị `history_length` sẽ thay đổi ngẫu nhiên và không đồng nhất (ví dụ: 0 -> 2 -> 0 -> 2 -> 4), vì thông tin lịch sử chỉ nằm trong bộ nhớ RAM riêng của từng instance mà không được chia sẻ chung.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Error: Invalid value for '--port': '$PORT' is not a valid integer.`
- Cách tìm nguyên nhân: Mở tab **Deploy Logs** trên Railway Dashboard, quan sát thấy lệnh khởi động `uvicorn` nhận tham số `--port $PORT` dưới dạng chuỗi thô `"$PORT"` do không được mở rộng biến môi trường trong shell.
- Cách sửa: Cập nhật file `railway.toml` bọc lệnh khởi động qua `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` để shell tự động giải mã `$PORT` thành số cổng thật do Railway gán.
