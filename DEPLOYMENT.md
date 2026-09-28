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
| Public URL | https://k4-l3a-day12-taquangdung-2a202602588-cloudservic-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | Railway service variable, giá trị được che kín |
| `REDIS_URL` | ✅ | Reference variable `${{Redis.REDIS_URL}}` từ Railway Redis |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Railway service variable, giá trị 10 |
| `MONTHLY_BUDGET_USD` | ✅ | Railway service variable, giá trị 10.0 |
| `LOG_LEVEL` | ✅ | Railway service variable, giá trị INFO |

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
POST /ask (có API key, X-User-Id=cp5-final) 200, response có answer
Rate limit (X-User-Id=cp5-rate-final): 200 x10, sau đó 429 x5
Railway dashboard: agent Online, Redis Online
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — Railway dashboard với agent và Redis Online
- `screenshots/health.png` — kết quả gọi public endpoint `/health`
