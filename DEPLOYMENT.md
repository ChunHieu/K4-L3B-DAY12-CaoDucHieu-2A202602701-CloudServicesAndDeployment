# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Cao Đức Hiếu |
| Mã học viên | 2A202602701 |
| Repo | https://github.com/ChunHieu/K4-L3B-DAY12-CaoDucHieu-2A202602701-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-9wxp.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-09-29 |
| Web service | `day12-agent` |
| Redis | Render Key Value `day12-redis` |

## Biến Môi Trường Đã Set Trên Cloud

Tài liệu chỉ ghi tên biến và nguồn cấu hình, không lưu giá trị secret.

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự cấp |
| `AGENT_API_KEY` | ✅ | Secret đặt trong Render dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Connection string lấy từ Render Key Value `day12-redis` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Cấu hình qua `render.yaml` |
| `MONTHLY_BUDGET_USD` | ✅ | Cấu hình qua `render.yaml` |
| `LOG_LEVEL` | ✅ | Cấu hình qua `render.yaml` |

## Lệnh Kiểm Tra

```bash
URL=https://day12-agent-9wxp.onrender.com

curl -i "$URL/health"
curl -i "$URL/ready"

curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```text
GET /health
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP 200
{"status":"ready","redis":true}

POST /ask không có API key
HTTP 401
{"detail":"invalid or missing API key"}
```

`DEPLOY_API_KEY` chỉ được lưu trong `.env` cục bộ để bộ test CP5 kiểm tra
request có xác thực. Giá trị khóa không xuất hiện trong tài liệu hoặc repository.

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — Render dashboard hiển thị service `day12-agent` ở trạng thái Live.
- `screenshots/health.png` — kết quả thật của endpoint `/health` trên public URL.
