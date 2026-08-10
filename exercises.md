# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời mẫu*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hà Anh Tuấn  Mã học viên: 2A202601582

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công trên production khi chúng ta quên cấu hình biến môi trường `API_TOKEN`. Kẻ tấn công hoặc người dùng bên ngoài có thể phát hiện và gọi API miễn phí hoặc trái phép bằng token `"changeme"`, làm thất thoát dữ liệu hoặc làm phát sinh hóa đơn LLM khổng lồ. Với cơ chế "Fail Fast" (không có giá trị mặc định), ứng dụng sẽ crash ngay lập tức khi khởi tạo, giúp cảnh báo sớm cho nhà phát triển biết lỗi thiếu cấu hình cấu hình ngay lúc deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:47:04.123456+00:00", "client_id": "sv-test", "prompt_tokens": 1, "completion_tokens": 33, "usd_cost": 0.00001995}`

Hai việc có thể làm được với dòng log này:
1. **Lọc và phân tích tự động:** Dễ dàng lập biểu đồ theo dõi chi phí theo thời gian thực hoặc truy vấn tìm "Client nào đang tiêu tốn nhiều tiền nhất hôm nay" thông qua các công cụ gom log (như Datadog, ELK, GCP Logging).
2. **Cảnh báo tự động (Alerting):** Thiết lập cảnh báo (Slack, Email) nếu mức độ log là `ERROR` xuất hiện quá nhiều trong 5 phút, hoặc nếu `usd_cost` của một lượt chat vượt quá ngưỡng an toàn. Điều này không thể làm được với log dạng text tự do từ `print()`.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.8 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (khoảng 1.5 GB) bao gồm:
1. Các công cụ phát triển phần mềm hệ thống và trình biên dịch (như gcc, build-essential), header files chỉ dùng để compile dependencies ở stage builder, đã được lược bỏ ở stage runtime.
2. Các package và thư viện hệ thống thừa thãi của HĐH Debian đầy đủ trong base image `python:3.11` (bản slim chỉ giữ lại các thành phần tối thiểu chạy python).
3. Thư mục cache cài đặt của pip (pip cache) được sinh ra khi cài đặt dependencies ở builder stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

* Khi sửa một ký tự trong `app/main.py` và build lại:
  * Tất cả các layer cài đặt dependencies (như `COPY requirements.txt .` và `RUN pip install ...`) được dùng lại từ cache của Docker.
  * Chỉ các layer từ `COPY app ./app` trở đi mới bị huỷ cache và phải chạy lại. Quá trình build diễn ra cực nhanh (dưới 2 giây).
* Nếu đặt `COPY . .` lên trước `RUN pip install`:
  * Bất kỳ thay đổi nhỏ nào trong mã nguồn (ngay cả sửa một ký tự) đều làm huỷ cache của layer `COPY . .`.
  * Docker sẽ buộc phải chạy lại lệnh `RUN pip install` từ đầu để tải xuống và cài đặt lại toàn bộ thư viện, làm tăng thời gian build lên vài phút và lãng phí băng thông mạng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

* Chuỗi sự kiện tấn công:
  1. Kẻ tấn công khai thác thành công một lỗ hổng trong code Python (ví dụ: Remote Code Execution - RCE qua việc upload file hoặc injection).
  2. Kẻ tấn công thực thi mã độc trong container. Do container chạy dưới quyền root, tiến trình Python trong container có quyền root.
  3. Kẻ tấn công thực hiện "container escape" (vượt ngục container) bằng cách mount các tài nguyên hệ thống từ máy host hoặc exploit nhân hệ điều hành.
  4. Một khi thoát ra ngoài máy host, vì tiến trình trong container chạy bằng root của host (UID 0), kẻ tấn công sẽ giành được quyền root của máy host.
* Lệnh `USER appuser` cắt đứt chuỗi này ở bước số 2: Tiến trình Python chạy với quyền hạn chế của `appuser` (non-root, UID 10001). Khi bị hack, mã độc chỉ có quyền của user thường trong container, không thể ghi đè file hệ thống của container, cũng không thể thực hiện các syscall nguy hiểm để thoát ra máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

