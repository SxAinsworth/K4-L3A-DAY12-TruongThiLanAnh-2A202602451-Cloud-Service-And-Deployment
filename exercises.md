# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trương Thị Lan Anh  Mã học viên: 2A202602451

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, lúc chưa có `AGENT_API_KEY`, `/health` vẫn trả 200
> nhưng các endpoint cần `Settings` trả 500. Fail fast giúp mình nhận ra ngay
> cấu hình production đang thiếu secret và sửa trước khi cho người dùng gọi.
> Nếu có khóa mặc định `"changeme"`, service vẫn chạy và người biết khóa mặc
> định có thể gọi `/ask`, làm tiêu quota và chi phí của mình.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log mình nhận được là:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T09:00:00+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":35,"cost_usd":0.00002145}`.
> Với JSON này, mình có thể lọc toàn bộ log theo `user_id="sv-test"` để điều
> tra request và có thể cộng/đặt cảnh báo theo `cost_usd`. Dòng
> `print("đã trả lời xong")` không có field ổn định để máy lọc hoặc tổng hợp.

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
| 1 stage (bản đầu) | khoảng 1.02 GB |
| Multi-stage | 305 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage mình build và kiểm tra bằng `docker images` có dung lượng
> 305 MB, còn bản đầu dùng image `python:3.11` đầy đủ khoảng 1.02 GB. Phần
> chênh lệch chủ yếu là hệ điều hành/base image đầy đủ, công cụ build và các
> file chỉ cần trong lúc cài dependency. Runtime dùng `python:3.11-slim` và
> chỉ copy virtualenv cùng source cần chạy nên không mang các phần đó theo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, tạo virtualenv, copy
> `requirements.txt` và `pip install` được lấy từ cache; phần copy source và
> các layer sau nó phải chạy lại. Nếu đặt `COPY . .` trước `pip install`, chỉ
> một thay đổi source cũng làm layer COPY đổi, kéo theo layer cài thư viện bị
> mất cache và phải tải/cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng thực thi lệnh trong API có thể cho kẻ tấn công chạy lệnh bên
> trong container. Nếu process chạy root, mã độc có quyền root trong container
> và có thể lợi dụng mount/socket hoặc một lỗ hổng container runtime để tác
> động máy host với quyền cao. `USER 10001` cắt chuỗi ở bước thực thi trong
> container: mã bị chiếm quyền chỉ có quyền của user thường, giảm đáng kể phạm
> vi đọc, ghi và leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây: gửi 10 request ở giây 59 của phút trước,
> rồi gửi tiếp 10 request ở giây 00 của phút sau. Bộ đếm theo phút vừa reset
> nên cả hai nhóm đều hợp lệ dù thực tế chúng nằm sát nhau. Sliding window 60
> giây vẫn nhìn thấy nhóm đầu khi nhóm sau tới nên sẽ chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng tiền của từng user trong tháng. Một user gọi ít request nhưng prompt rất
> dài có thể qua rate limit mà bị cost guard chặn vì đã hết ngân sách. Ngược
> lại, user còn nguyên ngân sách nhưng gửi 11 request liên tiếp sẽ bị rate
> limit chặn request thứ 11 dù chi phí tháng vẫn thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm probe chung của cả 3 container trả lỗi. Orchestrator
> hiểu nhầm cả ba process đã chết, lần lượt loại chúng khỏi traffic và restart.
> Container mới khởi động vẫn kiểm tra Redis đang lỗi nên tiếp tục fail, tạo
> vòng lặp restart và toàn bộ service mất khả dụng. Tách riêng thì `/health`
> vẫn 200 để không restart process khỏe, còn `/ready` trả 503 để tạm ngừng nhận
> request cho đến khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi request cùng `X-User-Id` thấy lịch sử của request
> trước nên `history_length` tăng nhất quán theo 0, 2, 4, ... dù request rơi vào
> instance nào. Nếu dùng dict Python, ba instance có ba dict riêng; load
> balancer phân phối request làm số liệu có thể nhảy như 0, 0, 2, 0, 2 thay vì
> tăng đều, và history còn mất hẳn khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Railway, `/health` trả 200 nhưng `/ready` và `/ask` ban đầu trả
> `500 Internal Server Error`. Mình gọi riêng từng endpoint để thấy health
> không phụ thuộc cấu hình, rồi kiểm tra Variables trên dashboard và phát hiện
> thiếu `AGENT_API_KEY`, còn `REDIS_URL` chưa được tạo đúng dạng reference.
> Mình thêm API key trong Railway, tạo `REDIS_URL` bằng Add Reference tới biến
> của service `day12-redis`, redeploy, rồi kiểm tra lại: `/ready` trả 200 với
> `redis:true`, thiếu key trả 401 và key hợp lệ trả 200.
