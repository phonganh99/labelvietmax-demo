# LBVM — LOGIC TỔNG THỂ 3 WORKFLOW (BẢN TRÌNH DUYỆT V2.3)

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

### B1. Ba module bảo đảm tin nhắn và đơn hàng — yêu cầu bắt buộc V1

**V1-01 — Message Buffer & Aggregator**

- Webhook hợp lệ phải được lưu **bền vững** trước khi xử lý AI; phản hồi webhook nhanh và xử lý bất đồng bộ bằng hàng đợi/worker. Lưu mã sự kiện, nhóm, người gửi, thời gian tin và thời gian nhận, loại tin, file, nội dung nguyên bản và trạng thái.
- Chống trùng bằng khóa duy nhất theo ID sự kiện/tin; nếu không đủ ID thì xây khóa chống trùng có phạm vi rõ. Một webhook được phát lại không tạo đơn hay phản hồi lặp.
- Gom các tin gần nhau theo nhóm/người/vấn đề. **15–20 giây từ tin cuối** chỉ là thông số thử nghiệm, không phải định nghĩa “khách đã gửi xong”. Câu “chờ em gửi file” phải đặt trạng thái chờ file; có timeout cấu hình và chuyển Sales khi chờ quá lâu.
- Tin đến muộn, đảo thứ tự, hoặc bản trích xuất mới mâu thuẫn bản cũ phải kích hoạt đánh giá lại, tăng phiên bản nháp và vô hiệu hóa tác vụ AI/outbox chưa gửi dựa trên phiên bản cũ.
- Khi có nhiều người/vấn đề đồng thời mà không thể ghép chắc chắn, tách ticket hoặc chuyển Sales; không nhập dữ liệu tùy tiện vào đơn khác.

**V1-02 — Order Completeness Validator**

- Các trường bắt buộc **theo từng loại đơn/sản phẩm**: khách/nhóm, mã đơn/mẫu cũ nếu đặt lại, tên sản phẩm, số lượng, kích thước, chất liệu, file/bản mẫu, các option in và gia công bắt buộc, thông tin deadline khi có cam kết và thông tin cần xác minh.
- Mỗi trường ghi `value`, `source_message_id`, `confidence/verification`, `last_updated`, `order_version`. Phân biệt: `KNOWN`, `MISSING`, `CONFLICTED`, `WAITING_FILE`, `NEEDS_CONFIRMATION`.
- `DRAFT/NEED_INFO/WAITING_FILE` được lưu nhưng **không được chuyển thành lệnh sản xuất**. Chỉ khi dữ liệu đầy đủ, không mâu thuẫn và đúng phiên bản mới được gửi quản lý duyệt; **người quản lý** chốt lệnh.
- File mới chỉ được tính là đã nhận khi webhook/file tồn tại và tải/lưu thành công, không phải chỉ vì khách thông báo “sẽ gửi”.
- Đơn đã duyệt khi có sửa đổi phải có phiên bản mới và người có quyền duyệt thay đổi; không tự ghi đè lịch sử hoặc âm thầm cập nhật lệnh sản xuất đang chạy.

**V1-03 — Reliability Monitor**

- Theo dõi lỗi webhook, bộ xử lý, token, API gửi OA/GMF, giới hạn gọi, lỗi file, backlog và các lần thử lại. Có `inbox_events`, `outbox_messages`, `processing_attempts`, `error_logs`.
- Gửi tin dùng outbox có khóa idempotency, trạng thái `QUEUED/SENDING/SENT/FAILED` và cơ chế thử lại có giới hạn chỉ với lỗi tạm thời. Không thông báo “đã gửi” khi API chưa xác nhận; tách gửi thành công ở API với khách thực sự đọc.
- Khi có quyền và endpoint phù hợp, đối soát tin gần đây sau mất kết nối. **Không được cam kết có thể thu hồi toàn bộ tin/file đã mất** nếu API Zalo không hỗ trợ hoặc không được cấp quyền.
- Cảnh báo Admin/Sales khi sai nhóm, gửi lỗi kéo dài, nhiều sự kiện lỗi, file chưa tải hoặc đơn đã chờ thông tin quá lâu. Có chế độ dừng tự trả lời nếu dữ liệu/nguồn không đáng tin.

**Ví dụ kiểm thử 5 tin:** (1) đặt lại mẫu cũ; (2) 50.000 chiếc; (3) kích thước 50 × 30 mm; (4) đổi decal nhựa; (5) “sẽ gửi file mới”. Hệ thống tạo **một đơn nháp phiên bản mới**, đánh dấu mẫu cũ cần đối chiếu và `WAITING_FILE`. Không coi lời hứa gửi file là file đã nhận, không trình duyệt sản xuất khi thiếu dữ liệu.

**Ca lỗi bắt buộc:** 5–10 tin liên tiếp; thiếu 1 tin; webhook trùng; đến muộn; đảo thứ tự; file đính kèm tải thất bại; đơn sửa sau duyệt; token lỗi; API từ chối; gửi thất bại; mâu thuẫn thông số; đồng thời Sales đang trả lời.

**Đo nghiệm thu:** so sánh tin được lưu với tin thực tế trong nhóm GMF thử (khi có quyền đối chiếu), độ chính xác trích xuất, số đơn/tin bị nhân đôi, lỗi phát lệnh khi thiếu dữ liệu, tỷ lệ gửi API được chấp nhận, tỷ lệ lỗi được cảnh báo. Không tự đặt tỷ lệ thành công trước khi có kiểm thử thực tế.

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
- **Mốc triển khai mục tiêu:** 8 tuần; **tuần 1–2** kiểm chứng OA/GMF + lưu bền vững webhook/chống trùng; **tuần 3–4** gom tin + kiểm tra đủ thông số + AI/duyệt; **tuần 5–6** điều phối sản xuất + Reliability Monitor/outbox; **tuần 7–8** kiểm thử tin nhiều phần/tin đến muộn/file lỗi và nghiệm thu nhóm thật trước khi mở rộng. Không cam kết chính xác 100% trước nghiệm thu.

