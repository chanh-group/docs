# Tài liệu Kỹ thuật & Báo cáo Kiểm thử: Hệ thống Media & File Attachments (Lemon Chat)

> **Dự án**: Lemon Chat Web Tech  
> **Nhánh**: `feat/chat-media-attachments-preview`  
> **Phiên bản**: v1.0.0 (Role 3 - Messaging, Realtime & Storage)  
> **Tác giả**: Role 3 Engineering Team  
> **Trạng thái**: Đã kiểm thử hoàn tất (Full Test Suite Passed 100%) & Đã triển khai

---

## 1. Tổng quan Kiến trúc (System Overview)

Hệ thống truyền tải và hiển thị tệp đính kèm trên Lemon Chat cung cấp giải pháp toàn diện cho việc gửi tệp đa phương tiện (Hình ảnh, Video, Âm thanh, Tài liệu PDF, Tệp nén, Tệp mã nguồn) với dung lượng **< 25MB**, tích hợp lưu trữ đám mây Cloudflare R2, cơ chế xem trước trong trình duyệt (*In-browser Previews*) và trải nghiệm giao diện người dùng (*UX*) tối ưu theo chuẩn Messenger / Zalo.

```mermaid
flowchart TD
    subgraph Client ["Frontend (lemon-ui)"]
        UI_Input["MessageComposer (File Picker / Drag & Drop / Paste)"]
        Compressor["Client Video Compressor (nén video < 25MB)"]
        UI_Bubble["Single Unified Message Bubble"]
        UI_Pill["Floating Action Bar (Hold/Hover: Emoji, Forward, 3-Dots)"]
        Modal_Preview["Preview Modals (Lightbox / Document UTF-8 / Video)"]
    end

    subgraph Backend ["Backend API (lemon-backend)"]
        File_Ctrl["File Controller (/files/presign & /files/upload)"]
        File_Svc["File Service & Validation (<25MB, MIME, Anti-Executable)"]
        Msg_Svc["Message Service & Repository"]
        Realtime["Realtime Gateway (Socket.IO + Redis + NATS)"]
    end

    subgraph Cloud ["Cloudflare R2 Object Storage"]
        R2_Bucket["lemon-chat-storage (S3 API Compliant)"]
    end

    UI_Input --> Compressor
    Compressor -->|1. Request Presign| File_Ctrl
    File_Ctrl --> File_Svc
    File_Svc -->|Tạo Presigned URL| Client
    Client -->|2. Direct Upload (PUT)| R2_Bucket
    Client -.->|2b. Fallback Upload nếu CORS/Network lỗi| File_Ctrl
    File_Ctrl -->|Proxy Stream| R2_Bucket
    Client -->|3. Gửi tin nhắn kèm metadata tệp| Msg_Svc
    Msg_Svc --> Realtime
    Realtime -->|Broadcast tin nhắn mới| UI_Bubble
    UI_Bubble --> UI_Pill
    UI_Bubble -->|Click xem trước| Modal_Preview
```

---

## 2. Thiết kế Giao diện Người dùng (Messenger-Style UX)

Hệ thống được thiết kế theo đúng quy chuẩn giao diện tin nhắn của Messenger:

### 2.1. Khung tin nhắn đồng nhất (Single Unified Bubble)
- **Không có card lồng nhau**: Toàn bộ nội dung gồm biểu tượng tài liệu, tên tệp, dung lượng và chú thích văn bản (*caption*) cùng nằm trong một khung bo góc duy nhất.
- **Biểu tượng tài liệu gấp góc (Dog-eared Document Badge)**:
  - Thiết kế vector CSS với góc gấp 45° chân thực (`clip-path` và bóng đổ viền mỏng).
  - Tự động hiển thị đuôi tệp viết hoa (`PDF`, `DOCX`, `ZIP`, `TXT`, `JS`,...) nổi bật ở chân biểu tượng.
  - Tương thích hoàn hảo với cả chế độ sáng (Light) và tối (Dark Mode).
