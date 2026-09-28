# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Thị Lan |
| Mã học viên | 2A202602621 |
| Repo | https://github.com/nan-bi/K4-L3A-DAY12-TranThiLan-2A202602621-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-day12-agent-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway (Internal Connection URL) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

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
# 1. GET /health
HTTP/1.1 200 OK
content-length: 53
content-type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. GET /ready
HTTP/1.1 200 OK
content-length: 31
content-type: application/json

{"status":"ready","redis":true}

# 3. POST /ask (Không có API key)
HTTP/1.1 401 Unauthorized
content-length: 40
content-type: application/json

{"detail":"invalid or missing API key"}

# 4. POST /ask (Có API key)
HTTP/1.1 200 OK
content-length: 218
content-type: application/json

{"answer":"Deploy là quá trình đóng gói, cấu hình và đưa ứng dụng lên máy chủ cloud để người dùng có thể truy cập qua Internet.","user_id":"sv-test","history_length":0,"cost_usd":0.000171,"tokens":{"in":15,"out":42}}

# 5. Rate limit loop test
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đã lưu ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` và `/ready` kiểm tra service

---

## Phương Án Dự Phòng (Local Fallback)

Khi cần kiểm thử độc lập ở môi trường local hoặc chưa kích hoạt cloud:
- Đặt `LOCAL_FALLBACK=true` trong `.env`
- Chạy `docker compose up -d` và kiểm tra với `docker compose ps`
- Toàn bộ service agent và redis hoạt động ở `http://localhost:8000`
- Bộ test `pytest tests/test_cp5.py -v` tự động kiểm tra stack local và screenshots thành công.
