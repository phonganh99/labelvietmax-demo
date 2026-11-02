# LBVM — Web App Dashboard Demo

**Bản demo tương tác, không phải hệ thống sản xuất.** Trang dùng hoàn toàn dữ liệu giả lập và không kết nối Zalo OA/GMF, Claude API hay database khách hàng.

## Xem giao diện khách hàng Zalo Bot

- [Mở Zalo Bot Demo (Cloudflare)](https://lbvm-zalo-bot-demo.phonganh99.workers.dev/)
- [Mã nguồn Zalo Bot](./zalo-bot.html)
- Hỗ trợ 3 nhóm GMF mẫu, gom 5 tin, FAQ được duyệt, tra cứu đúng/sai nhóm, chuyển Sales và mô phỏng mất kết nối.
- Bản chat là mô phỏng giao diện để đánh giá trải nghiệm, không phải giao diện chính thức của Zalo hay kết nối OA thật.

## Các màn hình đang chạy mô phỏng

1. **Tổng quan:** KPI cập nhật theo số đơn trong demo, bảng đơn cần xử lý, biểu đồ minh họa, hoạt động.
2. **Sales Inbox:** xem nhiều nhóm chat, soạn tin, AI gợi ý câu mẫu, Sales tiếp quản, bản nháp nội bộ khi mất kết nối.
3. **Đơn hàng:** lọc, tìm kiếm, tạo đơn mới, bổ sung thông số, phê duyệt (demo), kiểm tra khách duyệt mẫu.
4. **Sản xuất:** bảng Kanban thiết kế → sản xuất → QC → giao, chuyển trạng thái có điều kiện.
5. **AI Training:** kho FAQ đã duyệt, FAQ chờ duyệt, thêm và phê duyệt nội dung.
6. **Kết nối & Nhật ký:** mô phỏng mất OA, tạm khóa gửi, tạo nhóm 5 tin mẫu và một đơn nháp.

## Lưu ý an toàn
- Toàn bộ khách/đơn trong demo là nhân vật và dữ liệu giả, mọi nút “Gửi” đều **không gửi tới Zalo thật**.
- Gợi ý “AI soạn nháp” là mẫu tĩnh, **không gọi Claude API**.
- Thao tác dùng browser localStorage trong trình duyệt đó. Nhấn “Khôi phục demo” để quay về dữ liệu ban đầu.
- Demo chứa giả lập phân quyền nhưng **không phải hệ thống xác thực/phân quyền thực**; không nhập thông tin thật, token hay bí mật.
- Không dùng trang demo để xử lý đơn thật.

## Cách dùng
- Mở `index.html` bằng Safari/Chrome hoặc link Worker công khai.
- Trên điện thoại, dùng thanh điều hướng dưới màn hình.
- Dùng nút “Mô phỏng 5 tin” để thử gom tin tạo một đơn nháp.
- Vào “Kết nối & Nhật ký” để bật lỗi OA và thử nút “Gửi demo” chuyển thành lưu nháp nội bộ.

## Triển khai thực tế sau này
Dự kiến cần Cloudflare Worker/D1/R2, Zalo OA OpenAPI (thử quyền GMF), Claude API, bảo mật/đăng nhập thật, hàng đợi/outbox, đồng bộ lỗi và bộ nghiệm thu. Các dịch vụ này chưa được nối vào demo.

Trang kế hoạch dự án riêng: https://lbvm-blueprint-2026.phonganh99.workers.dev/
