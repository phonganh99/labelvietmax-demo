# LBVM — LOGIC TỔNG THỂ 3 WORKFLOW (BẢN TRÌNH DUYỆT V2.1)

**Đơn vị:** Label Vietmax (LBVM)  
**Ngày:** 30/10/2026  
**Trạng thái tài liệu:** Đề xuất trình duyệt — *chưa phê duyệt triển khai*  
**Công nghệ dự kiến:** Zalo OA/GMF (API chính thức) · Codex phát triển · Claude API xử lý AI · Web App LBVM · Cloudflare  
**Quy mô giả định:** 20–50 đơn/ngày; chuyển đổi Zalo cá nhân sang OA/GMF trong 1–2 tháng.

> **Quyết định cốt lõi cần duyệt:** 03 workflow **chạy song song trong MỘT hệ thống**, **không chọn 1 trong 3**. Workflow 3 là vòng học/kiểm duyệt kiến thức, không được tự cập nhật quy tắc trong sản xuất. Một tin nhắn có thể mở nhiều tác vụ, nhưng chỉ có **một bộ điều phối** quyết định nội dung nào được gửi cho khách.

## A. Phân biệt 3 workflow

| Workflow | Đầu vào / trigger | Xử lý tự động | Khi cần người duyệt | Kết quả |
|---|---|---|---|---|
| **WF1 — Đơn hàng & sản xuất** | Tin đặt hàng, sửa đơn; xác nhận quản lý; chuyển trạng thái | Phân tích yêu cầu, gắn mã khách/nhóm, đối chiếu đơn, tạo nháp, tạo lệnh giao việc **sau duyệt** | Chốt thông số, duyệt sản xuất, thay đổi cam kết | Hồ sơ đơn theo phiên bản, nhiệm vụ thiết kế/sản xuất/QC và tiến độ |
| **WF2 — AI CSKH + Sales takeover** | FAQ tự nhiên được phép; lệnh tra cứu đúng cú pháp; ngoại lệ | Tìm kiến thức đã duyệt, dựng câu theo template; kiểm tra quyền & điều kiện gửi | Không đủ nguồn, sai cú pháp, báo giá/giao gấp/khiếu nại, Sales tiếp quản | Một câu trả lời hợp lệ **hoặc** ticket Sales |
| **WF3 — Training & kiểm duyệt AI** | Phản hồi Sales, câu AI lỗi, câu chưa có trong kho | Lưu câu hỏi, bản nháp AI, câu Sales gửi, nguyên nhân; đề xuất FAQ/quy tắc mới | **Quản lý phê duyệt** trước khi xuất bản kiến thức | Phiên bản kho tri thức mới; lịch sử đánh giá |

### Luồng tổng quát — 1 Zalo, 1 backend, 3 workflow

```mermaid
flowchart TD
 A[Nhóm Zalo OA/GMF] --> B[Webhook / xác thực / chống trùng / gom tin]
 B --> C[Điều phối trung tâm]
 C --> D[Phân loại nhiều ý định và kiểm tra ngữ cảnh]
 D -->|Đặt hoặc sửa đơn| E[WF1: Tạo / cập nhật đơn nháp]
 E --> F{Quản lý duyệt trên Zalo nội bộ hoặc Web App?}
 F -->|Chưa| G[Chờ duyệt / yêu cầu bổ sung]
 F -->|Đã duyệt| H[Thiết kế → Duyệt mẫu → Sản xuất → QC → Giao hàng]
 D -->|FAQ hoặc lệnh tra cứu| I[WF2: Kiểm tra 6 điều kiện]
 I -->|Đạt| J[Trả lời Zalo OA theo mẫu đã duyệt]
 I -->|Không đạt| K[Ticket Sales / HUMAN takeover]
 D -->|Ngoại lệ rủi ro cao| K
 K --> L[Sales xử lý và gửi phản hồi]
 L --> M[WF3: Đề xuất kiến thức]
 M --> N{Quản lý duyệt kiến thức?}
 N -->|Đạt kiểm thử| O[Phát hành phiên bản kiến thức mới]
 N -->|Không| P[Lưu để sửa, không sử dụng]
 O -.-> I
 H --> Q[Thay đổi trạng thái được phép thông báo]
 Q --> J
```

