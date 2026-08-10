# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoàng Đạt  
> Mã học viên: 2A202601460

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy lên Render, nếu mình quên điền `API_TOKEN` trong dashboard thì
> service sẽ crash ngay lúc khởi động và Render báo "Deploy failed" — mình
> biết ngay và sửa được trước khi ai gọi vào service. Nếu `api_token` có mặc
> định `"changeme"`, app vẫn khởi động và trả lời request bình thường; vì
> repo là public nên bất kỳ ai đọc code cũng biết token mặc định, gọi thẳng
> vào `/chat` mà không cần đoán gì cả. Mình sẽ không phát hiện ra chuyện này
> cho tới khi nhìn thấy `usd_cost` cộng dồn bất thường trong log hoặc nhận
> hóa đơn — tức là đã mất tiền rồi mới biết có lỗ hổng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:16:27.923310+00:00", "client_id": "sv-exercise", "prompt_tokens": 3, "completion_tokens": 41, "usd_cost": 2.505e-05}
> ```
> Hai việc mình làm được mà `print()` không làm được:
> 1. Lọc theo `client_id` để tính tổng `usd_cost` của riêng một client trong
>    ngày — chỉ cần `grep`/query theo key `client_id`, còn với `print()` thì
>    phải tự viết regex đoán vị trí tên client trong câu văn.
> 2. Lọc theo `severity` để dựng cảnh báo (ví dụ: đếm số dòng `ERROR` trong 5
>    phút gần nhất) — công cụ log của Render/Railway hiểu field `severity`
>    sẵn để tô màu và lọc, còn `print()` không có cấu trúc để máy phân biệt
>    dòng nào là lỗi, dòng nào là thông tin bình thường.

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
| 1 stage (bản đầu) | 1.77 GB |
| Multi-stage | 316 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.45GB. Phần lớn là do bản 1-stage dùng base image
> `python:3.11` đầy đủ (kèm theo compiler, header C, các thư viện dựng sẵn
> cho nhiều mục đích) thay vì `python:3.11-slim`, cộng với việc mọi thứ dùng
> để cài đặt (cache pip, build tool) đều nằm lại trong layer cuối cùng vì
> chỉ có một stage. Bản multi-stage tách hẳn phần "cài đặt" (build-essential,
> pip cache, mã nguồn tạm) sang stage `builder` riêng, rồi stage `runtime`
> chỉ `COPY --from=builder /install /usr/local` — tức là chỉ mang theo đúng
> package đã cài xong, không mang theo bất cứ công cụ nào dùng để cài chúng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, `COPY requirements.txt .` và `RUN pip install`
> nằm ở stage `builder`, trước cả khi `app/` được copy vào — nên khi mình chỉ
> sửa `app/main.py`, layer `pip install` vẫn dùng lại cache (input của nó,
> tức nội dung `requirements.txt`, không đổi), Docker chỉ chạy lại từ layer
> `COPY app ./app` trở đi. Build lại chỉ mất vài giây thay vì tải lại toàn bộ
> dependency.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`, thì bất kỳ thay đổi nào
> trong toàn bộ source code (kể cả một dấu phẩy trong `main.py`) đều làm
> layer `COPY . .` bị đổi hash → Docker huỷ cache từ layer đó trở đi, kéo
> theo `pip install` phải chạy lại từ đầu dù `requirements.txt` không hề đổi.
> Với repo này build lại sẽ tốn thêm khoảng 20 giây mỗi lần chỉ vì sửa một
> dòng code không liên quan gì tới dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử `app/main.py` có một lỗ hổng cho phép chạy lệnh hệ thống tuỳ ý (ví
> dụ deserialize input không an toàn). Nếu container chạy bằng root, tiến
> trình uvicorn bị chiếm quyền đó cũng chạy với UID 0 bên trong container.
> Kẻ tấn công giờ có quyền ghi vào bất cứ file nào trong container, và nếu
> container có mount volume từ host hoặc có lỗ hổng thoát container (container
> escape, khá phổ biến khi kernel/daemon có bug), quyền root bên trong
> container có thể leo thang thành quyền root trên chính máy host.
>
> Lệnh `USER appuser` (trong `Dockerfile` của mình, UID 10001) cắt đứt chuỗi
> này ngay từ bước đầu: dù lỗ hổng vẫn tồn tại và kẻ tấn công vẫn chạy được
> lệnh, tiến trình đó chỉ có quyền của một user thường — không ghi được vào
> hầu hết filesystem hệ thống, và nếu có container escape thì thứ leo thang
> ra ngoài cũng chỉ là quyền user thường, không phải root.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là yêu cầu của chuẩn HTTP cho mọi response 401 —
> nó nói cho client (hoặc thư viện HTTP mà client dùng) biết phải xác thực
> theo kiểu nào để tự động thử lại đúng cách, thay vì client phải đoán.
>
> Trả cùng một thông báo cho cả ba trường hợp là để không "giúp" người đang
> dò token. Nếu response phân biệt "thiếu header" với "sai token", kẻ tấn
> công biết được khi nào mình đã gửi đúng định dạng và chỉ còn phải dò đúng
> giá trị token — tức là mình đã tự thu hẹp không gian cần dò cho họ. Trả
> chung một lỗi buộc họ phải thử mù hoàn toàn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Xô tối đa chứa 10 token, nên dù im lặng bao lâu, có `min(capacity, ...)`
> thì client cũng chỉ gửi được đúng **10 request liên tiếp** trước khi request
> thứ 11 bị 429.
>
> Nếu bỏ `min(...)`, token cứ cộng dồn theo thời gian: 10 phút × 10
> token/phút = **100 token** tích được, nên client sẽ gửi được khoảng 100
> request liên tiếp trước khi bị chặn — gấp 10 lần so với thiết kế ban đầu.
> Lý do là hàm `available()` không còn giới hạn trên, nên thời gian im lặng
> càng dài thì "hạn mức bùng nổ" càng lớn vô hạn, phá vỡ mục đích chống spam
> của rate limit.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng: nếu sự cố xảy ra đầu tháng và không ai để ý, client
> có thể tiêu tới gần $30 trước khi bị chặn — thiệt hại tối đa gần bằng cả
> ngân sách tháng. Service chỉ tự hồi phục khi sang tháng mới (reset theo
> chu kỳ tháng), có thể vài tuần sau sự cố.
>
> Với hạn mức $1/ngày: thiệt hại tối đa bị giới hạn còn khoảng $1 — đúng 1/30
> so với cách trên — vì khoá `spend:<client>:<ngày>` tự đổi mỗi ngày. Service
> tự hồi phục ngay 00:00 UTC hôm sau mà không cần ai can thiệp, vì key của
> ngày mới bắt đầu từ 0.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối.
> 2. Endpoint gộp (đóng vai trò cả liveness lẫn readiness) bắt đầu trả 503 ở
>    cả 3 container, vì cả 3 đều gọi `store.ping()` và đều thất bại.
> 3. Vì đây cũng là endpoint liveness, orchestrator hiểu 503 là "process
>    này hỏng, cần restart" — chứ không phải "tạm thời chưa sẵn sàng" — nên
>    nó restart cả 3 container gần như cùng lúc.
> 4. Trong lúc cả 3 đang restart, không còn container nào phục vụ được
>    request — toàn cụm downtime hoàn toàn, dù bản chất sự cố ban đầu chỉ là
>    Redis chập chờn 30 giây.
> 5. Redis phục hồi, nhưng 3 container vẫn đang trong quá trình khởi động
>    lại (kéo cold start, pull image, health check ban đầu...) nên thời gian
>    downtime thực tế còn dài hơn 30 giây gốc của sự cố Redis.
> Tách riêng /healthz (không đụng Redis) và /readyz (có đụng Redis) tránh
> được chuỗi này: Redis chết chỉ khiến /readyz báo 503, load balancer rút
> container khỏi vòng xoay chứ không restart, nên khi Redis sống lại thì
> container vẫn đang chạy sẵn, phục vụ lại ngay lập tức.

---

### Câu 10 — Deploy thật (CP5)

> không có, vì app khá đơn giản, ít service không bị config