- **Phân tách tự nhiên**: Khi có văn bản chú thích kèm theo tệp, đường phân tách mảnh tinh tế ngăn cách phần tệp và văn bản, giữ nguyên màu nền đồng nhất của bong bóng chat.

### 2.2. Thanh 3 nút chức năng nổi khi Hover / Holding
Thanh công cụ gồm 3 nút tròn tinh gọn xuất hiện bên ngoài bong bóng chat:
- **Kích hoạt linh hoạt**:
  - **Trên Desktop**: Tự động hiển thị khi di chuột (*Hover*) vào dòng tin nhắn.
  - **Trên Mobile / Màn hình cảm ứng**: Kích hoạt khi người dùng **chạm giữ (Holding 350ms)** vào tin nhắn.
  - Khi mở menu hoặc khay biểu cảm, thanh công cụ giữ trạng thái hiển thị cố định.
- **3 nút chức năng chính**:
  1. `😊` **Thả cảm xúc (Emoji)**: Mở nhanh khay 6 cảm xúc phổ biến (`👍`, `❤️`, `😆`, `😮`, `😢`, `😡`).
  2. `↩` **Chuyển tiếp (Reply / Forward)**: Trả lời hoặc chuyển tiếp tin nhắn.
  3. `⋮` **Tùy chọn khác (3 chấm)**: Mở menu chức năng dạng dropdown menu nổi.

### 2.3. Menu chức năng 3 chấm (`...`)
- **Gỡ (Revoke)**: Chỉ hiển thị đối với tin nhắn do chính người dùng gửi (`isMe`). Khi bấm, tin nhắn được thu hồi lập tức trên UI và đồng bộ realtime qua Socket.IO tới toàn bộ thành viên trong cuộc hội thoại.
- **Tải xuống (Download)**: Tự động xuất hiện bất cứ khi nào tin nhắn có tệp đính kèm.
- **Xem trước (Preview)**: Xuất hiện đối với các tệp hỗ trợ xem trực tiếp.
- **Chuyển tiếp (Forward)**: Hỗ trợ chuyển tiếp nội dung tin nhắn.
- **Sao chép (Copy)**: Xuất hiện khi tin nhắn có chứa nội dung văn bản.
- **Ghim (Pin)**: Ghim tin nhắn trong hội thoại.
- **Báo cáo (Report)**: Xuất hiện khi tin nhắn nhận từ người khác.

### 2.4. Hành vi click thông minh vào tệp đính kèm
- **Tệp xem trước được** (*Ảnh, Video, PDF, Tệp văn bản TXT / Source Code*):
  - Click trực tiếp vào tệp: **Tự động mở trình xem trước (Lightbox hoặc Document Modal)**.
- **Tệp không hỗ trợ xem trước** (*ZIP, RAR, DOCX, XLSX, APK,...*):
  - Click trực tiếp vào tệp: **Tự động kích hoạt tải tệp về máy** kèm thông báo trạng thái tải.
- Trong menu **3 chấm (`...`)** luôn có sẵn tùy chọn **Tải xuống** độc lập cho tất cả các loại tệp.

---

## 3. Kiến trúc Lưu trữ & Upload (Hybrid Resilient Architecture)

Hệ thống kết hợp cơ chế Direct-to-Cloudflare-R2 với Fallback-to-Backend-Proxy nhằm đảm bảo $0 chi phí băng thông máy chủ khi bình thường, và khả năng tải tệp 100% không bao giờ gián đoạn khi gặp lỗi mạng/CORS:

| Đặc tính | Direct Upload (Cloudflare R2) | Fallback Upload (Backend Proxy) |
| :--- | :--- | :--- |
| **Giao thức** | HTTP `PUT` trực tiếp tới URL Presigned của R2 | HTTP `POST` multipart/form-data tới `/files/upload` |
| **Chi phí Egress** | **0$ Egress Fee** (Chính sách miễn phí egress của Cloudflare) | Tốn băng thông máy chủ backend |
| **Tốc độ & Độ trễ** | Tối đa (kết nối trực tiếp qua Cloudflare Global Anycast Edge) | Phụ thuộc vị trí máy chủ backend |
| **Khi nào kích hoạt** | Mặc định ưu tiên thực hiện cho mọi upload | Tự động kích hoạt khi `Failed to fetch` (mạng lỗi, CORS chặn) |

