# Tài liệu Kỹ thuật: Bảng Thông tin Cuộc trò chuyện & Trình phát Đa phương tiện

> **Dự án**: Lemon Chat Web Tech  
> **Nhánh**: `feat/optimistic-chat-and-search-avatar`  
> **Mô-đun**: Chat Information Panel, User Profile Dialog & Media/Video Lightbox Player  
> **Trạng thái**: Đã kiểm thử hoàn tất (Full Test Suite Passed 100%) & Đã triển khai

---

## 1. Tổng quan Kiến trúc (Architecture Overview)

Bảng thông tin cuộc trò chuyện (**Conversation Info Panel**) cung cấp cái nhìn chi tiết về người bạn trò chuyện hoặc nhóm chat, quản lý thông báo, quyền riêng tư, và là kho lưu trữ toàn bộ phương tiện (Ảnh, Video, Tệp tin, Tài liệu) được chia sẻ trong phiên trò chuyện.

```mermaid
flowchart TD
    subgraph UI_Panel ["ConversationInfoPanel (Drawer bên phải)"]
        ResizeHandle["Thanh kéo thay đổi kích thước (280px - 600px)"]
        HeaderSection["Header (Avatar, Tên, Trạng thái Online, Bio)"]
        QuickActions["3 Nút Thao tác Nhanh (Profile, Mute, Search)"]
        AccordionMedia["Ngăn 1: File phương tiện & File (Ảnh, Video, Docs)"]
        AccordionPrivacy["Ngăn 2: Quyền riêng tư & Quản lý (Hủy bạn, Chặn)"]
    end

    subgraph Modals ["Modals & Popups (createPortal vào document.body)"]
        ProfileDialog["UserProfileDialog (Thẻ thông tin @username, Bio, Quan hệ)"]
        LightboxModal["ImageLightboxModal (Ảnh zoom/rotate hoặc Video HTML5 controls)"]
        DocPreview["DocumentPreviewModal (Xem trước PDF, Code, Văn bản UTF-8)"]
        ConfirmModals["Dialog xác nhận Hủy kết bạn / Chặn người dùng"]
    end

    subgraph Backend_Services ["Backend & Cloud Storage"]
        S3_Service["Cloudflare R2 / S3 Storage (Presign TTL = 24 giờ)"]
        Friend_Service["FriendService (Xử lý Unfriend, Block & Invalidate Cache)"]
        Realtime_Gateway["RealtimeService (Broadcast 'friend:removed' socket event)"]
    end

    ResizeHandle -.->|Lưu độ rộng| LocalStorage[("localStorage: lemon_chat_info_panel_width")]
    QuickActions -->|Mở thẻ cá nhân| ProfileDialog
    AccordionMedia -->|Click ảnh / video| LightboxModal
    AccordionMedia -->|Click tài liệu| DocPreview
    AccordionPrivacy -->|Xác nhận hủy bạn| ConfirmModals
    ConfirmModals --> Friend_Service
    Friend_Service --> Realtime_Gateway
    LightboxModal <-->|Tải stream video/ảnh (TTL 24h)| S3_Service
```

---

## 2. Các Thành phần Giao diện & Trải nghiệm Người dùng (UX Details)

### 2.1. Thanh Drawer co giãn linh hoạt (Resizable Width)
- **Tùy chỉnh chiều ngang trên Desktop**: Người dùng có thể rê chuột vào cạnh trái của bảng để kéo co giãn độ rộng từ `280px` đến `600px`. Nhấp đúp vào thanh kéo để đặt lại kích thước mặc định (`340px`).
- **Lưu trữ trạng thái**: Kích thước được tự động lưu vào `localStorage` với key `lemon_chat_info_panel_width` và giữ nguyên giữa các phiên đăng nhập.
- **Tương thích Responsive**: Trên màn hình di động/tablet nhỏ (< 768px), drawer tự động chuyển sang chế độ full-width màn hình trượt từ phải sang.

### 2.2. Thẻ Thông tin Người dùng (User Profile Dialog)
- **Chuẩn hóa thông tin**:
  - Tên đầy đủ và ảnh đại diện có viền trạng thái hoạt động (online dot).
  - Tên người dùng hiển thị dưới dạng `@username` kèm nút sao chép nhanh (copy to clipboard) và toast thông báo song ngữ.
  - Huy hiệu trạng thái quan hệ: `[ ✓ Bạn bè ]`.
  - Tiểu sử (Bio) hiển thị trang nhã; hiển thị *"Chưa có tiểu sử"* nếu người dùng để trống.
  - Loại bỏ hoàn toàn chuỗi ID UUID thô kỹ thuật.
- **Thao tác quản trị**: Trang bị 2 nút hành động trực quan có màu cảnh báo: **Hủy kết bạn** và **Chặn người dùng**, kèm modal xác nhận an toàn trước khi thực thi.

### 2.3. Quản lý Thông báo Cuộc trò chuyện (Conversation Mute)
- **Single Source of Truth**: Nút thao tác nhanh trên đỉnh điều khiển trực tiếp trạng thái chuông/tắt chuông thông qua hook `useConversationMute`.
- **Loại bỏ trùng lặp**: Loại bỏ dòng chuyển đổi trùng lặp ở ngăn *Quyền riêng tư & Quản lý* bên dưới, tránh gây rối mắt và mâu thuẫn trạng thái UI.

