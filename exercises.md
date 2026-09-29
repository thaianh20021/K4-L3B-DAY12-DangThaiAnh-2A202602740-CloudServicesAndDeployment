# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Thái Anh  Mã học viên: 2A202602740

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Một tình huống thực tế rất phổ biến là khi deploy ứng dụng lên môi trường Production hoặc Staging trên Cloud (như Render hoặc Railway), kỹ sư vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong mục Environment Variables của Dashboard.

Nếu để giá trị mặc định là `"changeme"`:
- Ứng dụng vẫn khởi động bình thường và báo trạng thái xanh (healthy).
- Hệ thống mở toang endpoint `/ask` ra Internet với key mặc định `"changeme"`. Các con bot tự động quét lỗ hổng hoặc kẻ xấu có thể dễ dàng dùng các dictionary key phổ biến (`changeme`, `admin`, `secret`) để gọi API miễn phí.
- Hậu quả: Ngân sách tài khoản LLM của bạn bị đốt sạch trong âm thầm, dữ liệu nhạy cảm có nguy cơ bị rò rỉ, và bạn chỉ phát hiện ra khi nhận được hóa đơn trừ tiền thẻ tín dụng vào cuối tháng.

Ngược lại, khi không có giá trị mặc định:
- Thư viện `pydantic-settings` sẽ lập tức ném ra ngoại lệ `ValidationError: Field required: AGENT_API_KEY` ngay khi container vừa bật lên.
- Quá trình deploy dừng lại ngay lập tức (Deploy Failed) ngay trước mắt bạn khi đang theo dõi log deployment.
- Việc "chết sớm" (fail-fast) này ép buộc developer phải bổ sung cấu hình secret hợp lệ trước khi hệ thống có thể nhận bất kỳ request thực tế nào từ bên ngoài, ngăn chặn hoàn toàn rủi ro bảo mật và thiệt hại tài chính.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ service:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T10:16:35.789123+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}
```

Hai việc làm được với dòng log có cấu trúc này mà `print("đã trả lời xong")` hoàn toàn bất khả thi:
1. **Truy vấn, lọc và tổng hợp số liệu theo trường (Structured Query & Metrics Aggregation):** Các hệ thống thu thập log tập trung hiện đại (như Datadog, Grafana Loki, AWS CloudWatch, ELK Stack) có thể tự động parse các trường JSON để chạy các câu truy vấn phức tạp như: *"Tính tổng số tiền `cost_usd` đã tiêu của user `sv-test` trong ngày"*, *"Vẽ biểu đồ phân phối p95 của `tokens_out` theo từng giờ"*, hoặc *"Lọc ra tất cả các event hoàn thành có chi phí lớn hơn 0.01 USD"*. Log bằng `print` là chuỗi text phi cấu trúc, không thể tổng hợp định lượng chính xác khi có hàng triệu dòng log.
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting & Monitoring):** Có thể cài đặt cảnh báo tức thời gửi về Slack/Telegram hoặc PagerDuty khi phát hiện một request bất thường, ví dụ: kích hoạt cảnh báo khi `cost_usd > 0.05` trong một lượt hỏi đơn lẻ, hoặc khi xuất hiện log với `"level": "error"`. Chuỗi `print("đã trả lời xong")` không cung cấp ngữ cảnh, không có timestamp UTC chuẩn máy đọc và không thể gắn điều kiện cảnh báo tự động.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch khổng lồ (~835 MB) đến từ:
1. **Trình biên dịch và công cụ phát triển (Build Tools & Compilers):** Ở bản 1-stage dùng base image `python:3.11` đầy đủ, hệ thống mang theo toàn bộ `gcc`, `g++`, `make`, `binutils`, các file C header (`linux-headers`, `libc-dev`, `python3-dev`) cần thiết để build các thư viện C extensions.
2. **Cache và file tạm của trình quản lý gói:** Bộ nhớ đệm tạm thời của `pip cache`, các file `.whl` tải về, và cache của `apt-get` sau khi cài đặt package.
3. **Các tiện ích hệ điều hành không cần thiết trong runtime:** Bản đầy đủ chứa nhiều công cụ dòng lệnh (man pages, tài liệu hướng dẫn, debug symbols, testing suites) hoàn toàn vô dụng khi chạy ứng dụng trên production.

Bản Multi-stage tối ưu bằng cách:
- Ở stage `builder`: Sử dụng để biên dịch và cài đặt thư viện vào thư mục đích `/install` với cờ `--no-cache-dir`.
- Ở stage `runtime`: Sử dụng base image siêu gọn nhẹ `python:3.11-slim` (chỉ chứa các shared libraries tối thiểu để chạy Python interpreter) và chỉ `COPY --from=builder /install /usr/local`. Toàn bộ compiler và rác phát sinh ở stage builder bị vứt bỏ, giúp image giảm hơn 80% kích thước, deploy nhanh gấp 5 lần và giảm đáng kể diện tích tấn công (attack surface).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Khi sửa một ký tự trong `app/main.py` và build lại với Dockerfile hiện tại:**
  - Các layer từ đầu cho đến hết bước cài đặt thư viện: `FROM python:3.11-slim AS builder`, `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...`, và ở stage runtime: `FROM python:3.11-slim AS runtime`, `WORKDIR /app`, `RUN useradd ...`, `COPY --from=builder /install /usr/local` đều được **dùng lại 100% từ cache** (Docker hiển thị `CACHED`).
  - Chỉ có layer `COPY app ./app` và các chỉ thị bên dưới nó (`COPY utils ./utils`, `RUN chown ...`) mới bị vô hiệu hóa cache (cache invalidated) và phải thực thi lại. Thời gian build lại chỉ mất khoảng 1 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Docker áp dụng cơ chế layer caching: khi một layer bị thay đổi, toàn bộ các layer tiếp theo phía sau nó đều bị huỷ cache và phải chạy lại từ đầu.
  - Do `COPY . .` chứa `app/main.py`, việc thay đổi dù chỉ một dấu chấm hay dấu cách trong code cũng làm layer `COPY . .` đổi mã hash. Khi đó, lệnh `RUN pip install -r requirements.txt` nằm sau sẽ bị ép buộc chạy lại, Docker phải kết nối mạng tải lại và cài đặt lại toàn bộ gói thư viện từ đầu. Quá trình build mỗi lần commit sẽ mất từ 2 đến 5 phút thay vì chỉ 1 giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện leo thang đặc quyền từ ứng dụng ra máy host:**
  1. Ứng dụng Python chứa một lỗ hổng bảo mật (ví dụ: Arbitrary Code Execution / RCE qua hàm `eval()`, lỗ hổng command injection khi dùng `os.system()`/`subprocess`, hoặc một thư viện bên thứ ba chưa vá lỗi).
  2. Kẻ tấn công gửi payload khai thác thành công và chiếm được quyền thực thi shell bên trong container.
  3. Mặc định container không có lệnh `USER`, tiến trình Python chạy với người dùng `root` bên trong container. Đáng chú ý, trong kiến trúc Linux container, `root` trong container có cùng User ID (`UID 0`) với `root` của Linux kernel trên máy host.
  4. Nếu máy host có cấu hình bất cẩn (như mount Docker socket `/var/run/docker.sock` vào container, chia sẻ thư mục nhạy cảm `/etc`, `/root` qua volume mount), hoặc nhân Linux tồn tại lỗ hổng container breakout (như CVE runc, dirty cow, cgroup release_agent), kẻ tấn công với UID 0 có thể dễ dàng vượt rào (container escape) và trực tiếp điều khiển máy host với toàn quyền root cao nhất của hệ thống máy chủ.
- **Lệnh `USER` cắt đứt chuỗi tấn công ở đâu:**
  - Lệnh `RUN useradd --create-home --uid 10001 appuser` và `USER appuser` cắt đứt chuỗi tấn công ngay tại **Bước 3**.
  - Tiến trình bên trong container bị hạ quyền xuống một tài khoản thông thường (UID 10001). Khi kẻ tấn công thực thi được shell, họ chỉ có quyền hạn tối thiểu của `appuser`: không thể cài thêm phần mềm, không thể sửa đổi file hệ thống trong container, không thể ghi vào thư mục ngoài `/app`, và quan trọng nhất là không thể tương tác với Docker socket hay kích hoạt các kỹ thuật leo thang đặc quyền container breakout yêu cầu UID 0.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số lượng request tối đa trong 2 giây liên tiếp:** Người dùng có thể gửi tối đa **20 requests**.
- **Giải thích cơ chế vượt hạn mức (Bursting exploit):**
  - Thuật toán đếm theo phút đồng hồ cố định (Fixed Window Counter) chia thời gian thành các khối riêng biệt (ví dụ: từ `10:00:00` đến `10:00:59` và từ `10:01:00` đến `10:01:59`), bộ đếm sẽ tự động reset về 0 ngay thời điểm giây thứ `00`.
  - Người dùng có thể khai thác kẽ hở tại ranh giới chuyển giao giữa hai phút:
    - Ở giây `10:00:59`: Người dùng xả dồn dập 10 requests. Cả 10 requests đều được chấp nhận vì vừa đúng hạn mức 10 req/phút của phút 10:00.
    - Ngay 1 giây sau, lúc `10:01:00`: Hệ thống reset counter về 0.
    - Ở giây `10:01:01`: Người dùng tiếp tục gửi ngay 10 requests tiếp theo. Cả 10 requests này vẫn được thông qua vì thuộc hạn mức của phút mới 10:01.
  - Kết quả: Trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:01`), server phải hứng chịu 20 requests liên tiếp (gấp đôi hạn mức thiết kế), có nguy cơ làm quá tải dịch vụ.
  - Thuật toán Cửa sổ trượt (Sliding Window Log) bằng Redis Sorted Set giải quyết triệt để vấn đề này vì nó luôn tính số lượng request trong đúng khoảng `[now - 60s, now]`, không có điểm mù ranh giới.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác biệt cốt lõi:**
  - **Rate Limiter (Tần suất):** Kiểm soát lưu lượng theo **thời gian ngắn** (số requests/phút). Mục tiêu là bảo vệ hạ tầng máy chủ khỏi bị quá tải, chống tấn công từ chối dịch vụ (DDoS) và chống spam request.
  - **Cost Guard (Ngân sách):** Kiểm soát chi phí tài chính theo **chu kỳ dài** (số USD/tháng). Mục tiêu là giới hạn chi tiêu token LLM thực tế phát sinh của mỗi user để bảo vệ túi tiền của chủ hệ thống.
