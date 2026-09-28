# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy tôi quên khai báo `AGENT_API_KEY`. Nếu cấu hình bắt buộc,
> process dừng ngay khi khởi động và log chỉ thẳng vào biến bị thiếu, nên bản lỗi không
> bao giờ nhận traffic. Nếu dùng mặc định `"changeme"`, health check vẫn xanh và người
> ngoài biết hoặc đoán được khóa mặc định có thể gọi `/ask`, làm phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log quan sát được khi gọi local service:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:14:51.822822+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`.
> Với JSON này tôi có thể (1) lọc/đếm theo `event`, `user_id`, `level` mà không cần
> phân tích chuỗi tự do và (2) tổng hợp `cost_usd`, token thành metric/cảnh báo. Một
> dòng `print("đã trả lời xong")` không mang các trường có cấu trúc để làm hai việc đó.

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
| 1 stage (bản đầu) | Chưa đo được — máy hiện tại không có Docker CLI/daemon |
| Multi-stage | 219 MB (`day12-agent:cp2-test`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi chưa build lại bản 1-stage gốc nên không ghi số MB giả cho cột đó. Image
> multi-stage đã đo thực tế là 219 MB. Phần chênh lệch chủ yếu là
> compiler/build tool, cache của pip và artifact trung gian chỉ cần ở builder; runtime
> stage chỉ nhận dependency đã cài, source và Python slim. Cần chạy lại hai lệnh build
> trên máy có Docker để bổ sung số đo thực trước khi nộp bản cloud.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, sửa `app/main.py` vẫn tái sử dụng các layer base image,
> `COPY requirements.txt`, `pip install` và phần dependency copy từ builder; chỉ các
> layer copy source phía sau và metadata kế tiếp cần tạo lại. Nếu đặt `COPY . .` trước
> `RUN pip install`, bất kỳ thay đổi source nào cũng làm mất cache của layer copy và
> buộc cài lại toàn bộ dependency, dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng RCE trong Python có thể cho kẻ tấn công chạy lệnh trong container. Nếu
> process chạy root và runtime/container engine còn có cấu hình yếu hoặc lỗ hổng
> escape, quyền root trong container làm tăng khả năng sửa filesystem, đọc mount nhạy
> cảm rồi leo sang host với quyền cao. `USER agent` cắt chuỗi ngay sau bước RCE: mã độc
> chỉ có quyền của user không đặc quyền, nên phạm vi ghi/đọc và tác động bị thu hẹp
> (dù đây không phải biện pháp thay thế việc vá lỗi và harden container).

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request sát cuối phút cũ (ví dụ 10:00:59.x), bộ đếm reset
> ở 10:01:00, rồi gửi thêm 10 request sát đầu phút mới (10:01:00.x). Cả hai nhóm nằm
> trong khoảng chưa tới 2 giây nhưng thuộc hai bucket phút khác nhau. Sliding window
> 60 giây vẫn nhìn thấy nhóm đầu nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một khoảng ngắn; cost guard giới hạn tổng tiền
> theo user trong cả tháng. Một request hiếm nhưng prompt/output rất lớn có thể qua
> rate limit mà bị cost guard chặn vì ngân sách đã gần hết. Ngược lại, nhiều request
> nhỏ liên tiếp có tổng chi phí rất thấp vẫn có thể bị rate limit chặn dù còn nhiều
> ngân sách tháng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp và liveness cũng ping Redis: Redis mất kết nối → `/health` của cả ba instance
> trả lỗi → orchestrator đánh dấu cả ba unhealthy → lần lượt restart chúng → Redis vẫn
> chưa lên nên probe tiếp tục lỗi và cụm rơi vào vòng restart, làm mất cả khả năng quan
> sát process. Tách riêng thì `/health` vẫn 200 (không restart oan), còn `/ready` 503 để
> load balancer tạm rút ba instance khỏi traffic; Redis hồi phục thì readiness tự xanh.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Máy hiện tại không có Docker nên tôi chưa chạy được bài scale 3 instance. Với Redis
> dùng chung, mỗi response kế tiếp phải thấy `history_length` tăng 2 dù request vào
> instance nào. Nếu dùng dict Python, mỗi instance có lịch sử riêng: qua round-robin
> con số sẽ lặp/nhảy theo instance (thường 0, 0, 0 rồi 2, 2, 2) thay vì 0, 2, 4, 6...
> và lịch sử của một instance mất hẳn khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế lúc chuẩn bị deploy là Docker đã cài nhưng terminal chưa nhận lệnh
> `docker` vì thư mục CLI chưa có trong `PATH`; sau đó build còn timeout khi tải
> `watchfiles` từ PyPI. Tôi tìm ra CLI trong thư mục cài Docker Desktop, gọi bằng đường
> dẫn đầy đủ, rồi tách `requirements-prod.txt` để image chỉ cài dependency runtime và
> tăng timeout/retry của pip. Kết quả image build thành công, agent/Redis đều healthy.
> Cloud HTTPS vẫn cần đăng nhập Railway/Render, tạo Redis add-on và set secret trên
> dashboard trước khi thay URL local bằng URL công khai.