### Bảo mật & Ràng buộc Upload
1. **Dung lượng tối đa**: `25 * 1024 * 1024` bytes (25MB) được kiểm tra chặt chẽ ở cả Frontend và Backend Guard.
2. **Loại tệp bị cấm tuyệt đối**: Chặn hoàn toàn các file thực thi nguy hiểm: `.exe`, `.bat`, `.cmd`, `.sh`, `.msi`, `.vbs`, `.pif`, `.com`, `.scr`.
3. **MIME type an toàn**:
   - Ảnh: `image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/svg+xml`, `image/bmp`, `image/heic`,...
   - Video: `video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`, `video/avi`,...
   - Âm thanh: `audio/mpeg`, `audio/wav`, `audio/ogg`, `audio/mp4`, `audio/aac`,...
   - Tài liệu: `application/pdf`, `text/plain`, `application/json`, `application/zip`, `application/vnd.openxmlformats-*`,...
4. **UTF-8 Filename Header**: Presigned URL sử dụng cú pháp chuẩn RFC 5987:
   `Content-Disposition: attachment; filename="<ascii>"; filename*=UTF-8''<url_encoded>`
   Đảm bảo tệp có dấu tiếng Việt không bị lỗi font khi tải về.

---

## 4. Các Trình Xem Trước Đa Phương Tiện (In-Browser Previews)

### 4.1. Image Lightbox Modal
- Xem hình ảnh độ phân giải gốc với nền tối làm mờ (*backdrop blur*).
- Điều khiển phóng to/thu nhỏ (*Zoom In / Out / Reset*), xoay ảnh từng bước 90° (*Rotate*).
- Hỗ trợ thao tác cuộn chuột để zoom và phím tắt `Esc` để đóng.

### 4.2. Document Preview Modal (Hỗ trợ PDF & Văn bản UTF-8)
- **Tệp PDF**: Nhúng thẻ `<iframe src="{previewUrl}#toolbar=1" />` cho phép duyệt tài liệu với đầy đủ tính năng trang, zoom và tìm kiếm văn bản native của trình duyệt.
- **Tệp văn bản / Mã nguồn (.txt, .md, .json, .js, .ts, .log,...)**:
  - Tự động fetch nội dung qua `previewUrl` với header `inline` và giải mã UTF-8 đầy đủ (`TextDecoder("utf-8")`).
  - Hộp hiển thị dạng terminal/editor sạch đẹp, hiển thị số ký tự và thanh cuộn mượt mà.

### 4.3. Client-side Video Compressor
- Phân tích bitrate và độ phân giải video trước khi gửi.
- Nếu video vượt quá 25MB hoặc có bitrate quá cao, client tự động nén video thông qua Canvas + WebCodecs / MediaRecorder API trước khi upload để tiết kiệm thời gian truyền tải.

---

## 5. Kết quả Kiểm thử Toàn diện (Full Test Suite Verification)

Hệ thống đã trải qua kiểm thử toàn diện trên cả Backend và Frontend:

### 5.1. Backend Unit Tests (`lemon-backend`)
- **Công cụ chạy test**: Vitest (`bun run test`)
- **Kết quả**: **72/72 test files passed, 681/681 unit tests passed (100%)**
- **Chi tiết các bộ test liên quan trực tiếp**:
  - `test/unit/domain/file/controller.spec.ts`: Kiểm tra endpoint presign, upload fallback, header bảo mật.
  - `test/unit/domain/file/service.spec.ts`: Kiểm tra kiểm tra định dạng, giới hạn 25MB, chặn file thực thi.
  - `test/unit/utils/storage/s3.spec.ts`: Kiểm tra tạo presigned PUT/GET, URL inline UTF-8, upload buffer.
  - `test/unit/domain/message/service.spec.ts`: Kiểm tra tạo tin nhắn đính kèm, liên kết attachment, thu hồi tin nhắn (`revokeMessage`).
  - `test/unit/domain/message/controller.spec.ts`: Kiểm tra route thu hồi tin nhắn và xử lý lỗi.