### 2.4. Ngăn Lưu trữ File phương tiện & Tài liệu (Media & Files Archive)
- **Thứ tự sắp xếp**: Sắp xếp đảo ngược (`.reverse()`) từ danh sách tin nhắn để hiển thị các tệp vừa gửi hoặc mới nhận **từ mới nhất đến cũ nhất**.
- **Giới hạn số lượng hiển thị khoa học**:
  - **Ảnh & Video**: Hiển thị tối đa **6 mục đầu tiên** (lưới 3 cột x 2 hàng).
  - **Tệp tin & Tài liệu**: Hiển thị tối đa **4 tệp đầu tiên**.
  - Khi vượt quá giới hạn, hệ thống hiển thị nút chuyển đổi **"Xem tất cả (N)"** / **"Thu gọn"**, giữ cho layout gọn gàng và không đẩy các thành phần bên dưới ra khỏi tầm nhìn.
- **Video Thumbnail tự động**:
  - Render thẻ `<video preload="metadata">` để trình duyệt tải frame đầu tiên làm ảnh thu nhỏ poster chân thực (thay thế khối xám camera trước đây).
  - Overlay phủ bóng mờ nhẹ với nút tròn **Play** màu trắng ở giữa (hiệu ứng scale khi hover).
  - Nhãn góc dưới hiển thị thời lượng video (ví dụ `0:15`) hoặc badge `VIDEO`.

---

## 3. Trình phát Video & Bộ xem Lightbox Toàn màn hình

### 3.1. Xem Video trong Trò chuyện bình thường
- Trong bong bóng chat ([AttachmentBubble.tsx](../lemon-ui/src/features/chat/components/conversation/AttachmentBubble.tsx)):
  - Hỗ trợ thanh điều khiển HTML5 native (phát/tạm dừng, thanh tua seekbar, âm lượng, chế độ picture-in-picture).
  - Nút **Xem toàn màn hình** (biểu tượng Fullscreen) xuất hiện ở góc trên bên phải khi rê chuột vào video bubble.

### 3.2. Lightbox Modal toàn màn hình ([ImageLightboxModal.tsx](../lemon-ui/src/features/chat/components/conversation/ImageLightboxModal.tsx))
- **Khắc phục lỗi CSS Stacking Context**: Bọc qua `createPortal(content, document.body)`. Lỗi trước đây khiến modal bị kẹp bên trong khung 340px của sidebar do thuộc tính `backdrop-blur` trên `<aside>` tạo ra containing block độc lập.
- **Xử lý linh hoạt theo định dạng**:
  - **Khi tệp là Video**: Render trình phát `<video controls autoPlay playsInline>` ở kích thước tối ưu (lên tới 85vw x 80vh), tự động ẩn các nút không phù hợp (Phóng to, Thu nhỏ, Xoay ảnh), giữ lại nút **Tải về máy** và nút **Đóng (Esc)**.
  - **Khi tệp là Ảnh**: Duy trì đầy đủ bộ công cụ xoay góc 90 độ, phóng to, thu nhỏ và kéo ảnh.

---

## 4. Xử lý Lỗi `ExpiredRequest` & Cấu hình Presign S3/R2

### 4.1. Nguyên nhân
Trước đây, thời hạn hết hạn (`S3_PRESIGN_TTL_SECS`) của liên kết tải trước trong backend mặc định là 300 giây (5 phút). Sau 5 phút trò chuyện, khi người dùng mở video hoặc ảnh, Cloudflare R2 / S3 từ chối yêu cầu với lỗi XML:
```xml
<Error>
  <Code>ExpiredRequest</Code>
  <Message>Request has expired</Message>
</Error>
```

### 4.2. Giải pháp Triển khai
1. **Backend**: Tại [mapper.ts](../lemon-backend/src/domain/message/dto/mapper.ts), thiết lập tường minh `expiresInSecs: 86400` (24 giờ) cho cả liên kết `inline` (xem trước) và `attachment` (tải xuống). Cập nhật `.env.example` với `S3_PRESIGN_TTL_SECS=86400`.
2. **Frontend**: Phát video trực tiếp bên trong ứng dụng thông qua Lightbox Modal thay vì gọi `window.open` ra URL thô.

---

## 5. Socket Realtime Event: `friend:removed`

Khi một người dùng thực hiện hủy kết bạn qua API `DELETE /api/friends/:friendId`:
- **Server phát xuống**: Tới room cá nhân của cả 2 phía: `user:{userIdA}` và `user:{userIdB}`.
```json
{
  "friendId": "0193a2b1-7c4d-7e8f-9a0b-1c2d3e4f5a6b"
}
```
- **Phía Client tiếp nhận**: Hook `FriendRequestRealtime.tsx` tự động gọi làm mới bộ nhớ đệm `invalidateQueries(['friends'])`, đồng bộ trạng thái quan hệ trên UI của cả hai phía mà không cần tải lại trang.

---

## 6. Tổng kết Kiểm thử (Test Results)

- **Backend Unit Tests**: **77/77 suites, 772/772 tests passed (100%)**
- **Frontend Unit Tests**: **13/13 files, 90/90 tests passed (100%)**
- **Linting & Code Formatting**: **0 errors, 0 warnings**
- **Production Build**: Biên dịch thành công cho cả `lemon-backend` (`nest build`) và `lemon-ui` (`tsc -b && vite build`).