**Điều phối song song:** Cùng một tin có thể tạo nháp ở WF1 và FAQ draft ở WF2. Nếu tin có cam kết rủi ro cao (giá, hạn giao, khiếu nại, sửa đơn có ảnh hưởng sản xuất) hoặc đang có Sales tiếp quản, **không tự gửi câu trả lời chuyên môn**; chuyển Sales và vẫn lưu đơn nháp. Mặc định chỉ **một phản hồi tự động cuối** cho mỗi cụm tin/ý định, tránh trả lời chồng chéo.

## B. Logic kích hoạt và ưu tiên

1. **Nhận tin:** chỉ chấp nhận webhook từ ứng dụng OA/GMF hợp lệ; ghi `event_id`, `group_id`, `sender_id`, thời gian, file/tin gốc; bỏ qua tin của bot, tin trùng và sự kiện không được hỗ trợ.
2. **Gom hội thoại:** ghép các tin liên tiếp trong cửa sổ thử nghiệm 15–20 giây theo nhóm/người/luồng; khi có tin mới phải đánh giá lại trước lúc gửi.
3. **Phân loại đa ý định:** một cụm tin có thể là `ORDER_NEW`, `ORDER_CHANGE`, `FAQ`, `ORDER_STATUS`, `ORDER_SPEC`, `PRICE`, `COMPLAINT`, `OTHER` đồng thời.
4. **Khóa ưu tiên:** nếu có ticket `HUMAN`, khiếu nại hoặc yêu cầu cam kết chưa được duyệt thì không tự trả lời. Ưu tiên xác minh/giải quyết đơn đang bị sửa hơn nội dung FAQ.
5. **WF1:** tạo nháp, không phát lệnh sản xuất cho đến khi quản lý duyệt đúng `order_id + version`. Khi duyệt trên Zalo nội bộ hoặc Web App phải gọi **cùng một API phê duyệt có giao dịch/khóa chống duyệt trùng**.
6. **WF2:** FAQ được hỏi bằng lời tự nhiên nếu nằm trong danh mục đã duyệt. Các lệnh tra cứu **bắt buộc cú pháp** (đề xuất `#LBVM TIENDO DH-001`, `#LBVM THONGSO DH-001`) và kiểm tra đơn thuộc đúng nhóm. Lệnh do backend định nghĩa, không mặc định là tính năng có sẵn trong GMF.
7. **WF3:** sau khi Sales trả lời, hệ thống chỉ tạo **đề xuất kiến thức**; nội dung mới được phép sử dụng sau khi quản lý duyệt và kiểm thử.
8. **Gửi tin:** backend dựng thông điệp từ template và dữ liệu được phép; lưu outbox, kết quả gửi, thử lại khi có lỗi phù hợp; không tự khẳng định đã gửi khi API chưa xác nhận.

### Bộ kiểm tra 6 điều kiện cho WF2

- Đúng loại câu hỏi được phép tự trả lời.
- Đúng cú pháp đầu vào áp dụng cho loại câu hỏi.
- Có nguồn kiến thức/đơn hàng đã được duyệt và còn hiệu lực.
- Đúng cấu trúc mẫu trả lời (không chỉ JSON hợp lệ, còn phải kiểm chứng nội dung).
- Đúng nhóm khách, quyền xem dữ liệu và trạng thái đơn.
- Không có Sales tiếp quản, thông điệp trùng hoặc lệnh tạm ngừng.

**Không đủ bất kỳ điều kiện nào → không tự gửi nội dung chuyên môn → ticket Sales.**

## C. Ví dụ cần được nghiệm thu

**Tin khách:** “Tem nhựa có chống nước không? Anh đặt lại 50.000 cái như đơn trước, thứ Sáu nhận được chứ?”

- WF1: tạo/cập nhật **đơn nháp** 50.000 cái, đối chiếu đơn cũ; đánh dấu cần xác minh ngày giao và vật liệu.
- WF2: phân loại FAQ kỹ thuật + yêu cầu cam kết thời gian; vì có yêu cầu giao hàng chưa được phê duyệt, **không tự xác nhận** hoặc hứa giao đúng hẹn; chuyển Sales xử lý toàn bộ nội dung trả lời ra khách (trừ mẫu thông báo tiếp nhận trung tính được phê duyệt riêng).
- WF3: sau khi Sales xử lý, lưu cặp câu hỏi/câu trả lời để người được phân quyền duyệt bổ sung kiến thức nếu phù hợp.
- Chốt đơn: quản lý duyệt qua **lệnh Zalo nội bộ** (ví dụ `DUYET DH-001 V2`) **hoặc Web App**, trạng thái đồng bộ ngay.

