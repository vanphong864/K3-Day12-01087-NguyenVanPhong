# Thông Tin Deploy — Checkpoint 5

> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Văn Phong |
| Mã học viên | 2A202601087 |
| Repo | https://github.com/vanphong864/K3-Day12-01087-NguyenVanPhong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-fb3c.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | fake:// (dùng fakeredis trong RAM) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://agent-production-fb3c.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://agent-production-fb3c.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://agent-production-fb3c.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://agent-production-fb3c.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://agent-production-fb3c.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:11:15 GMT
Server: railway-hikari
x-railway-request-id: _YzmG5DQTNaVKIYB9o6EoQ
Content-Length: 57
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:11:16 GMT
Server: railway-hikari
x-railway-request-id: XBMiH7tbTv6dc2uELPU1MQ
Content-Length: 31
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:11:16 GMT
Server: railway-hikari                                                                                                 x-railway-request-id: Alvw5K73Q6-wtpWvWUN5dQ                                                                           Content-Length: 39                                                                                                     x-hikari-trace: sin1.98a6                                                                                              x-railway-edge: sin1                                                                                                   Connection: keep-alive                                                                                                                                                                                                                        {"detail":"invalid or missing API key"}                                                                                                                                                                                                       answer         : Ngáº¯n gá»n: Deploy la gi phá»¥ thuá»c vÃo ba yáº¿u tá» â cáº¥u hÃ¬nh qua biáº¿n mÃ´i                                  trÆ°á»ng, health check Äá» orchestrator biáº¿t tráº¡ng thÃ¡i, vÃ giá»i háº¡n tÃi nguyÃªn. (MÃ¬nh                       Äang nhá» 20 lÆ°á»£t trao Äá»i trÆ°á»c ÄÃ³.)                                                          
user_id        : sv-test
history_length : 20
cost_usd       : 9.675E-05
tokens         : @{in=457; out=47}

200
200
200
200
200
200
200
200
200
429
429
429
429
429
429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