- **Tình huống Rate Limiter cho qua nhưng Cost Guard chặn:**
  - Người dùng chỉ gửi duy nhất 1 câu hỏi trong ngày (hoàn toàn thỏa mãn giới hạn 10 requests/phút của Rate Limiter). Tuy nhiên, người dùng này đã tiêu hết hạn mức 10.0 USD của tháng hiện tại từ các ngày trước. Khi nhận request, Rate Limiter cho qua nhưng Cost Guard phát hiện `spent >= 10.0` nên lập tức chặn lại và trả về lỗi `402 Payment Required`.
- **Tình huống Cost Guard cho qua nhưng Rate Limiter chặn:**
  - Đầu tháng, người dùng mới chỉ tiêu 0.0 USD trên tổng ngân sách 10.0 USD được cấp. Người dùng viết vòng lặp script gửi liên tục 15 requests trong vòng 3 giây với các câu hỏi ngắn 2 từ (mỗi câu chỉ tốn 0.00001 USD). Ngân sách tháng còn rất nhiều (Cost Guard chấp nhận), nhưng do gửi quá 10 request trong cửa sổ 60 giây, Rate Limiter sẽ lập tức chặn từ request thứ 11 và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp `/health` và `/ready` thành một endpoint duy nhất và phụ thuộc vào Redis, thảm họa xảy ra theo trình tự sau:
