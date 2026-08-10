# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Hà Anh Tuấn |
| Mã học viên | 2A202601582 |
| Repo | https://github.com/anhtuans-project/K4-Day12-2A202601582-HaAnhTuan |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-production-3c87.up.railway.app |
| Platform | Railway |
| Ngày deploy | 10-08-2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-chat-production-3c87.up.railway.app/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-chat-production-3c87.up.railway.app/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://day12-chat-production-3c87.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-chat-production-3c87.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-production-3c87.up.railway.app/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
(.venv) PS D:\Anh Tuan\VinAI\K4-Day12-Cloud-Services-And-Deployment> curl.exe -i https://day12-chat-production-3c87.up.railway.app/healthz                                                                                                  
HTTP/1.1 200 OK                                                                     
Content-Type: application/json
Date: Mon, 10 Aug 2026 08:45:11 GMT
Server: railway-hikari
x-railway-request-id: w5HtikgVR4GS96_uss7a6g
Content-Length: 64
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

(.venv) PS D:\Anh Tuan\VinAI\K4-Day12-Cloud-Services-And-Deployment> curl.exe -i https://day12-chat-production-3c87.up.railway.app/readyz                                                                                                   
HTTP/1.1 200 OK                                                                     
Content-Type: application/json
Date: Mon, 10 Aug 2026 08:45:31 GMT
Server: railway-hikari
x-railway-request-id: Hre6CHqXSrOh5p8TYqdHTg
Content-Length: 31
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}

(.venv) PS D:\Anh Tuan\VinAI\K4-Day12-Cloud-Services-And-Deployment> curl.exe -i -X POST https://day12-chat-production-3c87.up.railway.app/chat -H "Content-Type: application/json" -d '{\"message\":\"Hello\"}'
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 10 Aug 2026 08:49:08 GMT
Server: railway-hikari
www-authenticate: Bearer
x-railway-request-id: za5ZhFP6TBaZl6jwnTga3g
Content-Length: 44
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing bearer token"}

(.venv) PS D:\Anh Tuan\VinAI\K4-Day12-Cloud-Services-And-Deployment> curl.exe -i -X POST https://day12-chat-production-3c87.up.railway.app/chat -H "Content-Type: application/json" -H "Authorization: Bearer $env:API_TOKEN" -H "X-Client-Id: sv-test" -d '{\"message\":\"Deploy\"}'
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 08:51:48 GMT
Server: railway-hikari
x-railway-request-id: HEKgcqOSRTu-w2ce7fhULg
Content-Length: 337
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"reply":"Câu hỏi hay. Deploy thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 2 lượt trao đổi trước đó.)","client_id":"sv-test","turns_before":2,"usd_cost":3.18e-05,"usage":{"prompt":36,"completion":44}}

(.venv) PS D:\Anh Tuan\VinAI\K4-Day12-Cloud-Services-And-Deployment> 1..15 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " -X POST https://day12-chat-production-3c87.up.railway.app/chat -H "Content-Type: application/json" -H "Authorization: Bearer $env:API_TOKEN" -H "X-Client-Id: sv-test" -d '{\"message\":\"test\"}' }; ""
200 200 200 200 200 200 200 200 200 200 429 200 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

