# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Anh Tuấn  Mã học viên: 2A202602700

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tôi deploy lên Railway nhưng quên set `AGENT_API_KEY` trên dashboard. Tôi phát hiện khi chạy khởi tạo Settings() khi không có AGENT_API_KEY và thấy đúng lỗi ValidationError ... agent_api_key Field required. Nếu có mặc định "changeme", app vẫn khởi động, /health trả 200, và service chạy với khóa "changeme" mà ai đọc repo cũng biết, nên người lạ có thể gọi /ask tốn tiền của tôi. 

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được khi gọi `/ask` (chạy local, `REDIS_URL=fake://`):
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T06:01:00.645869+00:00", "user_id": "sv01", "tokens_in": 43, "tokens_out": 46, "cost_usd": 3.405e-05}
> ```

> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc theo trường**: tìm đúng các request của một user hoặc chỉ các dòng `level = "error"` trên log search của Railway hay bằng `jq`, thay vì grep chuỗi tự do rồi đoán.
> 2. **Tính toán và cảnh báo**: cộng `cost_usd` / `tokens_in + tokens_out` theo user hoặc theo giờ để biết ai đang tiêu nhiều tiền, và đặt cảnh báo khi tổng vượt ngưỡng.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB; nén 447 MB) |
| Multi-stage | 310 MB (nén 72.1 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.42 GB. Xem `docker history` thì
> layer cài thư viện gần như bằng nhau ở hai bản (95.1 MB so với 96 MB của
> venv), code chỉ 291 kB. Vậy gần như toàn bộ phần chênh lệch nằm ở **base
> image**: `python:3.11` bản đầy đủ mang theo cả bộ công cụ build của Debian —
> `gcc` (chạy `which gcc` trong bản 1 stage thấy có `/usr/bin/gcc`), header,
> các thư viện `-dev`, git... — những thứ chỉ cần lúc biên dịch, không cần lúc
> chạy. Bản multi-stage dùng `python:3.11-slim`, và stage runtime chỉ copy
> `/opt/venv` từ builder sang nên mọi thứ phát sinh lúc build bị bỏ lại. Ngoài
> ra bản đầu chạy `pip install` không có `--no-cache-dir` nên còn giữ thêm
> cache pip.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi đổi `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` trong `app/main.py` rồi
> build lại với `--progress=plain`:
> - **Dùng lại cache**: toàn bộ stage builder (`WORKDIR`, `python -m venv`,
>   `COPY requirements.txt`, `RUN pip install`), và ở stage runtime là
>   `useradd` và `COPY --from=builder /opt/venv` — tất cả báo `CACHED`.
> - **Chạy lại**: chỉ `COPY . .` (0.1s) và `RUN chown -R appuser /app` (0.5s),
>   vì hai layer này nằm sau chỗ code thay đổi. Cả lần build chưa tới 2 giây.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` (đúng như Dockerfile 1 stage
> ban đầu), sửa một ký tự bất kỳ cũng làm layer `COPY . .` đổi hash, kéo theo
> mọi layer sau nó mất cache, nên `pip install` phải chạy lại từ đầu. Tôi đo
> trên bản 1 stage: `pip install` chạy lại mất 25.1s (và tải lại thư viện qua
> mạng) chỉ vì đổi một ký tự.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

#### Chuỗi tấn công khi không dùng `USER`

```
1. Lỗ hổng RCE/Command Injection trong Python code
   ↓
2. Attacker chạy lệnh với quyền root (UID 0) trong container
   ├─ Đọc tất cả environment variables (bao gồm secrets)
   ├─ Sửa đổi code, install packages với apt
   └─ Chuẩn bị công cụ để thoát khỏi container
   ↓
3. Escape khỏi container (dùng lỗ hổng kernel/docker.sock)
   ↓
4. Trở thành root trên host machine → toàn quyền điều khiển
```

#### Lệnh `USER` cắt đứt ở bước 2

**Không có `USER`:**
- Lỗ hổng được khai thác → code chạy với `uid=0` (root)
- Attacker có đủ quyền để:
  - Ghi vào `/var/run/docker.sock` → điều khiển docker daemon
  - Khai thác kernel vulnerability (CVE-2019-5736) → thoát container
  - Mount volume từ host → chỉnh sửa file host trực tiếp

**Có `USER appuser`:**
- Lỗ hổng được khai thác → code chạy với `uid=1000` (user thường)
- Attacker **bị giới hạn**:
  - ❌ Không thể `apt install` (cần sudo)
  - ❌ Không thể ghi vào `/var/run/docker.sock`
  - ❌ Không thể khai thác các exploit cần root
  - ✅ Ngay cả khi thoát container → chỉ là user thường trên host (không phải root)

