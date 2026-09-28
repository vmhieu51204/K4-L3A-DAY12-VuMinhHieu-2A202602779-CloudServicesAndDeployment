# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Minh Hiếu  Mã học viên: 2A202602779

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên Railway hoặc Render, nếu mình lỡ quên cấu hình biến môi trường `AGENT_API_KEY` trên dashboard mà trong code lại để mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động bình thường và báo healthy. Lúc này bất kỳ ai hay bot quét dạo trên Internet chỉ cần thử key `"changeme"` là có thể gọi endpoint `/ask` thoải mái, âm thầm tiêu hết sạch tiền API LLM của mình mà mình không hề hay biết cho đến khi nhận hóa đơn cuối tháng. Nhờ việc không đặt giá trị mặc định, app ném lỗi `ValidationError` và dừng ngay lập tức lúc khởi động. Đợt deploy thất bại ngay trên màn hình giúp mình nhận ra và bổ sung secret kịp thời trước khi hệ thống đón bất kỳ traffic nào từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ console:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:49:05.123456+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}
```

Với dòng log có cấu trúc chuẩn như thế này, các hệ thống giám sát tập trung như Datadog hay CloudWatch có thể tự động bóc tách các trường để tính toán xem user nào đang tốn nhiều chi phí nhất trong ngày dựa vào `user_id` và `cost_usd`. Ngoài ra, mình có thể dễ dàng thiết lập cảnh báo tự động khi phát hiện chi phí hoặc số lượng token vượt ngưỡng bất thường theo mốc thời gian ISO UTC, điều mà dòng chữ `print("đã trả lời xong")` hoàn toàn không làm được vì thiếu thông tin định danh và không có số liệu định lượng để máy tính xử lý tự động.

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
| 1 stage (bản đầu) | 1.15 GB |
| Multi-stage | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch gần 1 GB chủ yếu nằm ở hệ điều hành đầy đủ và các công cụ biên dịch mã. Bản 1-stage dùng image `python:3.11` tiêu chuẩn chứa trọn bộ Debian kèm các compiler như `gcc`, `make`, các header C/C++ và công cụ gỡ lỗi vốn chỉ cần khi cài package. Khi chuyển sang kỹ thuật multi-stage, mình tách riêng stage `builder` để biên dịch thư viện rồi chỉ copy thư mục kết quả `/install` sang stage `runtime` dùng `python:3.11-slim`, đồng thời dùng thêm cờ `--no-cache-dir` để bỏ qua bộ nhớ đệm pip, giúp image thành phẩm cực kỳ gọn nhẹ và deploy nhanh hơn nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, khi chỉ sửa một ký tự trong code, Docker sẽ dùng lại toàn bộ cache từ các layer trước như nạp base image, tạo thư mục làm việc, copy file `requirements.txt` và chạy lệnh `pip install`. Chỉ duy nhất layer `COPY app ./app` và các chỉ thị bên dưới nó là phải chạy lại, giúp thời gian build lại diễn ra gần như tức thì. Nếu đưa `COPY . .` lên trước `RUN pip install`, mỗi lần sửa code sẽ làm thay đổi checksum của layer copy, vô hiệu hóa toàn bộ cache phía sau và bắt Docker phải tải lại từng thư viện từ Internet, làm chậm quy trình phát triển và deploy rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu container chạy bằng root và ứng dụng bị dính một lỗ hổng thực thi mã từ xa (RCE), kẻ tấn công sẽ chiếm được shell với quyền root (UID 0) bên trong container. Từ đây, nếu hệ thống có gắn nhầm Docker socket hoặc nhân Linux của máy host chưa được vá lỗi bảo mật, kẻ tấn công có thể khai thác để thoát khỏi container (container breakout) và nghiễm nhiên có quyền root trên chính máy host, kiểm soát toàn bộ server vật lý. Lệnh `USER appuser` cắt đứt chuỗi này ngay từ trong container vì hạ đặc quyền ứng dụng xuống user thường, khiến kẻ tấn công khi xâm nhập chỉ có quyền hạn tối thiểu, không thể chỉnh sửa file hệ thống hay thực hiện các kỹ thuật leo thang đặc quyền để vượt rào sang host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request chỉ trong vòng 2 giây liên tiếp. Họ làm được điều này bằng cách gửi dồn dập 10 request vào đúng giây cuối cùng của phút thứ nhất (10:00:59), sau đó ngay khi đồng hồ nhảy sang giây đầu tiên của phút tiếp theo (10:01:00), bộ đếm theo phút bị reset về 0 và họ lập tức gửi tiếp 10 request nữa. Kết quả là trong 2 giây hệ thống phải hứng trọn 20 request mà người dùng vẫn không hề vi phạm quy định của thuật toán đếm theo phút. Cửa sổ trượt (sliding window) khắc phục triệt để lỗ hổng này vì nó luôn tính tổng request trong phạm vi 60 giây động tính ngược từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit dùng để kiểm soát tần suất gọi API trong một khoảng thời gian ngắn nhằm bảo vệ máy chủ khỏi bị quá tải hoặc spam, còn cost guard nhằm bảo vệ ngân sách tài chính bằng cách giới hạn tổng số tiền chi trả cho nhà cung cấp LLM trong cả tháng. Một tình huống rate limit cho qua nhưng cost guard chặn là khi người dùng cả ngày mới gửi một câu hỏi ngắn duy nhất (tần suất rất thấp nên rate limit duyệt ngay), nhưng tài khoản của họ đã tiêu hết 10 USD của tháng trước đó nên cost guard sẽ chặn lại với lỗi 402. Ngược lại, người dùng mới toanh chưa tiêu đồng nào trong tháng nhưng chạy script gửi liên tục 15 request chỉ trong 3 giây thì sẽ bị rate limit chặn ngay từ request thứ 11 với lỗi 429 dù chi phí chỉ tốn một phần nhỏ của xu.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Đầu tiên khi Redis bị đứt kết nối 30 giây, bộ kiểm tra liveness định kỳ gọi vào cả 3 container và đều nhận về lỗi 503 do không ping được Redis. Thấy liveness probe báo hỏng, orchestrator sẽ tưởng các container bị treo chết và lập tức kill rồi restart đồng loạt cả 3 container. Lúc này toàn bộ cụm rơi vào tình trạng downtime hoàn toàn vì không còn container nào chạy để phục vụ khách hàng. Đến khi Redis kết nối lại được, các container mới lại mất thêm thời gian khởi động môi trường và tải thư viện, biến một sự cố chập chờn ngắn của Redis thành thảm họa sập toàn bộ hệ thống. Tách riêng `/ready` giúp container không bị restart oan mà load balancer chỉ tạm thời ngừng điều phối traffic vào cho đến khi Redis online trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi lưu trên Redis, cả 3 container đều dùng chung một kho dữ liệu nên dù các request được phân phối ngẫu nhiên vào bất kỳ container nào, `history_length` vẫn luôn tăng đều đặn theo từng lượt hỏi đáp (0, 2, 4, 6...). Nếu lưu trong dict của Python, mỗi container chỉ giữ một bộ nhớ RAM riêng biệt, dẫn đến việc request lần 1 vào container A thì A lưu, nhưng request lần 2 load balancer lại đẩy sang container B đang có bộ nhớ trống nên trả về độ dài bằng 0 như người lạ. Kết quả là con số `history_length` sẽ nhảy lộn xộn ngẫu nhiên tùy theo container tiếp nhận, làm cho agent bị mất ngữ cảnh hội thoại liên tục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi deploy lên Railway, container của mình bị crash liên tục ngay lúc khởi động với thông báo `NotImplementedError: TODO (CP4): cài đặt install` xuất phát từ hàm `lifespan` trong `app/main.py`. Mình tìm ra nguyên nhân bằng cách theo dõi luồng log trực tiếp của lệnh `railway up` và nhận thấy code trên server vẫn đang là bản cũ do mình chưa thực hiện git commit các file vừa hoàn thiện ở máy local. Để khắc phục, mình đã chạy `git add .` cùng `git commit` để lưu toàn bộ các cài đặt của `Lifecycle` và `Store` vào git, sau đó chạy lại `railway up --service agent -y` để đóng gói bản build mới; sau bước này container đã khởi chạy êm đẹp, nhận đúng biến `$PORT` động và vượt qua bài test health check của Railway.
