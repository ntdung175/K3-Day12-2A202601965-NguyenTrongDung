# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder (dòng bắt đầu bằng dấu `>` in nghiêng)
> bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trọng Dũng  
> Mã học viên: 2A202601965

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: mình deploy lên Render nhưng quên set biến `AGENT_API_KEY` trong
> dashboard. Nếu `Settings` để mặc định `"changeme"`, container vẫn khởi động
> bình thường, health check xanh, mình tưởng mọi thứ ổn. Nhưng lúc đó `/ask`
> đang chấp nhận key `"changeme"` — bất kỳ ai đọc source hoặc đoán ra chuỗi
> mặc định phổ biến này đều gọi được API và đốt token/tiền của mình, và mình
> chỉ phát hiện khi thấy chi phí lạ hoặc log truy cập bất thường. Với fail-fast
> (không mặc định), thiếu biến thì `Settings()` ném `ValidationError` ngay lúc
> khởi động → container không bao giờ lên "Live", health đỏ → mình biết ngay
> phải vào dashboard thêm biến. Chết to lúc deploy an toàn hơn nhiều so với
> chạy sai âm thầm bằng một cái khóa mà ai cũng biết.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON thật mình thu được:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:17:47.180172+00:00", "user_id": "sv-rate", "tokens_in": 305, "tokens_out": 45, "cost_usd": 7.275e-05}`
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và cộng dồn theo trường**: vì mỗi trường là một khóa riêng, mình có
>    thể lọc tất cả log của `user_id="sv-rate"` rồi cộng `cost_usd` để biết
>    user nào tốn tiền nhất trong tháng, hoặc lọc `level="error"` để soi lỗi.
>    Với `print` là text tự do thì máy không tách được trường để lọc/cộng.
> 2. **Tự động tạo dashboard và cảnh báo**: Render/Datadog/Grafana parse được
>    JSON nên vẽ biểu đồ số request theo thời gian, hoặc đặt cảnh báo kiểu
>    "báo khi `tokens_out` vượt ngưỡng". Text `print` chỉ để người đọc bằng
>    mắt, máy không dựng cảnh báo tự động từ nó được.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.43 GB đến từ hai nguồn. Thứ nhất là base image: bản 1
> stage dùng `python:3.11` full (mang theo compiler gcc/g++, git, các thư viện
> `-dev`, tài liệu và nhiều công cụ hệ thống), còn multi-stage dùng
> `python:3.11-slim` đã lột sạch những thứ đó. Thứ hai là rác của quá trình
> build: ở bản 1 stage, pip cache và các gói/công cụ chỉ cần lúc `pip install`
> vẫn nằm lại trong image cuối; còn multi-stage để chúng ở stage `builder` và
> chỉ `COPY --from=builder` phần thư viện đã cài sang stage runtime, nên
> compiler và rác build không lên image production. Image runtime nhỏ hơn ~6
> lần → deploy nhanh hơn, ít bề mặt tấn công hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của mình để `COPY requirements.txt` + `RUN pip install` trước, rồi
> mới `COPY app/ app/`. Khi sửa một ký tự trong `app/main.py` và build lại:
> các layer được **dùng lại từ cache** gồm `FROM ... AS builder`,
> `COPY requirements.txt`, `RUN pip install` (vì `requirements.txt` không đổi),
> `FROM python:3.11-slim` runtime và `COPY --from=builder /install`. Layer
> phải **chạy lại** là từ `COPY app/ app/` trở đi — nhưng chỉ là copy code và
> tạo user, rất nhanh (vài giây). Bước nặng `pip install` không phải chạy lại.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: sửa một dòng code làm layer
> `COPY . .` thay đổi → Docker phải chạy lại **mọi layer sau nó**, bao gồm cả
> `RUN pip install` → cài lại toàn bộ thư viện từ đầu, build từ vài giây thành
> vài phút. Đó là lý do phải để thứ ít đổi (requirements) lên trên, thứ hay đổi
> (source) xuống dưới.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của mình có lỗ hổng — ví dụ nhận input không
> kiểm rồi để nó chạy thành lệnh hệ thống (RCE). (2) Kẻ tấn công lợi dụng lỗ
> hổng để chạy lệnh tùy ý *bên trong container*, với quyền của process — nếu
> container chạy bằng root thì đó là **root trong container**. (3) Từ root
> trong container, kẻ tấn công có nhiều đường leo ra host: khai thác lỗ hổng
> kernel, lạm dụng Docker socket nếu bị mount vào, ghi đè file qua volume mount
> chung, hoặc thoát namespace. (4) Kết quả là chiếm được quyền cao trên máy
> host, ảnh hưởng cả các container/dịch vụ khác.
>
> Lệnh `USER appuser` cắt chuỗi ở bước (2)→(3): sau `USER`, process chạy bằng
> user thường không đặc quyền. Dù kẻ tấn công chiếm được shell qua lỗ hổng ở
> bước (2), nó chỉ là `appuser` — không đọc/ghi được file của root, không dùng
> được các kỹ thuật leo thang đòi quyền root. Chuỗi bị chặn ngay từ trong
> container, không lan tới host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt: với cách đếm theo phút đồng hồ
> (reset lúc giây :00), mình gửi 10 request vào cuối phút — ví dụ lúc
> 10:00:59 — vẫn nằm trong hạn mức của "ô phút" thứ nhất. Ngay khi đồng hồ
> sang phút mới (10:01:00), bộ đếm reset về 0, mình gửi tiếp 10 request nữa lúc
> 10:01:00. Hai cụm cách nhau chưa tới 2 giây nhưng tổng cộng là 20 request,
> vì chúng rơi vào hai ô phút khác nhau nên mỗi ô đều thấy "chỉ 10, đúng luật".
>
> Sliding window 60 giây của mình không bị lỗ hổng này: nó đếm 60 giây *gần
> nhất* bất kể mốc đồng hồ, nên tại 10:01:00 nó vẫn thấy 10 request của
> 10:00:59 còn trong cửa sổ → chặn ngay ở request thứ 11. Mình đã kiểm chứng
> thực tế: bắn 12 request liên tiếp với hạn mức 10 → nhận `200` mười lần rồi
> `429` hai lần cuối.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau: rate limit giới hạn *số lượng* request trong một khoảng thời gian
> (bảo vệ hạ tầng khỏi bị dội quá nhanh), còn cost guard giới hạn *số tiền*
> token tích lũy theo tháng cho mỗi user (bảo vệ ngân sách). Một cái đo tốc độ,
> một cái đo tiền.
>
> Rate limit cho qua nhưng cost guard chặn: user gửi đúng 1 request mỗi phút
> (không vi phạm tốc độ) nhưng mỗi request là một tài liệu 100 trang tốn cực
> nhiều token → chi phí tích lũy vượt `monthly_budget_usd` → cost guard trả
> `402 Payment Required`, dù rate limit thấy hoàn toàn bình thường.
>
> Ngược lại — cost guard cho qua nhưng rate limit chặn: user gửi 100 request
> mỗi phút toàn câu một chữ như "hi" → mỗi request gần như miễn phí, tổng chi
> phí chưa chạm budget nên cost guard cho qua, nhưng vượt hạn mức 10 request/
> phút nên rate limit trả `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` và `/ready` thành một endpoint có kiểm Redis, khi Redis
> mất kết nối 30 giây, thứ tự sự kiện là:
> 1. Redis rớt → endpoint gộp (đóng vai liveness) bắt đầu trả `503` trên cả 3
>    container cùng lúc.
> 2. Orchestrator hiểu `503` của liveness = "container đã chết" → ra lệnh
>    **restart** cả 3 container (chứ không chỉ ngừng đẩy traffic).
> 3. Cả cụm cùng restart một lúc → service gián đoạn hoàn toàn, không còn
>    container nào phục vụ, kể cả những request lẽ ra vẫn xử lý được.
> 4. Container khởi động lại; nếu Redis vẫn chưa hồi (còn trong 30 giây đó),
>    liveness lại `503` → restart lặp vô tận (crash loop).
>
> Tức là một sự cố Redis tạm thời — lẽ ra chỉ nên tạm ngắt traffic — bị khuếch
> đại thành restart toàn cụm. Tách hai endpoint tránh được điều này: `/health`
> không đụng Redis nên vẫn `200` (container không bị restart oan), còn `/ready`
> trả `503` để load balancer tạm ngừng gửi request mới, và tự phục hồi khi
> Redis sống lại — không container nào bị giết.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy `docker compose up -d --scale agent=3` và gọi `/ask` 3 lần với cùng
> `X-User-Id: sv-demo` qua Nginx (cổng 8000): `history_length` trả về lần lượt
> **0, 2, 4** — tăng đều đặn (mỗi lượt thêm 2 message: câu hỏi của user + câu
> trả lời của assistant). Lý do: cả 3 container đọc/ghi cùng một Redis, nên dù
> Nginx round-robin đẩy mỗi request sang một container khác nhau, chúng vẫn
> nhìn thấy chung lịch sử. Mình còn kiểm bằng `redis-cli LLEN history:sv-demo`
> = 6, đúng bằng 3 lượt × 2 message.
>
> Nếu lịch sử lưu trong một dict Python (nằm trong RAM của từng process), con
> số sẽ nhảy lung tung thay vì tăng đều: mỗi container chỉ nhớ những request
> rơi vào chính nó, nên `history_length` có thể là 0, 0, 2 hoặc 0, 2, 0... tùy
> request rơi vào đâu — agent "mất trí nhớ" giữa các lượt, và chỉ cần một
> container restart là mất sạch phần của nó. Đó chính là lý do state phải nằm
> ngoài process, trong Redis, để service stateless và scale ngang được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trở ngại mình gặp: lần đầu gọi `curl https://day12-agent-dq83.onrender.com/health`
> ngay sau khi deploy, lệnh **treo rất lâu** (gần một phút) rồi mới trả `200` —
> ban đầu trông như health check bị timeout / service chết.
>
> Cách tìm nguyên nhân: mình mở tab **Logs** trên dashboard Render và thấy dòng
> báo instance đang "spinning up / starting" đúng lúc đó, và khi gọi lần thứ
> hai thì `/health` trả `200` gần như tức thì. Từ đó mình hiểu đây không phải
> lỗi code mà là đặc tính **free tier ngủ đông**: Render tắt container khi
> không có traffic, request đầu tiên phải chờ nó khởi động lại (cold start).
>
> Cách xử lý: đây là hành vi bình thường của gói free nên không cần sửa code —
> mình chỉ cần chờ request đầu, hoặc gọi trước một nhịp để "đánh thức" service
> rồi mới đo. Ngoài ra, nhờ Dockerfile đã đọc cổng động qua `${PORT:-8000}`,
> mình **tránh được** lỗi kinh điển hơn là app nghe cố định cổng 8000 trong khi
> Render gán cổng khác → health check không bao giờ pass.
