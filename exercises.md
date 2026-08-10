# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trọng Toàn  Mã học viên: 2A202601493


1. Khi deploy lên Railway, nếu tôi quên đặt `AGENT_API_KEY`, ứng dụng sẽ báo lỗi cấu hình ngay thay vì tiếp tục chạy với khóa mặc định như `"changeme"`. Điều này giúp tôi phát hiện sai cấu hình trước khi public API bị người khác gọi bằng một khóa dễ đoán.

2. Dòng log tôi thu được:

```text
[DÁN MỘT DÒNG JSON CÓ event="ask_completed" TỪ LOG]
```

Với log JSON, tôi có thể lọc các request theo `user_id` và thống kê chi phí theo `cost_usd`. Dòng `print("đã trả lời xong")` không có trường dữ liệu rõ ràng nên khó tìm kiếm, tổng hợp và đưa vào hệ thống giám sát.

3. Kết quả thực tế:

```text
1 stage: [ĐIỀN KÍCH THƯỚC] MB
Multi-stage: [ĐIỀN KÍCH THƯỚC] MB
```

Phần dung lượng chênh lệch chủ yếu đến từ base image Python đầy đủ, pip cache, compiler và các công cụ chỉ cần trong quá trình build. Multi-stage chỉ chuyển dependency đã cài và source cần thiết sang runtime nên image nhỏ hơn.

4. Khi chỉ sửa một ký tự trong `app/main.py`, các layer base image, copy `requirements.txt` và cài dependency được lấy lại từ cache. Layer copy source và các layer phía sau phải chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm layer copy thay đổi, khiến pip phải cài lại toàn bộ thư viện.

5. Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với quyền của process trong container. Nếu container chạy bằng root, kẻ tấn công có quyền root trong container và có thể tiếp tục lợi dụng volume mount, capability hoặc lỗ hổng container runtime để tấn công host. Lệnh `USER appuser` giới hạn quyền ngay tại bước chiếm process, khiến kẻ tấn công chỉ có quyền của user thường.

6. Với fixed window 10 request/phút, người dùng có thể gửi tối đa 20 request trong 2 giây: gửi 10 request ở giây 59 của phút trước và thêm 10 request ở giây 00 của phút sau. Sliding window ngăn trường hợp này vì luôn đếm trong 60 giây gần nhất.

7. Rate limit giới hạn số request trong một khoảng thời gian ngắn, còn cost guard giới hạn tổng số tiền trong tháng. Một request rất dài và đắt có thể được rate limit cho qua nhưng bị cost guard chặn. Ngược lại, nhiều request rất rẻ gửi liên tục có thể vẫn còn ngân sách nhưng bị rate limit chặn vì vượt số request/phút.

8. Nếu gộp `/health` và `/ready` rồi kiểm tra Redis trong cùng endpoint, khi Redis mất kết nối thì cả ba container đều bị đánh dấu unhealthy. Orchestrator restart cả ba container, nhưng Redis vẫn chưa hoạt động nên chúng tiếp tục fail và rơi vào vòng lặp restart. Tách hai endpoint giúp container vẫn được xem là còn sống, trong khi load balancer chỉ tạm ngừng gửi traffic tới instance chưa ready.

9. Nếu lịch sử được lưu bằng dict Python, mỗi container có một bản lịch sử riêng. Khi load balancer phân phối request sang các container khác nhau, `history_length` có thể lúc là 0, lúc là 2 hoặc quay lại số cũ. Khi dùng Redis, mọi container nhìn thấy cùng dữ liệu nên lịch sử tăng ổn định theo các lượt hội thoại: 0, 2, 4, 6.

10. Khi deploy Railway, tôi gặp lỗi:

```text
Invalid value for '--port': '$PORT' is not a valid integer
```

Tôi tìm nguyên nhân bằng cách đọc deployment log và thấy Uvicorn nhận nguyên chuỗi `$PORT` thay vì một số cổng. Nguyên nhân là start command của Docker chạy ở exec form nên không tự nội suy biến môi trường. Tôi sửa `railway.toml` bằng cách bọc lệnh trong `/bin/sh -c`, commit, push và deploy lại. Sau đó Railway truyền đúng cổng và service khởi động thành công.