* Header `WWW-Authenticate: Bearer` là bắt buộc theo đặc tả của chuẩn HTTP (RFC 6750) đối với mã lỗi 401 để chỉ rõ cho client biết phương thức xác thực được hỗ trợ là Bearer token.
* Trả về **cùng một** thông điệp lỗi cho cả 3 trường hợp là để ngăn chặn việc rò rỉ thông tin cho kẻ tấn công (Information Disclosure). Nếu trả về thông báo cụ thể (ví dụ: "token sai"), kẻ tấn công sẽ biết rằng scheme họ đoán đã đúng và họ chỉ cần tiếp tục brute-force token. Việc dùng thông báo chung buộc kẻ tấn công phải dò dẫm mà không có bất kỳ phản hồi định hướng nào.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

* Client gửi được tối đa **10 request** liên tiếp trước khi bị lỗi 429 (vì xô chứa tối đa `capacity=10` token).
* Nếu bỏ đoạn `min(capacity, ...)` trong `available()`: Sau 10 phút im lặng, lượng token tích lũy lý thuyết trong xô sẽ là: $10 \text{ (token ban đầu)} + 10 \text{ (phút)} \times 10 \text{ (token nạp thêm/phút)} = 110 \text{ token}$. Khi đó, client có thể gửi liên tục **110 request** trước khi bị chặn 429. Điều này làm mất đi hoàn toàn ý nghĩa khống chế tải của rate limiter, khiến server có thể bị quá tải khi có client gửi dồn dập request sau một thời gian im lặng.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

* **Với hạn mức $30/tháng:**
  * Thiệt hại tối đa: **$30** (bị tiêu hết toàn bộ hạn mức tháng chỉ trong vài giờ từ 2h sáng).
  * Khả năng tự hồi phục: Ứng dụng của client đó sẽ bị khóa hoàn toàn và chỉ tự hồi phục vào **đầu tháng sau** (hoặc cần quản trị viên can thiệp thủ công mở khóa).
* **Với hạn mức $1/ngày:**
  * Thiệt hại tối đa: Chỉ **$1** (khi đạt ngưỡng $1, giao dịch của ngày hôm đó sẽ bị chặn ngay lập tức).
  * Khả năng tự hồi phục: Ứng dụng sẽ **tự động hoạt động bình thường trở lại vào ngày hôm sau** (khi key Redis chứa thông tin chi tiêu theo ngày tự động chuyển sang ngày mới).

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối trong 30 giây.
2. Cả 3 container chạy chat service khi gọi endpoint gộp (health check) sẽ đồng loạt trả về lỗi `503 Service Unavailable` vì không kết nối được Redis.
3. Orchestrator thấy cả 3 container đều báo unhealthy sẽ tự động gửi lệnh tắt và **restart toàn bộ cả 3 container**.
4. Khi cả 3 container đang trong quá trình restart và khởi động lại, dịch vụ chat bị sập hoàn toàn (downtime). Lúc này, kể cả khi Redis đã kết nối lại bình thường, hệ thống vẫn không thể nhận request vì không có container nào sẵn sàng.
*Kết luận:* Việc tách biệt `/healthz` (liveness - không check Redis) và `/readyz` (readiness - check Redis) giúp ngăn chặn vòng lặp restart vô hạn nguy hiểm này.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

* **Thông báo lỗi gặp phải:** `Error: Invalid value for '--port': '$PORT' is not a valid integer.` khi Railway chạy lệnh start command của container, dẫn đến việc healthcheck của Railway trả về `503 Service Unavailable` và báo lỗi `1/1 replicas never became healthy!`.
* **Cách tìm ra nguyên nhân:** Tôi kiểm tra trong mục **Deploy Logs** của bản deploy lỗi trên Railway dashboard và phát hiện uvicorn báo lỗi không parse được cổng `$PORT` thành số nguyên vì Railway chạy trực tiếp command dạng `exec` chứ không qua một shell.
* **Cách sửa:** Sửa dòng cấu hình `startCommand` trong file `railway.toml` từ `uvicorn app.main:app --host 0.0.0.0 --port $PORT` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port $PORT'`. Lệnh `sh -c` sẽ tạo ra một shell trung gian để biên dịch chính xác biến môi trường `$PORT` thành số cổng trước khi truyền cho uvicorn.
