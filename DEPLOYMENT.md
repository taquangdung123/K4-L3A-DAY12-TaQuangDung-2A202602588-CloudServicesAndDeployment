# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tạ Quang Dũng |
| Mã học viên | 2A202602588 |
| Repo | https://github.com/taquangdung123/K4-L3A-DAY12-TaQuangDung-2A202602588-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | Chưa có — phương án Docker Compose local tại `http://localhost:8000` |
| Platform | Railway (cấu hình mục tiêu); hiện dùng Docker Desktop local fallback |
| Ngày kiểm tra local | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ local | 8000 trong Docker Compose; cloud chưa set |
| `AGENT_API_KEY` | ✅ local | Compose nội suy từ `.env` bị Git ignore; cloud chưa set |
| `REDIS_URL` | ✅ local | `redis://redis:6379/0` trong Compose; cloud chưa tạo Redis add-on |
| `RATE_LIMIT_PER_MINUTE` | ✅ local | 10; cloud chưa set |
| `MONTHLY_BUDGET_USD` | ✅ local | 10.0; cloud chưa set |
| `LOG_LEVEL` | ✅ local | INFO; cloud chưa set |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
GET /health  200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET /ready   200 {"status":"ready","redis":true}
POST /ask (không API key) 401
POST /ask (có API key, X-User-Id=sv-test) 200
docker compose exec -T redis redis-cli ping -> PONG
docker compose exec -T agent id -> uid=100(agent) gid=101(agent)
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — chỉ cần bổ sung sau khi deploy cloud thật
- `screenshots/health.png` — kết quả gọi `/health` từ agent chạy bằng Docker Compose

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```text
Docker Compose đã chạy thật hai service `agent` và `redis`; cả hai đều healthy.
Agent dùng `redis://redis:6379/0`, Redis trả `PONG`, `/ready` trả 200 và lịch sử
hội thoại được quan sát trong key `history:docker-real-redis`. Chưa có browser session
cloud đã đăng nhập, vì vậy đây là local fallback chứ không được trình bày như deploy cloud.
```