1. **Giây 0:** Redis gặp sự cố mạng hoặc khởi động lại, tạm thời không nhận kết nối trong 30 giây.
2. **Giây 5 – 10:** Bộ điều phối cụm (Container Orchestrator như Docker Swarm, Kubernetes hoặc Cloud Platform) gửi request kiểm tra định kỳ (Liveness Probe) vào endpoint `/health`.
3. **Giây 10:** Do `/health` kiểm tra kết nối Redis và ping thất bại, cả 3 container agent đều đồng loạt trả về mã lỗi HTTP `503`.
4. **Giây 15:** Liveness probe thất bại khiến Orchestrator đưa ra kết luận sai lầm rằng *"cả 3 tiến trình container đều đã bị lỗi/deadlock không thể tự phục hồi"*. Orchestrator ra lệnh **ép buộc dừng và restart toàn bộ cả 3 container**.
5. **Giây 20:** Cả 3 container mới được khởi động lại cùng lúc, tiếp tục tự probe Redis khi vừa bật lên và vẫn thấy Redis chưa sống lại, tiếp tục trả về 503 và tiếp tục bị Orchestrator restart lần 2 (vòng lặp tử thần CrashLoopBackOff).
6. **Hậu quả:** Toàn bộ hệ thống sập hoàn toàn (total blackout) trong suốt 30 giây và mất thêm nhiều phút sau đó để ổn định.
*(Nếu tách đúng: `/health` chỉ kiểm tra process bản thân -> container không bị restart; `/ready` trả về 503 -> Load Balancer chỉ tạm thời ngưng định tuyến traffic vào cho đến khi Redis kết nối lại bình thường).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi chạy 3 instance (`agent=3`) sau một bộ cân bằng tải (Load Balancer - LB):
- Các request liên tiếp của cùng một `X-User-Id` sẽ được Load Balancer luân chuyển tuần tự (Round-Robin) tới các container khác nhau: Request 1 vào `agent_1`, Request 2 vào `agent_2`, Request 3 vào `agent_3`, Request 4 quay lại `agent_1`...
- **Nếu lưu trong một `dict` Python trong RAM:**
  - Mỗi container là một process độc lập trên hệ điều hành, sở hữu một không gian bộ nhớ RAM hoàn toàn cô lập, không nhìn thấy dữ liệu của nhau.
  - Khi người dùng gửi 5 câu hỏi liên tiếp, giá trị `history_length` trả về sẽ biến thiên thất thường và lộn xộn:
    - Lượt 1 (vào `agent_1`): RAM trống → `history_length = 0` (sau đó `agent_1` lưu câu 1 vào RAM của nó).
    - Lượt 2 (vào `agent_2`): RAM của `agent_2` hoàn toàn chưa có gì → `history_length = 0` (thay vì mong đợi là 2).
    - Lượt 3 (vào `agent_3`): RAM của `agent_3` cũng chưa có gì → `history_length = 0`.
    - Lượt 4 (vào lại `agent_1`): `agent_1` chỉ nhớ câu 1 nó từng xử lý → `history_length = 2` (bỏ mất câu 2 và 3).
    - Lượt 5 (vào lại `agent_2`): `agent_2` chỉ nhớ câu 2 → `history_length = 2`.
  - Kết quả: Agent bị hiện tượng "mất trí nhớ ngẫu nhiên", câu trả lời của LLM bị mất ngữ cảnh hội thoại liên tục. Ngược lại, khi dùng Redis tập trung, mọi container đều đọc/ghi chung một Redis nên `history_length` luôn tăng đều đặn: `0 -> 2 -> 4 -> 6 -> 8...`.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** `Application failed to respond / Health check timeout`. Container build thành công và được deploy lên Railway/Render, nhưng platform báo service không phản hồi (unhealthy) và tự động rollback hoặc restart liên tục.
