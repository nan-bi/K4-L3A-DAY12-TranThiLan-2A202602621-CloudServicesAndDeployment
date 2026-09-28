# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời của bạn vào từng câu bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Thị Lan  Mã học viên: 2A202602621

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Production (như Railway hoặc Render), người quản trị viên vô tình quên khai báo biến môi trường `AGENT_API_KEY` trong dashboard quản lý bí mật. 

Nếu hệ thống để giá trị mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động thành công và container báo Healthy. Khi đó, API endpoint `/ask` mở công khai ra Internet với khóa `"changeme"`. Bất kỳ ai hoặc các bot tự động quét API trên mạng đều có thể dùng khóa mặc định phổ biến này để gửi hàng loạt request gọi LLM. Hậu quả là tài khoản LLM bị đốt sạch ngân sách và phát sinh chi phí lớn, mà bạn chỉ phát hiện ra khi nhận hóa đơn vào cuối tháng. 

Ngược lại, nhờ nguyên tắc Fail-Fast (không gán giá trị mặc định cho secret), ứng dụng sẽ ném lỗi `ValidationError` và crash ngay lập tức trong quá trình khởi động (startup). Khi quan sát quá trình deploy thấy service không thể khởi động và log báo rõ thiếu biến môi trường, bạn sẽ nhận ra lỗi ngay lúc đó và bổ sung cấu hình trước khi service tiếp nhận bất kỳ request nào từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:00:00.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000171}
```

Hai việc làm được với dòng log có cấu trúc JSON này:
1. **Lọc, tổng hợp và thống kê tự động bằng công cụ thu thập log (Datadog, ELK, CloudWatch, Loki):** Các công cụ phân tích có thể parse trường JSON tự động để chạy câu truy vấn tổng hợp như: tính tổng chi phí `cost_usd` theo từng `user_id` trong 24 giờ qua (`SUM(cost_usd) GROUP BY user_id`), hoặc tìm ra top 10 người dùng tiêu tốn nhiều token nhất. Với `print("đã trả lời xong")`, thông tin về chi phí, token và user hoàn toàn bị mất.
2. **Thiết lập cảnh báo (Alerting) thời gian thực dựa trên ngưỡng giá trị:** Dễ dàng tạo rule cảnh báo tự động (Alert) khi trường `cost_usd` vượt quá một ngưỡng nhất định trong một request, hoặc phát hiện lưu lượng token đầu vào (`tokens_in`) tăng đột biến nhằm ngăn chặn các hành vi tấn công spam prompt dài.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800MB) bao gồm:
1. **Base image:** Bản 1-stage dùng `python:3.11` đầy đủ, chứa toàn bộ hệ điều hành Debian hoàn chỉnh với rất nhiều công cụ biên dịch (gcc, g++, make), thư viện C/C++ header (`build-essential`), git, curl, tài liệu trợ giúp man pages và các tiện ích dòng lệnh không phục vụ cho việc chạy app. Trong khi bản multi-stage ở stage runtime dùng `python:3.11-slim`, chỉ giữ lại phần nhân tối thiểu cần thiết để thực thi Python.
2. **Quá trình build và cache của pip:** Trong multi-stage build, toàn bộ file wheels, cache tạm thời khi tải các gói và các file object biên dịch chỉ nằm ở stage `builder`. Stage runtime cuối cùng chỉ copy kết quả các thư viện đã được cài đặt hoàn chỉnh (`/install` sang `/usr/local`), loại bỏ toàn bộ compiler và file rác trung gian khỏi image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile tối ưu hiện tại:**
  - Các layer trước: `FROM python:3.11-slim AS builder`, `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...`, và lệnh `COPY --from=builder /install /usr/local` đều được dùng lại hoàn toàn từ cache (`CACHED`).
  - Chỉ có layer `COPY . .` (và các chỉ thị sau đó như `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`) bị mất cache và phải thực thi lại. Do đó thời gian build lại chỉ mất chưa đầy 1–2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi sửa dù chỉ một ký tự trong file code (như `app/main.py`), checksum của layer `COPY . .` thay đổi, Docker sẽ vô hiệu hóa (bust) toàn bộ cache từ layer đó trở đi.
  - Kết quả là lệnh `RUN pip install -r requirements.txt` bị buộc phải chạy lại từ đầu: tải lại toàn bộ thư viện qua mạng và cài đặt lại, khiến mỗi lần sửa code nhỏ phải mất vài phút chờ build thay vì vài giây, gây lãng phí băng thông và tài nguyên CI/CD.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

**Chuỗi sự kiện leo quyền (Container Escape):**
1. Ứng dụng Python có một lỗ hổng thực thi mã từ xa (RCE) — ví dụ qua `eval()`, tải module không an toàn, command injection qua subprocess hoặc một thư viện bên thứ ba có lỗ hổng bảo mật.
2. Kẻ tấn công gửi payload khai thác thành công để mở một interactive shell bên trong container. Do container mặc định chạy bằng user `root` (UID 0), kẻ tấn công ngay lập tức có đặc quyền tối cao trong filesystem và namespace của container.
3. Từ quyền `root` trong container, kẻ tấn công khai thác các lỗ hổng kernel của máy host (như Dirty COW, container breakout bugs), lợi dụng các capabilities mặc định, hoặc truy cập vào các thư mục / volume được mount từ host (ví dụ docker socket `/var/run/docker.sock` hoặc thư mục nhạy cảm). Kẻ tấn công ghi đè các tiến trình trên host hoặc tạo user mới trên host với UID 0 (root của máy chủ thật).

**Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
Lệnh `USER appuser` chuyển tiến trình sang user không đặc quyền (non-root, UID 10001, không nằm trong file sudoers). Khi kẻ tấn công khai thác được code Python và chiếm được shell:
- Kẻ tấn công chỉ có quyền hạn của user thường: không thể cài đặt thêm công cụ mạng hay kernel modules, không thể can thiệp vào các tiến trình hệ thống, bị từ chối truy cập (Permission Denied) vào hầu hết các file hệ thống và socket nhạy cảm.
- Điều này ngăn chặn hoàn toàn bước leo thang đặc quyền để thoát ra kiểm soát máy chủ host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Con số tối đa:** Một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
- **Giải thích cách đạt được:**
  - Với cách đếm cố định theo phút đồng hồ (Fixed Window Counter, reset bộ đếm về 0 vào giây `:00` của mỗi phút):
    - Ở phút thứ nhất, vào giây cuối cùng (giây thứ 59, lúc `10:00:59`), người dùng gửi liên tiếp 10 request. Hệ thống kiểm tra thấy chưa vượt quá 10 request của phút này nên cho qua toàn bộ.
    - Đúng 1 giây sau, khi đồng hồ chuyển sang phút tiếp theo (giây thứ 00, lúc `10:01:00`), bộ đếm tự động reset về 0. Người dùng lập tức gửi thêm 10 request nữa. Hệ thống kiểm tra thấy phút mới chưa có request nào nên tiếp tục cho qua toàn bộ 10 request này.
    - Như vậy, trong khoảng thời gian chỉ 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh tới 20 request — gấp đôi hạn mức cho phép.
  - Thuật toán Sliding Window (cửa sổ trượt 60s liên tục) giải quyết triệt để vấn đề này vì nó luôn tính tổng số request trong đúng 60 giây trôi qua tính từ thời điểm hiện tại (`now - 60`), đảm bảo không bao giờ có quá 10 request trong bất kỳ khoảng 60 giây nào.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau:**
  - **Rate Limit:** Bảo vệ về mặt **tần suất và lưu lượng (Velocity)** trong thời gian ngắn (ví dụ: tối đa 10 request/phút) nhằm ngăn chặn hành vi spam, DoS, và bảo vệ máy chủ không bị nghẽn tài nguyên CPU/RAM.
  - **Cost Guard:** Bảo vệ về mặt **tài chính và ngân sách (Financial Budget)** trong chu kỳ dài (ví dụ: tối đa 10.0 USD/tháng) nhằm giới hạn tổng chi phí tiêu thụ token mô hình LLM của từng người dùng.
- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  - Một người dùng mới gửi 1 request trong ngày (tần suất 1 request/phút, hoàn toàn nằm trong hạn mức 10 req/phút của Rate Limit). Tuy nhiên, người dùng này trong tháng trước đó đã tiêu hết 10.0 USD / 10.0 USD ngân sách tháng. Cost Guard lập tức chặn lại và trả về mã lỗi `402 Payment Required`.
- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  - Một người dùng mới toanh chưa tiêu đồng nào (ngân sách còn nguyên 10.0 USD). Nhưng người dùng này dùng script tự động gửi liên tục 15 request trong vòng 3 giây với các câu hỏi ngắn chỉ tốn $0.0001 mỗi câu (tổng chi phí phát sinh chỉ $0.0015, rất nhỏ so với ngân sách $10.0). Cost Guard vẫn cho qua nhưng Rate Limit sẽ phát hiện vượt quá 10 request/phút và chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện (Cascading Failure):
1. **Sự cố bên ngoài:** Redis bị khởi động lại hoặc mất kết nối mạng tạm thời trong 30 giây.
2. **Liveness probe thất bại đồng loạt:** Vì endpoint kiểm tra liveness gọi vào Redis, cả 3 container đang chạy đều phản hồi lỗi 503 Unhealthy khi orchestrator gửi request kiểm tra sức khỏe.
3. **Orchestrator kích hoạt restart toàn bộ:** Orchestrator (Docker daemon, Kubernetes, Railway) mặc định coi liveness fail là tiến trình đã bị chết/treo, nên ra lệnh dừng và restart lại toàn bộ cả 3 container cùng lúc.
4. **Hệ thống sập hoàn toàn (Total Outage):** Cả 3 container đều đang trong trạng thái tắt hoặc khởi động lại. Tất cả người dùng gửi request vào hệ thống trong thời gian này đều nhận về lỗi `502 Bad Gateway` do không còn bất kỳ instance nào hoạt động.
5. **Vòng lặp khởi động thất bại:** Khi Redis vừa kết nối lại sau 30 giây, các container vẫn đang loay hoay trong chu kỳ khởi động, tải thư viện hoặc bị nghẽn do quá trình restart hàng loạt. Sự cố mất kết nối 30 giây của một service phụ thuộc bên ngoài đã biến thành sự cố sụp đổ toàn bộ hệ thống.
*(Nếu tách riêng: `/health` độc lập trả về 200 process vẫn sống không bị restart; `/ready` trả về 503 để Load Balancer tạm ngừng phân phối request vào. Khi Redis sống lại sau 30 giây, `/ready` lại trả về 200 và Load Balancer lập tức tiếp tục định tuyến mà không có container nào bị khởi động lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong một dictionary Python trong bộ nhớ RAM của process, khi có 3 container chạy song song sau Load Balancer:
- Lượt 1: Request được định tuyến vào container A -> A lưu vào RAM của nó -> `history_length = 0`.
- Lượt 2: Load Balancer phân phối request tiếp theo sang container B. Vì RAM của container B hoàn toàn rỗng, B không biết gì về lượt hỏi trước đó -> `history_length` lại là `0` (thay vì phải là `2`)!
- Lượt 3: Request sang container C -> `history_length` vẫn tiếp tục là `0`!
- Lượt 4: Request quay lại container A (nếu dùng round-robin) -> lúc này A đọc từ RAM của nó và thấy 2 tin nhắn từ lượt 1 -> `history_length = 2`.
- Lượt 5: Request vào container B -> B thấy 2 tin nhắn của lượt 2 trong RAM của nó -> `history_length = 2`.

Kết quả: Giá trị `history_length` sẽ nhảy lộn xộn (0, 0, 0, 2, 2...), người dùng sẽ thấy AI agent liên tục bị "mất trí nhớ", không thể duy trì ngữ cảnh cuộc trò chuyện liền mạch. Ngược lại, khi chuyển state sang Redis tập trung, cả 3 container đều đọc và ghi chung vào một nguồn dữ liệu duy nhất, đảm bảo `history_length` luôn tăng đều đặn (0, 2, 4, 6...) bất kể request được phục vụ bởi container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Health check timeout khi deploy lên Cloud do ứng dụng cố định cổng lắng nghe và bind sai host.
- **Thông báo lỗi trong deploy logs:**
  ```text
  Deploy failed: Health check timed out after 30s on path /health. Service returned connection refused.
  ```
- **Cách tìm ra nguyên nhân:**
  1. Mở Deployment logs trên Dashboard: Quan sát thấy nền tảng tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `PORT` (ví dụ `PORT=7821`), trong khi lệnh CMD trong Dockerfile ban đầu lại cố định `--port 8000`. Do đó request kiểm tra health check của nền tảng gửi vào cổng `7821` bị từ chối kết nối (`Connection refused`).
  2. Đồng thời kiểm tra thông số host: Nếu uvicorn bind vào `127.0.0.1` thì chỉ có tiến trình bên trong container mới gọi được, Load Balancer bên ngoài container sẽ không thể kết nối tới service.
- **Cách sửa:**
  1. Trong `Dockerfile`, cập nhật lệnh `CMD` sang dạng shell wrapper để ưu tiên đọc giá trị biến môi trường `PORT` từ cloud, nếu không có mới fallback về 8000, đồng thời luôn bind vào `0.0.0.0`:
     `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  2. Push commit mới lên repository. Sau khi build lại, nền tảng nhận diện ứng dụng lắng nghe đúng cổng và health check phản hồi HTTP 200 OK ngay trong vòng 3 giây.