## D. Trạng thái hệ thống và dữ liệu

| Đối tượng | Trạng thái chính |
|---|---|
| Đơn hàng | `DRAFT` → `NEED_INFO` / `PENDING_APPROVAL` → `APPROVED` → `IN_DESIGN` → `AWAITING_SAMPLE_APPROVAL` → `IN_PRODUCTION` → `QC` → `READY_TO_SHIP` → `DELIVERED`; có `CANCELLED` và cơ chế phiên bản đơn |
| Hội thoại CSKH | `AUTO` → `PENDING_SALES` → `HUMAN` → `RESOLVED` → `AUTO` |
| Kiến thức đào tạo | `DRAFT` → `REVIEW` → `APPROVED` → `PUBLISHED` hoặc `REJECTED` / `RETIRED` |

Cần ít nhất các bảng nghiệp vụ: `customers`, `zalo_groups`, `messages/events`, `orders`, `order_versions`, `order_items`, `approvals`, `production_tasks`, `conversation_tickets`, `knowledge_entries`, `knowledge_reviews`, `outbox`, `audit_logs`.

**Phân quyền:** Quản lý duyệt đơn/kiến thức; Sales xử lý khách và đơn nháp theo quyền; thiết kế/sản xuất chỉ thấy công việc được phân công; Admin quản trị quyền và cấu hình. Web App và bot Zalo dùng chung quy tắc server.

## E. Kế hoạch và chi phí

- **PA2 được đề xuất:** Codex để phát triển; Claude API cho AI; Cloudflare Workers/D1/R2; Zalo OA/GMF; Web App LBVM.
- **Đã tính hai thuê bao phát triển:** ChatGPT Plus khoảng **516.000đ/tháng** và Claude Pro khoảng **516.000đ/tháng** theo tỷ giá tham khảo 30/10/2026.
- **Dự toán nền PA2:** **2,34–3,09 triệu đồng/tháng**, theo giả định **30 nhóm khách + 3 nhóm nội bộ = 33 nhóm GMF-10**. Mức mục tiêu ~3 triệu/tháng **chưa bảo đảm bao trọn AI CSKH** vì chưa đo số câu hỏi, lượt gọi Claude và phí tin từ 2027.
- **Chuyển đổi:** 1–2 tháng từ nhóm cá nhân sang OA/GMF; nhóm cũ do Sales nhập/chuyển tiếp thủ công trong giai đoạn quá độ.
- **Mốc triển khai mục tiêu:** 8 tuần; xác minh quyền OA/GMF, webhook, nhận/gửi tin trên nhóm thử **trước** khi nhân rộng. Không cam kết chính xác 100% trước nghiệm thu.

## F. Danh sách quản lý cần duyệt

- [ ] **D1 — Kiến trúc:** Đồng ý 3 workflow chạy song song, cùng một bộ điều phối và database.
- [ ] **D2 — Phê duyệt đơn:** Đồng ý duyệt qua GMF nội bộ **hoặc** Web App; cả hai ghi người duyệt, mã đơn và phiên bản.
- [ ] **D3 — AI CSKH:** Đồng ý “BOTH + chủ động có chọn lọc”; FAQ tự nhiên nhưng tra cứu đơn theo cú pháp chuẩn.
- [ ] **D4 — Giới hạn AI:** Không tự báo giá chưa duyệt, hứa lịch giao, xác nhận sửa đơn, trả lời khi có Sales tiếp quản.
- [ ] **D5 — Training:** Mọi kiến thức mới cần quản lý duyệt trước khi xuất bản; không học tự động từ mọi câu Sales.
- [ ] **D6 — Dữ liệu & quyền:** Phân quyền nhóm khách, lưu nhật ký, giữ phiên bản đơn và chống lặp.
- [ ] **D7 — Chi phí & lộ trình:** Chấp thuận PA2 làm hướng nền; cập nhật lại ngân sách sau đo thực tế và xác minh chi phí GMF 2027.

**Tình trạng duyệt:** Chưa có phê duyệt chính thức. Các ô trên dành cho người quản lý xác nhận.  
**Ranh giới phạm vi:** Đây là bản logic/phân tích, chưa phải ứng dụng đã được tích hợp hay thử nghiệm với Zalo OA của LBVM.

---
**Tài liệu đầy đủ (HTML):** [Kế hoạch AI Zalo OA — LBVM 2026](./ke-hoach-ai-zalo-oa-lbvm-2026.html)
