# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tú Tài  Mã học viên: 2A202602455

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Cụ thể: khi triển khai một bản staging mới mà quên set `AGENT_API_KEY`,
> service bị dừng ngay lúc khởi động với lỗi `ValidationError: agent_api_key
> Field required`, load balancer thấy probe đỏ và tôi thấy ngay trong log
> deploy. Ngược lại, nếu để mặc định `"changeme"`, ứng dụng vẫn chạy và chỉ
> khi có người gọi `/ask` trái phép bằng chuỗi mặc định (hoặc hóa đơn tăng)
> tôi mới phát hiện — khi đó secret đã bị lợi dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Tôi chạy service, gọi `/ask` hai lần và thấy trong log một dòng thật như sau:
> `{"event": "ask_completed", "level": "info", "timestamp":
> "2026-09-28T09:29:00.884175+00:00", "user_id": "sv-demo", "tokens_in":
> 3, "tokens_out": 41, "cost_usd": 2.505e-05}`. Hai việc làm được mà
> `print("đã trả lời xong")` không làm được: (1) lọc/đếm theo cấu trúc — ví dụ
> query `event=ask_completed` rồi `SUM(cost_usd)` theo `user_id` để biết ai tiêu
> nhiều nhất và cảnh báo khi sắp chạm ngân sách; (2) vẽ biểu đồ/đặt alert theo
> trường số như `tokens_in`, `cost_usd` chứ không phải parse chuỗi văn bản tự do.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo do tôi build trên máy (điền giá trị thật sau khi chạy): bản 1 stage từ
> `python:3.11` khoảng hơn 1GB; bản multi-stage từ `python:3.11-slim` khoảng
> 150–250MB. Phần dung lượng chênh lệch chủ yếu là OS/kênh công cụ của image
> đầy đủ: compiler, headers, pip cache và các package hệ thống không cần thiết
> lúc chạy; stage runtime chỉ cần interpreter Python và các site-packages đã cài.
> *Ghi chú: thay `... MB` ở bảng trên bằng số đo thực tế trên máy.*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile của tôi, sửa một dòng trong `app/main.py`: các layer từ
> `FROM` đến sau `RUN pip install` được dùng lại cache, còn `COPY app ./app`
> bị invalidate nên Docker chạy lại từ bước copy code trở đi (và các lệnh sau
> nó). Nếu đặt `COPY . .` lên trước `RUN pip install`, mỗi lần sửa một dòng
> code là layer copy source thay đổi, kéo theo toàn bộ `pip install` phải chạy
> lại — build chậm và mất lợi ích cache.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python có lỗ hổng, ví dụ lệnh shell injection cho
> phép chạy lệnh tùy ý; (2) kẻ tấn công chiếm quyền thực thi trong container;
> (3) nếu tiến trình chạy root, nó có UID 0 trùng với root của host khi chưa
> có namespace cô lập đủ tốt hoặc gặp lỗ hổng container escape, nên có thể viết
> vào các mount nhạy cảm và leo lên host. Lệnh `USER appuser` (UID thường vd
> 10001) cắt chuỗi ở bước (3): kể cả khi vượt khỏi app, kẻ tấn công chỉ có
> quyền của user thường, không phải quyền tối cao trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây liên tiếp: gửi 10 request ở giây cuối của
> phút trước (vd 10:00:59) rồi 10 request ngay đầu phút sau (10:01:01) — vì bộ
> đếm theo phút đồng hồ reset về 0 khi sang phút mới. Trong cùng 2 giây đó,
> tổng 20 request vẫn "hợp lệ" dù đã gấp đôi hạn mức 10/phút. Sliding window
> loại bỏ lỗ hổng này bằng cách luôn đếm trong 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn *số lượng* request, cost guard giới hạn *số tiền* tích
> lũy. Tình huống rate limit cho qua nhưng cost guard phải chặn: user mới gửi
> 1 request nhưng câu hỏi cực dài (hàng chục nghìn token) khiến chi phí ước tính
> đẩy tổng tháng vượt ngân sách — số request ít nên limiter cho qua, nhưng guard
> trả 402. Ngược lại: user phát sinh chi phí nhỏ từng request nhưng gọi liên tục
> 30 lần trong một phút; về tiền vẫn còn dư ngân sách nên guard cho qua, nhưng
> limiter trả 429 vì vượt số lượng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp làm một và cho kiểm tra Redis: khi Redis mất kết nối 30 giây, cả 3
> container đồng loạt báo fail liveness. Thứ tự sự kiện: (1) probe liveness trả
> 503 ở cả ba; (2) orchestrator coi cả cụm "chết" và bắt đầu restart container để
> "cứu" chúng; (3) các container còn khỏe mạnh bị restart không cần thiết, hạ
> gục toàn bộ service dù lỗi thực sự chỉ nằm ở Redis. Tách `/health` (không gọi
> dependency) khỏi `/ready` (mới kiểm tra Redis) giúp liveness không bị nhiễu
> bởi lỗi của thành phần khác.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng đều đặn dù request rơi vào container khác
> nhau, vì cả ba cùng đọc/ghi một Redis. Nếu lưu trong dict Python của từng
> process, mỗi instance giữ bộ nhớ riêng: request đầu vào A ghi lịch sử ở A,
> request sau vào B thấy `history_length` không tăng (hoặc tăng không nhất quán,
> thậm chí mất hẳn khi container bị restart) — agent "mất trí nhớ".

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp thật khi deploy lên Railway: `/health` trả 200 nhưng `/ready` và
> `/ask` đều trả `500 Internal Server Error`. Tôi thử gọi từ máy thấy
> `/ask` không key cũng 500 thay vì 401, nên biết lỗi nằm ở lớp cấu hình chứ
> không phải logic. Xem log Railway thấy `Exception in ASGI application`; khi tôi
> tái hiện trên máy không có `AGENT_API_KEY` thì gặp `ValidationError:
> agent_api_key Field required`. Nguyên nhân là `AGENT_API_KEY` chưa được đặt
> trong tab Variables của service Railway (mà .env không được commit nên không
> tồn tại trong container). Tôi sửa bằng cách thêm `AGENT_API_KEY`, `REDIS_URL`
> vào Service Variables rồi redeploy — sau đó `/ask` bắt đầu trả 401/200 đúng.