**Kết luận:** `USER` giảm tối đa quyền của attacker ở mỗi bước, khiến chuỗi tấn công phức tạp hơn và damage nhỏ hơn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**.
>
> Cách đạt: đếm theo phút đồng hồ thì bộ đếm reset về 0 đúng giây 00. Người
> dùng gửi 10 request lúc 10:00:59 (phút 10:00 còn đủ 10 lượt nên đều qua), rồi
> gửi tiếp 10 request lúc 10:01:00 — phút mới, bộ đếm vừa reset, lại có thêm
> 10 lượt. Kết quả là 20 request trong khoảng 10:00:59–10:01:00, trong khi xét
> từng phút đồng hồ thì vẫn "đúng luật" 10/phút.
>
> Với sliding window 60 giây, lúc 10:01:00 hàm `hit_count` vẫn đếm 10
> request của giây 59 (chưa cũ quá 60 giây nên chưa bị `zremrangebyscore`
> xóa), nên request thứ 11 bị 429 — phải đợi đến khoảng 10:01:59 mới có lượt
> mới.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác biệt: **rate limit** giới hạn *số request* trong 60 giây gần nhất (đơn vị
> là lượt gọi, trả 429), bảo vệ hệ thống khỏi bị spam hay quá tải. **Cost
> guard** giới hạn *số tiền* đã tiêu trong tháng (đơn vị USD, lưu ở
> `cost:{user}:{tháng}`, trả 402), bảo vệ ngân sách.
>
> - **Rate limit cho qua, cost guard chặn**: một user gọi đều đặn 5
>   request/phút (luôn dưới 10/phút), nhưng câu hỏi dài và lịch sử hội thoại
>   lớn nên mỗi request tốn nhiều token. Sau nhiều ngày, tổng `cost_usd` trong
>   tháng chạm `MONTHLY_BUDGET_USD = 10` → request tiếp theo bị 402 dù tốc độ
>   gọi hoàn toàn bình thường.
> - **Cost guard cho qua, rate limit chặn**: một user bắn 15 request ngắn liên
>   tiếp trong vài giây. Mỗi request chỉ tốn khoảng 2e-05 USD nên ngân sách
>   còn gần như nguyên, nhưng từ request thứ 11 trở đi bị 429. Tôi thấy đúng
>   điều này khi chạy lệnh kiểm tra trên Railway: 9 lần `200` rồi 6 lần `429`
>   (lượt thứ 10 đã dùng ở lệnh gọi ngay trước đó).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Giả sử gộp thành một `/health` có ping Redis, và orchestrator dùng nó làm
> liveness probe (restart container khi probe fail liên tiếp):
> 1. **t = 0s**: Redis mất kết nối. Cả 3 container thật ra vẫn khỏe — process
>    chạy bình thường, chỉ dependency bên ngoài có vấn đề.
> 2. **Lần probe kế tiếp**: cả 3 container cùng lúc ping Redis thất bại →
>    `/health` trả 503. Load balancer thấy cả 3 đều lỗi và bỏ hết khỏi vòng
>    xoay → **mọi** request đều 502, kể cả request không cần Redis.
> 3. **Sau vài lần fail liên tiếp** (vd. 3 lần × 10s ≈ 30s): orchestrator kết
>    luận cả 3 container "chết" và **restart đồng loạt cả 3**. Request đang
>    chạy dở bị cắt, cụm còn 0 instance.
> 4. **t ≈ 30s**: Redis trở lại, nhưng các container đang khởi động lại nên
>    chưa nhận traffic được. Restart không sửa được gì vì lỗi nằm ở Redis chứ
>    không ở container.
> 5. Kết quả: sự cố Redis 30 giây biến thành downtime toàn cụm lâu hơn 30 giây.
>    Nếu Redis chập chờn, cụm rơi vào vòng lặp restart liên tục.
>
> Khi tách như code hiện tại: `/health` không đụng Redis nên vẫn 200 → không
> container nào bị restart; chỉ `/ready` trả 503 `{"redis": false}` → load
> balancer tạm ngưng đẩy traffic. Khi Redis về, `/ready` tự trả 200 và các
> instance vào lại vòng xoay ngay, không mất thời gian khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trả lời: `history_length` sẽ chỉ tăng trong phạm vi container nhận request. Mỗi khi check lại, container khác sẽ không thấy lịch sử của request trước đó, vì dict Python chỉ tồn tại trong RAM của container đó.
---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi gặp hai lỗi khi deploy: đầu tiên app crash ngay lúc start vì thiếu biến
> môi trường (log báo `KeyError`/`os.getenv(...)` trả `None`), nguyên nhân là
> tôi chưa set hết các biến bắt buộc trên dashboard của cloud platform như đã
> khai trong `.env.example`. Sau khi bổ sung đủ biến thì app chạy được nhưng
> `/ready` lại trả 503 với lỗi kết nối Redis (`ConnectionError`), do
> `REDIS_URL` bị để trống hoặc sai định dạng. Tôi xác định nguyên nhân bằng
> cách đọc log ứng dụng và gọi thử `/ready` để xem field nào báo `false`. Cách
> sửa là cấu hình đúng `REDIS_URL` theo format `redis://user:pass@host:port`
> trỏ tới Redis instance thật trên cloud, rồi test lại bằng `redis-cli -u
> $REDIS_URL ping`. Bài học rút ra là nên validate toàn bộ biến môi trường bắt
> buộc và test kết nối tới các service phụ thuộc (Redis, DB) ngay khi app
> khởi động, thay vì để lỗi lộ ra ở runtime.