### 5.2. Frontend Unit Tests & Build (`lemon-ui`)
- **Công cụ chạy test**: Bun Native Test Runner (`bun test`)
- **Kết quả Unit Tests**: **17/17 tests passed (100%)**
  - `src/features/chat/utils/fileTypes.test.ts` (13 tests):
    * `formatBytes`: 0 B, KB, MB, Bytes dưới 1KB.
    * `isImage`: Kiểm tra MIME type và đuôi mở rộng.
    * `isVideo`: Kiểm tra video MIME và extension.
    * `isAudio`: Kiểm tra audio MIME và extension.
    * `isPdf`: Kiểm tra PDF MIME và extension.
    * `isText`: Kiểm tra Text, JSON, XML, MD, YAML,...
    * `isPreviewable`: Tách biệt chính xác tệp xem trước được và tệp không xem trước được.
    * `getKindFromMime`: Map sang phân loại upload chuẩn backend.
  - `src/features/chat/utils/chat.mapper.test.ts` (4 tests):
    * Map message có ảnh đính kèm.
    * Map message có file đính kèm của người khác.
    * Map message đã bị thu hồi (`deletedAt` -> text fallback).
    * Gom nhóm tin nhắn theo ngày (`buildDayGroupsFromMessages`).
- **Linter**: `oxlint` quét 140 files -> **0 warnings, 0 errors**.
- **Typecheck & Production Build**: `tsc -b && vite build` -> **Thành công 100% trong 2.09s**.

---

## 6. Danh mục Mã nguồn & Commit

| Thành phần | Đường dẫn tệp | Vai trò |
| :--- | :--- | :--- |
| **Backend Schema** | `lemon-backend/src/domain/message/schema.ts` | Bổ sung cột `file_name` trong `message_attachments` |
| **Backend Storage** | `lemon-backend/src/utils/storage/s3.ts` | S3 client, Presigned PUT/GET inline & attachment |
| **Backend File API** | `lemon-backend/src/domain/file/controller.ts` | Endpoint `/files/presign` & `/files/upload` |
| **Backend Limits** | `lemon-backend/src/utils/cache/constants.ts` | Giới hạn 25MB, MIME types & blocked extensions |
| **Frontend API** | `lemon-ui/src/features/chat/api/chat.api.ts` | `presignFile`, `uploadBinaryToUrl`, `uploadFileViaBackend` |
| **Frontend Action Pill** | `lemon-ui/src/features/chat/components/conversation/AttachmentActionMenu.tsx` | 3 nút tròn nổi (Emoji, Forward, 3-dots) & Popups |
| **Frontend Bubble** | `lemon-ui/src/features/chat/components/conversation/AttachmentBubble.tsx` | Single box file row, Dog-eared Badge, Auto click |
| **Frontend Message List** | `lemon-ui/src/features/chat/components/conversation/MessageList.tsx` | Single unified bubble, Hover/Hold trigger, Revoke wiring |
| **Frontend Lightbox** | `lemon-ui/src/features/chat/components/conversation/ImageLightboxModal.tsx` | Image Zoom, Rotate, Fullscreen Modal |
| **Frontend Document Modal**| `lemon-ui/src/features/chat/components/conversation/DocumentPreviewModal.tsx` | In-browser PDF & UTF-8 Text Viewer |
| **Frontend Locales** | `lemon-ui/src/locales/vi.json` & `en.json` | Đầy đủ bản dịch tiếng Việt và tiếng Anh |

### Git Commits
- **`lemon-backend`**: `1e92224` (*feat(message): add file_name to message_attachments schema and implement presign/fallback upload APIs*)
- **`lemon-ui`**: `f03729a` (*feat(chat): implement Messenger-style single box bubble and 3-button floating action bar on hover/hold*)
- **`lemon-ui` (test & tsconfig)**: `2161db0` (*test(chat): add full unit tests for fileTypes and chat mapper with bun test integration*)