## G. Backlog giai đoạn 2 — Tối ưu AI CSKH (không nằm trong scope 8 tuần V1)

**Quyết định bổ sung V2.2:** Bảy đề xuất tối ưu CSKH được xếp vào giai đoạn sau khi V1 đã vận hành ổn định. Đây là **backlog cần phê duyệt trước khi triển khai**, không phải cam kết xây ngay trong giai đoạn đầu.

| Mã | Hướng tối ưu sau V1 | Mục tiêu |
|---|---|---|
| CS2-01 | Ngữ cảnh hội thoại nâng cao | Tách nhiều chủ đề/ticket cùng một nhóm, liên kết đúng khách, sản phẩm và mã đơn |
| CS2-02 | Khóa AI theo từng vấn đề | Sales tiếp quản từng ticket; các câu hỏi độc lập vẫn có thể được hỗ trợ; tùy chọn tạm dừng cả nhóm |
| CS2-03 | Phân tầng rủi ro 3 cấp chi tiết | FAQ cơ bản / tư vấn kỹ thuật có điều kiện / cam kết thương mại |
| CS2-04 | Ticket và SLA cho Sales | Phân công, ghi nhận đã tiếp nhận, nhắc hạn, chuyển cấp và theo dõi đã giải quyết |
| CS2-05 | Kho kiến thức kỹ thuật hai lớp | FAQ chung + tài liệu sản phẩm, vật liệu, keo và điều kiện sử dụng; kiểm soát phiên bản |
| CS2-06 | Tối ưu chi phí, tần suất AI/tin GMF | Không gọi LLM khi không cần, lọc tin xã giao, cache FAQ, giới hạn phản hồi |
| CS2-07 | Dashboard chất lượng & bộ test | Đo độ đúng, tỷ lệ tự xử lý/chuyển Sales, sai ngữ cảnh, chi phí/ticket |

**V1 vẫn bắt buộc có các kiểm soát nền:** xác minh nhóm và quyền xem đơn, chống trùng tin, dữ liệu/kiến thức đã được duyệt, không tự cam kết thương mại, Sales takeover tối thiểu theo ticket, lưu lịch sử và cơ chế dừng AI. Không hoãn những yêu cầu an toàn này sang V2.

**Điều kiện mở V2:** V1 nghiệm thu và đã sử dụng với nhóm GMF thực tế; thu được câu hỏi mẫu, số lỗi, khối lượng cuộc trao đổi và chi phí; quản lý duyệt ưu tiên, phạm vi, KPI và ngân sách riêng. Chưa cố định thời gian hoặc giá phát triển/nâng cấp V2 do chưa có dữ liệu đo lường.

**Hệ quả ngân sách:** dự toán nền PA2 (2,34–3,09 triệu/tháng theo 33 nhóm GMF-10) không tự động bao gồm chi phí phát triển và vận hành tất cả hạng mục CS2-01…CS2-07.

## F. Danh sách quản lý cần duyệt

- [ ] **D1 — Kiến trúc:** Đồng ý 3 workflow chạy song song, cùng một bộ điều phối và database.
- [ ] **D2 — Phê duyệt đơn:** Đồng ý duyệt qua GMF nội bộ **hoặc** Web App; cả hai ghi người duyệt, mã đơn và phiên bản.
- [ ] **D3 — AI CSKH:** Đồng ý “BOTH + chủ động có chọn lọc”; FAQ tự nhiên nhưng tra cứu đơn theo cú pháp chuẩn.
- [ ] **D4 — Giới hạn AI:** Không tự báo giá chưa duyệt, hứa lịch giao, xác nhận sửa đơn, trả lời khi có Sales tiếp quản.
- [ ] **D5 — Training:** Mọi kiến thức mới cần quản lý duyệt trước khi xuất bản; không học tự động từ mọi câu Sales.
- [ ] **D6 — Dữ liệu & quyền:** Phân quyền nhóm khách, lưu nhật ký, giữ phiên bản đơn và chống lặp.
- [ ] **D7 — Chi phí & lộ trình:** Chấp thuận PA2 làm hướng nền; cập nhật lại ngân sách sau đo thực tế và xác minh chi phí GMF 2027.
- [ ] **D8 — Giai đoạn sau:** Chấp thuận xếp 7 tối ưu AI CSKH vào backlog V2 sau nghiệm thu V1; chưa cam kết thời hạn/chi phí V2.
- [ ] **D9 — Reliability V1:** Bắt buộc có Message Buffer & Aggregator, Order Completeness Validator và Reliability Monitor trong V1; kiểm thử tin nhiều phần, webhook đến muộn, lỗi file/API trước khi triển khai khách thật.

**Tình trạng duyệt:** Chưa có phê duyệt chính thức. Các ô trên dành cho người quản lý xác nhận.  
**Ranh giới phạm vi:** Đây là bản logic/phân tích, chưa phải ứng dụng đã được tích hợp hay thử nghiệm với Zalo OA của LBVM.

---
**Tài liệu đầy đủ (HTML):** [Kế hoạch AI Zalo OA — LBVM 2026](./ke-hoach-ai-zalo-oa-lbvm-2026.html)
