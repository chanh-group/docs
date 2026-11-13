Tuần tới chúng ta bắt tay vào fix các lỗi:
  1. Lỗi cuộn trang tin nhắn với phân trang và tanstack query, làm sao để cuộn mượt, vô trang là auto ở tin nhắn cuối (ko có hiệu ứng cuộn xuống), khi có tin nhắn mới đến thì ko đc cuộn xuống mà hiện 1 nút "có tin nhắn mới" để người dùng tự click vào, khi click vào thì cuộn xuống tin nhắn mới. 

  2. Màn story, kho lưu trữ tin, (chưa có chức năng xoá) (Hải)
  4. Chưa có avarta trong màn tìm kiếm bạn bè (An) [x] (Đã hoàn thành)
  5. Chưa có chức năng bấm vô avatar để xem tin của bạn bè (Hải)
  6. Phát triển tính năng reply tin nhắn từ đó phát triển kèm trả lời tin của bạn bè. (Hải)
  7. Optimistic UI khi gửi tin nhắn, khi gửi tin nhắn thì hiển thị tin nhắn đó ngay lập tức, nếu gửi thất bại thì hiển thị thông báo lỗi và xoá tin nhắn đó đi. (An) [x] (Đã hoàn thành)
  8. Kho lưu trữ tin đang chưa khớp với hệ thống, chưa i18n, chưa theme theo hệ thống, (có chức năng xoá cứng tin lưu trữ chưa) (Hải)
  9. Chức năng mute tin của 1 bạn bè đang 1 chiều, chưa có mở khoá trở lại =))) xem lại cái luồng nhá (Hải hoặc xoá luôn cho nhẹ việc)
  10. Chưa gộp tin của 1 bạn bè (chỉ gộp ở phía owner đã hoạt động, phía đối phương thì chưa) (Hải)

  11. Design lại màn tạo tin, sửa conflict và push mới 2 màn voice call và video call để mọi người ở core test chức năng sớm nhất có thể (Công)

Phát triển feature mới:
  -1. Thông tin cuộc trò chuyện (Chat Info Panel): (An) [x] (Đã hoàn thành toàn diện)
      - Header thông tin hội thoại: Avatar người dùng, tên hiển thị, trạng thái online, bio.
      - 3 nút thao tác nhanh (Quick Actions):
        + Trang cá nhân: Thẻ UserProfileDialog chuẩn hóa hiển thị @username kèm nút sao chép nhanh, nhãn bạn bè [✓ Bạn bè], bio, 2 nút hủy kết bạn và chặn người dùng (loại bỏ hoàn toàn ID thô UUID).
        + Tắt/bật thông báo: Tích hợp hook useConversationMute (single source of truth), loại bỏ nút trùng lặp ở ngăn dưới.
        + Tìm kiếm: Gắn thông báo phản hồi tính năng đang phát triển.
      - Bố cục Drawer co giãn: Cho phép kéo thả thay đổi độ rộng panel trên desktop (280px - 600px), lưu và khôi phục từ localStorage, tương thích tối đa Light/Dark theme.
      - Kho lưu trữ Media & File:
        + Sắp xếp đảo ngược theo thứ tự thời gian (từ mới nhất đến cũ nhất).
        + Giới hạn số lượng hiển thị khoa học (6 media, 4 file) kèm nút bấm "Xem tất cả (N)" / "Thu gọn".
        + Video thumbnail tự động sinh poster qua <video preload="metadata">, overlay nút Play và nhãn VIDEO/duration.
        + Xem video trực tiếp trên trình duyệt qua ImageLightboxModal toàn màn hình (HTML5 controls, autoPlay, tải về, phím Esc) và nút xem toàn màn hình trong tin nhắn chat.
      - Ngăn Quyền riêng tư & Quản lý: Hủy kết bạn và chặn người dùng có Modal xác nhận song ngữ, tự động phát sự kiện Socket realtime friend:removed.
      - Backend Storage: Nâng thời hạn S3/R2 presigned URL lên 24 giờ (86400s) loại bỏ triệt để lỗi ExpiredRequest.
      - Đảm bảo kiểm thử 100% passed (Backend: 772/772 tests, Frontend: 90/90 tests, 0 lint warnings).
  0. API & Giao diện Tìm kiếm tin nhắn trong cuộc trò chuyện (Message Search): (An) [ ] (Kế hoạch thực hiện)
      - Backend API (REST Endpoint):
        + Xây dựng endpoint: `GET /api/conversations/:id/messages/search?q={keyword}&limit={limit}&cursor={cursor}`.
        + Xác thực & phân quyền: Kiểm tra người gọi bắt buộc phải là thành viên hợp lệ của cuộc hội thoại (`conversationId`).
        + Bộ lọc & Thuật toán tìm kiếm: Tìm kiếm từ khóa theo chuỗi văn bản (`body`), hỗ trợ tiếng Việt không dấu/có dấu (PostgreSQL `ILIKE` / Full-Text Search với `unaccent`), loại trừ các tin nhắn đã bị thu hồi (`deletedAt IS NULL`).
        + Kết quả trả về: Danh sách tin nhắn khớp từ khóa kèm thông tin người gửi, mốc thời gian, tệp đính kèm và con trỏ phân trang `nextCursor` cho danh sách dài.
        + Tối ưu cơ sở dữ liệu: Đảm bảo index trên `(conversation_id, created_at)` và cân nhắc GIN/Trigram index trên nội dung tin nhắn để tối ưu tốc độ phản hồi < 50ms.
      - Frontend UI/UX:
        + Tích hợp trực tiếp vào nút Quick Action "Tìm kiếm" trên Header và trong Chat Info Panel.
        + Thanh tìm kiếm dạng popover/floating bar phía trên khung tin nhắn với debounce nhập liệu (300ms) và phím tắt `Esc` để đóng.
        + Danh sách kết quả tìm kiếm kèm trích đoạn tin nhắn và highlight từ khóa khớp.
        + Tính năng Scroll-to-Message: Khi nhấp vào một kết quả tìm kiếm, tự động cuộn dòng thời gian tới đúng tin nhắn đó và kích hoạt hiệu ứng nhấp nháy sáng (highlight flash animation) để người dùng nhận diện ngay ngữ cảnh tin nhắn.
        + Điều hướng kết quả: Hỗ trợ bộ đếm kết quả (ví dụ `1/12`) kèm nút mũi tên Lên/Xuống để chuyển tiếp nhanh giữa các tin nhắn tìm được.
  1. Group chat (An)
  2. Chức năng voice call direct (Hiệu)
  3. Chức năng video call direct (Hiệu)

  Tương lai:
  1. Call group
  2. Video group
  3. Streaming screen direct
  4. Streaming screen group
