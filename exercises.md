# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Duy Hoanh  Mã học viên: 01777

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Deploy lên Railway/Render nhưng quên set biến môi trường `AGENT_API_KEY`. Nếu không có fail fast, app khởi động bình thường với key mặc định `"changeme"` — bất kỳ ai biết key mặc định này đều gọi được API, LLM token bị tiêu thụ, hóa đơn tăng vọt mà mình không hay biết cho đến khi nhận bill cuối tháng. Với fail fast, app chết ngay lúc khởi động, container restart liên tục → Railway/Render báo lỗi deploy → mình thấy ngay log `ValidationError: agent_api_key field required` → set biến môi trường rồi deploy lại. Phát hiện trong 5 phút thay vì sau nhiều ngày bị khai thác.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:45:12.123456+00:00", "user_id": "sv01", "tokens_in": 42, "tokens_out": 128, "cost_usd": 0.00025}`
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc/tìm kiếm theo trường cụ thể**: Trong Grafana/Datadog, có thể query `user_id = "sv01"` hoặc `cost_usd > 0.001` để tìm những request tốn tiền nhất — dòng print thuần túy không có cấu trúc nên không lọc được.
> 2. **Tạo dashboard/alert tự động**: Hệ thống monitoring parse được JSON → tự động vẽ biểu đồ tokens_in/tokens_out theo thời gian, đặt cảnh báo khi cost_usd tăng đột biến. Với `print("đã trả lời xong")` thì máy không biết tách thông tin nào ra từ chuỗi text.

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
| 1 stage (bản đầu) | ~350 MB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch (~170 MB) chủ yếu là build dependencies: pip cache, các header file C dùng để compile native extensions, compiler (gcc), và các package chỉ cần lúc build (wheel, setuptools). Trong bản 1-stage, tất cả nằm lại trong image cuối. Với multi-stage, stage builder cài dependencies vào `/install`, stage production chỉ COPY kết quả đã cài xong — các file build trung gian, pip cache, compiler đều ở lại stage builder và không đi vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (COPY requirements.txt → RUN pip install → COPY app/ utils/): khi chỉ sửa `app/main.py`, layer `COPY requirements.txt .` và `RUN pip install` được dùng lại từ cache (vì requirements.txt không đổi). Chỉ layer `COPY app/ app/` phải chạy lại. Build nhanh chỉ mất vài giây.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: mỗi khi sửa bất kỳ file nào (kể cả 1 ký tự trong main.py), layer `COPY . .` bị invalidate → `RUN pip install` cũng phải chạy lại từ đầu (vì nó nằm sau layer bị invalidate). Mỗi lần build phải tải và cài lại toàn bộ dependencies, mất vài phút thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python có lỗ hổng RCE (Remote Code Execution), ví dụ endpoint nhận input mà dùng `eval()` hoặc lỗi deserialization. (2) Kẻ tấn công gửi payload qua HTTP, code thực thi lệnh tùy ý bên trong container. (3) Vì process chạy bằng root trong container, kẻ tấn công có quyền root trong container — đọc/ghi mọi file, cài công cụ, mount filesystem. (4) Nếu Docker daemon có lỗ hổng hoặc container share kernel với host, root trong container có thể escape ra host với quyền root — kiểm soát toàn bộ máy chủ.
>
> Lệnh `USER appuser` cắt đứt chuỗi ở bước (3): process chạy dưới quyền user thường (uid 1001, không có sudo), nên dù kẻ tấn công RCE được, họ chỉ có quyền hạn chế — không cài thêm package, không đọc file nhạy cảm, và đặc biệt không thể thực hiện container escape vì các kỹ thuật đó yêu cầu quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request trong 2 giây liên tiếp.** Cách đạt được: gửi 10 request vào lúc 10:00:59 (cuối phút 10:00). Bộ đếm reset về 0 lúc 10:01:00. Gửi thêm 10 request vào lúc 10:01:01 (đầu phút 10:01). Cả hai đợt đều "hợp lệ" vì mỗi đợt nằm trong một phút đồng hồ riêng, nhưng thực tế user đã gửi 20 request trong vòng 2 giây (từ 10:00:59 đến 10:01:01). Sliding window không có lỗ hổng này vì nó luôn nhìn lại 60 giây gần nhất — 10 request lúc 10:00:59 vẫn còn trong cửa sổ khi đếm lúc 10:01:01, nên request thứ 11 bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau**: Rate limit giới hạn *tần suất* (số request/phút), bất kể mỗi request tốn bao nhiêu. Cost guard giới hạn *chi phí tích lũy* (USD/tháng), bất kể gửi bao nhiêu request.
>
> **Rate limit cho qua, cost guard chặn**: User gửi 3 request/phút (dưới hạn 10), nhưng mỗi request dùng prompt rất dài (50,000 tokens), chi phí mỗi request cao. Sau vài ngày, tổng chi phí tháng vượt budget $10 → cost guard chặn dù rate limit vẫn cho qua.
>
> **Cost guard cho qua, rate limit chặn**: Đầu tháng mới, user chưa tốn đồng nào (spent = $0, dưới budget $10), nhưng trong 60 giây vừa rồi user đã gửi liên tục 10 request → request thứ 11 bị rate limit chặn với 429, dù cost guard vẫn thấy còn dư ngân sách.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối → endpoint gộp (vừa liveness vừa readiness) kiểm tra Redis thất bại → trả 503.
> 2. Orchestrator (K8s/Docker) thấy liveness probe fail → đánh dấu container "unhealthy".
> 3. Sau vài lần retry (ví dụ 3 lần × 30s interval), orchestrator kết luận container đã chết → **kill và restart** cả 3 container.
> 4. Container mới khởi động → vẫn không kết nối được Redis (Redis chưa hồi phục) → liveness fail tiếp → bị kill lại → **restart loop**.
> 5. Toàn bộ cụm 3 container rơi vào trạng thái CrashLoopBackOff, không container nào phục vụ được request, dù bản thân process Python hoàn toàn khỏe mạnh.
>
> Với thiết kế tách riêng: `/health` (liveness) không kiểm tra Redis nên luôn trả 200 → container không bị restart. `/ready` (readiness) trả 503 → load balancer ngừng gửi traffic vào, nhưng container vẫn sống. Khi Redis hồi phục sau 30 giây, `/ready` trả 200 lại → traffic tự động quay lại, không mất container nào.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless đúng): `history_length` tăng đều 0, 2, 4, 6, 8 (mỗi request thêm 2 message: user + assistant) bất kể request rơi vào container nào, vì cả 3 container cùng đọc/ghi vào một Redis chung.
>
> Với dict Python (stateful sai): `history_length` sẽ nhảy lung tung. Ví dụ Nginx round-robin gửi request 1 → container A (history_length=0), request 2 → container B (history_length=0, vì B không biết gì về request 1), request 3 → container C (history_length=0), request 4 → container A lại (history_length=2, chỉ nhớ request 1). Con số không tăng đều mà phụ thuộc request rơi vào container nào — agent bị "mất trí nhớ" mỗi khi đổi container. Nếu container bị restart, mất toàn bộ lịch sử luôn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp**: Health check timeout — sau khi deploy lên Railway, container khởi động nhưng bị đánh dấu unhealthy sau 90 giây, tự restart liên tục.
>
> **Thông báo lỗi**: Railway dashboard hiện "Health check failed" và log cho thấy `ConnectionRefusedError: [Errno 111] Connection refused` khi health check gọi tới port 8000.
>
> **Nguyên nhân**: Railway inject biến `$PORT` với giá trị khác 8000 (ví dụ 3000 hoặc 443), nhưng CMD trong Dockerfile ban đầu hard-code `--port 8000`. Health check của Railway gọi vào `$PORT` nhưng app lại listen trên 8000 → connection refused.
>
> **Cách sửa**: Đổi CMD thành `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để đọc biến `$PORT` từ môi trường, fallback về 8000 khi chạy local. Deploy lại → app listen đúng port → health check pass.