- **Cách tìm ra nguyên nhân:**
  - Mở tab **Deploy Logs** trên dashboard của platform để đọc log khởi động của container.
  - Phát hiện Uvicorn báo đang lắng nghe tại cổng mặc định `Uvicorn running on http://0.0.0.0:8000`.
  - Trong khi đó, các dịch vụ Cloud PaaS (như Render, Railway, Google Cloud Run) hoạt động theo cơ chế cấp phát cổng động: platform tự động tiêm một biến môi trường `PORT` ngẫu nhiên (ví dụ `PORT=10000` hoặc `PORT=54321`) và router bên ngoài chỉ kiểm tra healthcheck vào cổng đó. Do Dockerfile cũ hardcode cố định `--port 8000`, cổng của app không khớp với cổng router chờ đợi, dẫn đến timeout.
- **Cách sửa chữa:**
  1. Trong [`Dockerfile`](file:///e:/AIthucchien/K4-L3B-DAY12-DangThaiAnh-2A202602740-CloudServicesAndDeployment/Dockerfile), cập nhật lệnh `CMD` để chạy thông qua shell nhằm nội suy biến môi trường `$PORT`:
     ```dockerfile
     CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
     ```
  2. Trong file cấu hình [`app/config.py`](file:///e:/AIthucchien/K4-L3B-DAY12-DangThaiAnh-2A202602740-CloudServicesAndDeployment/app/config.py), đảm bảo trường `port: int = 8000` tự động ánh xạ với biến môi trường `PORT` của hệ thống.
  3. Sau khi sửa và push lên GitHub, platform deploy lại bản mới, Uvicorn nhận đúng biến `$PORT` được cấp, và health check lập tức chuyển sang màu xanh (200 OK).
