# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lưu Mạnh Hùng  Mã học viên: L3A202602942

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tôi deploy service lên Railway, tôi phải set biến `AGENT_API_KEY` thủ
> công trong dashboard/CLI — nếu tôi quên set (rất dễ xảy ra vì Railway không
> tự đọc file `.env` của repo, mình phải set lại từng biến bằng tay), app sẽ
> raise `ValidationError` ngay lúc khởi động và container rớt liên tục, log
> báo rõ thiếu field nào. Tôi biết ngay và sửa trước khi ai gọi được service.
> Nếu `agent_api_key` có mặc định `"changeme"`, app vẫn khởi động bình thường,
> healthcheck vẫn xanh, nhưng ai cũng gọi được `/ask` bằng key `"changeme"` —
> tôi sẽ không biết mình đang để hở API cho tới khi nhận hóa đơn LLM bất
> thường hoặc bị lạm dụng, lúc đó đã tốn tiền và dữ liệu đã bị truy cập.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ service khi tôi gọi `/ask` qua curl:
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:57:46+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
> Hai việc làm được mà `print` thường không làm được:
> 1. **Lọc/truy vấn theo trường** — vì log là JSON có cấu trúc, tôi có thể
>    `grep`/`jq` để lọc mọi dòng có `"user_id":"sv-test"` hoặc tính tổng
>    `cost_usd` của một user trong ngày mà không cần parse chuỗi bằng regex
>    thủ công như với `print`.
> 2. **Cảnh báo/thống kê tự động** — một hệ thống như Datadog/Grafana có thể
>    đọc trực tiếp field `level` để đếm số dòng `error` trong 5 phút và bắn
>    cảnh báo, hoặc vẽ biểu đồ `cost_usd` cộng dồn theo thời gian. Với
>    `print("đã trả lời xong")`, máy không biết đâu là số tiền, đâu là user,
>    phải có người đọc bằng mắt.

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
| 1 stage (bản đầu) | 1700 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build một Dockerfile 1-stage dùng `FROM python:3.11` (bản đầy đủ, không
> phải `-slim`), `COPY . .` rồi `pip install` ngay trong cùng stage, không
> tách builder/runtime — kết quả **1.7GB**. Với `Dockerfile` multi-stage thực
> tế của tôi (build ở CP2, `FROM python:3.11-slim AS builder` / `AS runtime`,
> chỉ copy `/install` từ builder sang runtime) — kết quả **271MB**, tức chênh
> lệch khoảng **1.43GB**.
> Phần chênh lệch đó chủ yếu là: (1) `python:3.11` đầy đủ mang theo rất nhiều
> gói hệ thống, header, công cụ biên dịch (gcc, make, các thư viện dev...)
> chỉ cần lúc `pip install` chứ không cần lúc chạy — bản `-slim` không có các
> gói này; (2) do multi-stage chỉ `COPY --from=builder /install /usr/local`,
> mọi build-cache, file tạm của pip, và toàn bộ stage `builder` (kể cả
> compiler) bị bỏ lại, không lọt vào image cuối cùng; (3) bản 1-stage
> `COPY . .` mang theo cả những file không cần cho runtime (`.git` nếu không
> có `.dockerignore` tốt, test, tài liệu...) trong khi bản multi-stage chỉ
> `COPY app ./app` và `COPY utils ./utils` — đúng những gì cần chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại của tôi, stage `builder` chỉ có `COPY requirements.txt`
> rồi `RUN pip install --prefix=/install`. Khi tôi sửa `app/main.py`, Docker so
> sánh checksum từng layer: `requirements.txt` không đổi nên toàn bộ stage
> `builder` (kể cả layer `pip install`, vốn là layer tốn thời gian nhất) được
> lấy lại từ cache, không chạy lại. Ở stage `runtime`, chỉ có layer
> `COPY app ./app` trở đi (COPY app, useradd...) phải build lại vì nội dung
> thư mục `app/` đã đổi.
> Nếu tôi đặt `COPY . .` lên **trước** `RUN pip install` (như bản gốc lab đưa
> cho lúc chưa sửa), thì mọi lần sửa một dòng code — dù chỉ đổi 1 ký tự trong
> `main.py` — cũng làm checksum của layer `COPY . .` đổi, kéo theo layer
> `pip install` NGAY SAU nó bị coi là "cần chạy lại" dù `requirements.txt`
> không hề thay đổi. Kết quả là mỗi lần sửa code, Docker cài lại toàn bộ
> dependency từ đầu, build chậm đi rất nhiều so với chỉ mất vài giây copy code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của tôi có một lỗ hổng, ví dụ endpoint nhận
> input rồi truyền vào một lệnh shell hoặc deserialize không an toàn (RCE) →
> (2) kẻ tấn công khai thác lỗ hổng đó, chạy được lệnh tùy ý *bên trong*
> process của container → (3) nếu process đó đang chạy bằng `root` (UID 0),
> lệnh tùy ý đó cũng chạy với quyền root bên trong container → (4) container
> chỉ cô lập bằng namespace/cgroup chứ không phải một máy ảo thật; nếu có thêm
> một lỗ hổng thứ hai ở tầng container runtime, hoặc container được chạy với
> cấu hình thiếu an toàn (mount `/var/run/docker.sock`, `--privileged`,
> capability thừa...), quyền root bên trong container có thể leo thang thành
> quyền cao trên chính máy host.
> Lệnh `USER appuser` cắt đứt chuỗi này ngay ở bước (3): dù kẻ tấn công khai
> thác được lỗ hổng ở bước (1)-(2), lệnh họ chạy được chỉ có quyền của user
> thường `appuser` (UID 10001) bên trong container — không đủ quyền ghi vào
> các thư mục hệ thống, không đủ quyền để tận dụng những kỹ thuật leo thang
> đặc quyền vốn đòi hỏi UID 0, nên thiệt hại bị giới hạn lại nhiều so với khi
> chạy bằng root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đúng 10 request vào
> giây 10:00:59 (cuối phút thứ nhất, bộ đếm của phút đó vẫn còn quota đầy đủ
> vì chưa dùng lần nào), rồi gửi tiếp 10 request nữa vào giây 10:01:01 — ngay
> khi đồng hồ hệ thống vừa sang phút mới, bộ đếm reset về 0 nên 10 request này
> lại được tính vào "hạn mức của phút mới", hoàn toàn hợp lệ theo logic đếm
> theo phút đồng hồ. Vậy chỉ trong cửa sổ thời gian thực tế dài 2 giây
> (10:00:59 → 10:01:01), user đã gửi được 20 request dù hạn mức danh nghĩa là
> 10/phút — đúng lỗ hổng mà sliding window (đếm 60 giây gần nhất tính từ thời
> điểm hiện tại, không neo vào mốc giây 00) được thiết kế để chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số lượng** request trong một cửa sổ thời gian, không quan
> tâm mỗi request tốn bao nhiêu tiền. Cost guard đếm **tổng số tiền** đã tiêu
> trong tháng, không quan tâm số lượng request. Hai trục hoàn toàn độc lập.
>
> - **Rate limit cho qua, cost guard phải chặn**: user gọi `/ask` với câu hỏi
>   rất dài, mỗi request tốn nhiều token/tiền, nhưng chỉ gọi 3 lần/phút —
>   dưới hạn mức `rate_limit_per_minute=10` nên rate limit không chặn; tuy
>   nhiên nếu 3 request đó đã đủ tốn hết `monthly_budget_usd` của tháng
>   (ví dụ ngân sách nhỏ, câu hỏi rất dài), cost guard sẽ trả 402 dù rate
>   limit hoàn toàn cho qua.
> - **Cost guard cho qua, rate limit phải chặn**: user gửi 15 request/phút
>   liên tục nhưng mỗi câu hỏi rất ngắn (gần như miễn phí, ví dụ "hi"), tổng
>   chi phí cả tháng vẫn rất nhỏ so với ngân sách — cost guard không có lý do
>   để chặn — nhưng vì vượt `rate_limit_per_minute=10`, rate limiter vẫn trả
>   429 từ request thứ 11 trở đi, bất kể chi phí có rẻ tới đâu.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối → (2) `/health` (đã gộp với `/ready`)
> gọi `store.ping()`, thất bại, cả 3 container cùng lúc trả 503 cho endpoint
> mà đáng lẽ chỉ dùng để kiểm tra "process còn sống" → (3) orchestrator
> (Docker/Railway/K8s) đọc 503 đó như tín hiệu liveness probe thất bại, nghĩ
> rằng process bị treo/chết chứ không phải một dependency bên ngoài đang lỗi
> → (4) orchestrator restart cả 3 container gần như đồng thời để "sửa" —
> nhưng khởi động lại app không hề khắc phục được việc Redis đang chết, nên
> ngay khi container mới lên và gọi lại `/health`, nó vẫn thấy Redis chết và
> tiếp tục trả 503 → (5) orchestrator lại restart tiếp, tạo thành vòng lặp
> crash-restart liên tục (crash loop) suốt 30 giây Redis còn down, và trong
> lúc đó service hoàn toàn không phục vụ được request nào — tệ hơn nhiều so
> với việc chỉ `/ready` báo not-ready (loại instance khỏi load balancer) còn
> `/health` vẫn báo process sống bình thường, không kích hoạt restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi đã chạy `docker compose up -d --scale agent=3` (kèm nginx làm load
> balancer round-robin trước 3 container `agent`), rồi gọi `/ask` 3 lần liên
> tiếp qua nginx với cùng `X-User-Id: stateless-test`. Kết quả `history_length`
> nhận được là **0 → 2 → 4** — tăng đều đặn đúng như một service duy nhất,
> mặc dù mỗi request có thể rơi vào một trong 3 container khác nhau (nginx
> không đảm bảo "sticky session" theo user).
> Nếu lịch sử được lưu trong một `dict` Python trong RAM của từng process thay
> vì Redis, mỗi container sẽ có một bản dict riêng, hoàn toàn không đồng bộ
> với nhau. Kết quả `history_length` khi đó sẽ "nhảy lung tung" thay vì tăng
> đều: ví dụ request 1 rơi vào container A → A tự thấy `history_length=0`;
> request 2 rơi vào container B (chưa từng thấy user này) → B cũng trả về
> `history_length=0` thay vì `2`; nếu tình cờ request 3 rơi lại vào A thì A
> mới thấy `history_length=2`. Agent trông như "mất trí nhớ" ngẫu nhiên tùy
> vào con xúc xắc load balancer rơi vào container nào — đúng cái mà thiết kế
> stateless (state nằm ở Redis) tránh được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi link service `day12-agent` trên Railway và set biến môi trường,
> service build xong nhưng trạng thái báo **Failed**. Tôi chạy
> `railway logs --deployment` và thấy traceback lặp lại liên tục:
> ```
> File "/app/app/lifecycle.py", line 59, in install
>     raise NotImplementedError("TODO (CP4): cài đặt install")
> NotImplementedError: TODO (CP4): cài đặt install
> ```
> Nguyên nhân: Railway build trực tiếp từ nhánh `main` trên GitHub, nhưng tôi
> mới sửa code CP1–CP4 ở máy local, chưa `git commit`/`git push` — nên Railway
> vẫn đang build phiên bản code cũ (còn nguyên `NotImplementedError`) chứ
> không phải code tôi vừa sửa. Tôi xác nhận bằng `git status` local, thấy
> đúng là các file trong `app/` đang ở trạng thái "modified" chưa commit.
> Cách sửa: commit toàn bộ thay đổi và `git push origin main`. Railway tự
> động phát hiện push mới, build lại, và lần này deploy thành công
> (`day12-agent: ● Online`), `/health` và `/ready` đều trả 200.
