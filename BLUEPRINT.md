# LBVM AI · Blueprint 2026

Website trình bày kế hoạch triển khai hệ thống AI cho Label Vietmax theo cách dễ hiểu.

## Nội dung
1. Logic tổng quan và phạm vi hệ thống
2. Ba workflow: đơn hàng, AI CSKH, Training
3. Sales Inbox kiểu Pancake (mô phỏng)
4. Timeline 8 tuần
5. Chi phí theo từng giai đoạn, PA2 và bảng tính tương tác
6. Ba phương án A/B/C khi Zalo OA/GMF không đồng bộ được
7. Tiêu chí nghiệm thu, trách nhiệm, giải thích thuật ngữ

## Trạng thái
- **Kịch bản giả định để lập ngân sách:** bắt đầu từ số 0, chưa có tài khoản ChatGPT/Claude, Zalo OA, ZBS hoặc nhóm GMF. Đây không phải thông tin xác nhận về tài khoản thực tế của LBVM.
- Đồng bộ GMF hai chiều **chưa được kiểm chứng**.
- Đây là **website trình bày kế hoạch**, không phải hệ thống vận hành.
- Dự toán tiền là giả định từ bản kế hoạch nội bộ, cần đối soát thực tế.

## Xem
- Mở file `index.html` trong trình duyệt.
- Website được thiết kế responsive cho iPhone, có nút In/Lưu PDF.
- Không phụ thuộc thư viện CDN hay yêu cầu tài khoản đăng nhập.

## Ghi chú về GitHub
Bản này hiện được lưu tách trong nhánh `lbvm-blueprint-2026` của kho lưu trữ đang kết nối. Khi tạo repository mới độc lập, đưa `index.html` và tài liệu này vào nhánh `main` của repository đó.

## Nguồn đối chiếu
- [OA OpenAPI](https://oa.zalo.me/home/documents/guides/Khoi-tao-ung-dung-va-cap-quyen_117071366476220195)
- [GMF](https://oa.zalo.me/home/documents/vie/guides/quan-ly-nhom-gmf_1954166378348758227)
- [Giá OA hiện hành](https://zalo.solutions/oa/pricing)


## Ngân sách từ số 0 (giả định 30/10/2026)

- **Khởi động tuần 1–2** (ChatGPT Plus + Claude Pro tháng đầu, OA Tăng trưởng trả trước): **2.432.000đ** nếu chọn OA 6 tháng; **3.532.000đ** nếu chọn OA 12 tháng. OA chỉ mua sau khi xác thực và liên kết ZBS.
- **Cộng dồn 8 tuần** (2 kỳ thuê bao cá nhân, 1 gói OA, dự phòng Claude API một kỳ và hạ tầng một kỳ, 1 GMF-100 trong gói): **3,584–4,514 triệu đồng** với OA 6 tháng hoặc **4,684–5,614 triệu đồng** với OA 12 tháng.
- **Đầy đủ 33 nhóm** (OA 12 tháng + 32 GMF-10 mua thêm): **2,340–3,090 triệu/tháng**, là chi phí quy đổi theo tháng chứ không phải tiền mua ban đầu.
- Không gồm phát sinh phí quốc tế/thuế ngoài giá công bố, nhân công phát triển, mở rộng nhóm trong 8 tuần và phí tin GMF sau 31/12/2026.
- Trang HTML có bộ chọn OA 6/12 tháng, Claude Pro có/không và số nhóm để tính lại.

## Nguồn giá

- [Zalo OA/GMF, 01/06/2026](https://zalo.solutions/oa/pricing)
- [Zalo OA yêu cầu xác thực và ZBS](https://oa.zalo.me/home/documents/guides/Quan-ly-goi-zalo-oa_5417812380640953602)
- [ChatGPT Plus, giá tiêu chuẩn](https://help.openai.com/en/articles/6950777-what-is-chatgpt-plus)
- [Claude Pro và API riêng](https://support.claude.com/en/articles/8325606-what-is-the-pro-plan)
