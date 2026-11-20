# Kế hoạch Kỹ thuật Chi tiết: Triệt tiêu API Calls Trùng lặp & Tối ưu Tốc độ Chuyển Hội thoại (Issue #35)

- **Mã công việc:** Issue #35 (Post-PR #34 Optimization)
- **Nhánh Frontend (`lemon-ui`):** `fix/chat-switch-duplicate-calls` (tách từ commit `1c66ba9` của `main`)
- **Nhánh Backend (`lemon-backend`):** `fix/read-receipt-lock-contention` (tách từ commit `e5491c9` của `main`)
- **Thời gian lập:** 20/11/2026
- **Trạng thái:** Sẵn sàng thực thi (Ready for Execution)

---

## 1. Bối cảnh & Hiện trạng Vấn đề (Context & Issue Summary)

### 1.1. Hiện tượng ghi nhận
Sau khi merge PR #34 (Instant Chat Cache Switch), UI đã có khả năng hiển thị tin nhắn ngay từ cache. Tuy nhiên, khi người dùng thực hiện chuyển tab liên tục giữa 2 cuộc hội thoại bạn bè (warm cache), app **vẫn có độ trễ 1 – 1.5 giây** trước khi trở nên mượt mà.

### 1.2. Dữ liệu thực nghiệm qua Playwright (Recorded Metrics)
Kiểm thử tự động trên môi trường localhost với dữ liệu thật khi chuyển đổi qua lại 4 lần:

```text
Switch 1 (→ Bạn A, cold cache): 0 API calls  ✅  (prefetch hoạt động tốt)
Switch 2 (→ Bạn B):              5 API calls  ~1200ms network
  - 3× story fetch (460ms + 808ms + 461ms) — render quote story reply
  - 1× messages syncMissed (882ms)
  - 1× read (1193ms)
Switch 3 (→ Bạn A, warm cache): 7 API calls  ~3800ms network
  - 4× conversations?limit=50 🔴 (540ms + 939ms + 341ms + 679ms) — TRÙNG LẶP NGHIÊM TRỌNG
  - 1× messages syncMissed (289ms)
  - 1× read (786ms)
  - 1× members (520ms)
Switch 4 (→ Bạn B, warm cache): 7 API calls  ~2700ms network
  - 3× R2 media (voice notes, story video)
  - 1× story fetch (144ms)
  - 1× messages syncMissed (569ms)
  - 1× read (907ms)
  - 1× conversations?limit=50 (535ms)
```

### 1.3. Hệ quả
1. Chuyển tab giữa 2 bạn bè lẽ ra chỉ mất **0 – 50ms (0-1 call ngầm)** nhưng thực tế mất **7 – 10 calls, kéo dài 2.5s – 3.8s**.
2. Riêng API `conversations?limit=50` lãng phí **2.5 giây** mỗi lần switch.
3. API `POST /read` backend mất **800ms – 1200ms** do lock contention trong cơ sở dữ liệu.

---

## 2. Phân tích Nguyên nhân Gốc rễ (Root Cause Analysis)

### Nguyên nhân 1: Toàn bộ 16 Socket Listeners bị Re-bind khi Switch Conversation
- **Vị trí:** `lemon-ui/src/features/messaging/hooks/useMessagingRealtime.ts:697`
- `activeConversationId` được khai báo trong mảng phụ thuộc của `useEffect` quản lý socket listeners.
- **Hệ quả:** Mỗi khi `activeConversationId` đổi từ Bạn A sang Bạn B:
  1. Toàn bộ 16 listeners bị hủy (`socket.off`) và gắn lại (`socket.on`).
  2. Effect mới tự động chạy lại `refreshInbox()` và `verifyActiveConversation`.
  3. Bất kỳ socket event nào đến đồng thời (ví dụ `read:updated`) sẽ kích hoạt các chuỗi invalidation đè lên nhau.

### Nguyên nhân 2: `verifyConversation` chạy ngầm cưỡng bức `refetchQueries`
- **Vị trí:** `lemon-ui/src/features/messaging/hooks/useMessagingRealtime.ts:305-312, 261`
- Biến `trust` trong `verifyActiveConversation` được đọc trước khi ghi:
  ```ts
  const trust = conversationTrustMap.current.get(conversationId)
  if (!trust) {
    conversationTrustMap.current.set(conversationId, { verifiedAt: Date.now() })
  }
  setConversationAccess('allowed')
  void syncMissedMessages(conversationId).catch(() => {})
  if (!trust || Date.now() - trust.verifiedAt >= 5 * 60_000) {
    void verifyConversation(conversationId, { background: true }) // LUÔN CHẠY TRONG LẦN ĐẦU!
  }
  ```
  Vì `trust` ban đầu là `undefined`, điều kiện `!trust` luôn `true` $\rightarrow$ gọi `verifyConversation(..., { background: true })`.
- Trong `verifyConversation`:
  - Dòng 239: Gọi `getConversationById(conversationId)` (dư thừa với 1-1 chat).
  - Dòng 261: Gọi `await queryClient.refetchQueries({ queryKey: CHAT_QUERY_KEYS.conversations, type: 'active' })` $\rightarrow$ **ép buộc fetch lại toàn bộ danh sách inbox từ server**.
  - Dòng 271: Lại gọi `syncMissedMessages` lần thứ 2.

### Nguyên nhân 3: Chuỗi phản ứng Invalidation dây chuyền (Cascading Invalidation Loop)
- **Vị trí:** `useMessages.ts:694` & `useMessagingRealtime.ts:624`
- Khi mở hội thoại:
  1. `markVisibleMessageRead` bắn `POST /api/conversations/:id/read`.
  2. Khi API thành công, `useMarkReadMutation.onSuccess` gọi `invalidateQueries(CHAT_QUERY_KEYS.conversations)`.
  3. Cùng thời điểm đó, backend emit socket event `read:updated`.
  4. Frontend nhận `read:updated` $\rightarrow$ lại gọi `invalidateQueries(CHAT_QUERY_KEYS.conversations)`.
  5. Cùng lúc đó, `verifyConversation` gọi `refetchQueries(conversations)`.
  6. Vì React Query hoàn tất mỗi request trong khoảng ~300-500ms trước khi event kế tiếp tới, **React Query không dedupe được mà liên tiếp gửi 4–5 request `GET /conversations?limit=50`**.

### Nguyên nhân 4: `syncMissedMessages` bắn API mỗi cú click dù WebSocket đang connected 100%
- **Vị trí:** `useMessagingRealtime.ts:310, 336, 271`
- Khi WebSocket đang online liên tục, mọi tin nhắn mới đều được push realtime qua `socket.on('message:new')` và merge vào `liveMessages` cache (xem dòng 406).
- Việc gọi `syncMissedMessages` tại mỗi cú click chuyển tab là dư thừa network I/O (~300-900ms). Ngoài ra, hàm này không có in-flight deduplication, dẫn đến 2 luồng gọi đồng thời sẽ bắn 2 request `GET /messages?after=...` lên server.

### Nguyên nhân 5: Double Fetch do Prefetch Hover + PointerDown
- **Vị trí:** `MiddlePanel.tsx` & `MainPane.tsx`
- Rê chuột vào item kích hoạt timer debounce 100ms.
- Nhấp chuột kích hoạt `onPointerDown`. Nếu không xóa timer debounce cũ, cả hai sẽ cùng bắn 2 lượt prefetch.

### Nguyên nhân 6: Story Reply Quote & Story Media thiếu Caching dài hạn
- **Vị trí:** `MessageList.tsx:164` & `useStories.ts:226`
- Mỗi tin nhắn quote story render component `StoryReplyQuote`. Component này dùng `staleTime: 60s`. Khi switch tab qua lại, các tin nhắn quote story này bị fetch lại hàng loạt metadata và media presigned URL.

### Nguyên nhân 7 (Backend): Lock Contention `FOR UPDATE` trên bảng `conversations`
- **Vị trí:** `lemon-backend/src/domain/conversation/pg.ts:1235-1240`
- `updateLastRead` thực hiện câu lệnh:
  ```ts
  await tx
    .select({ id: conversations.id })
    .from(conversations)
    .where(eq(conversations.id, conversationId))
    .for("update")
    .limit(1);
  ```
- Khóa độc quyền row-level trên bảng `conversations` gây nghẽn giữa các member cùng đọc tin và xung đột trực tiếp với các tiến trình gửi tin (`POST /messages` cập nhật `conversations.lastMessageAt`).
- Thời gian xử lý API `POST /read` bị kéo dài lên **800ms – 1200ms**.

---

## 3. Kiến trúc Giải pháp (Target Architecture)

```
[Chuyển đổi Hội thoại (Tab Switch)]
                 │
                 ├── 1. Render tức thì từ React Query Cache (0ms latency)
                 ├── 2. Giữ nguyên Socket Event Listeners (Không unbind / re-bind)
                 │
[Tác vụ Background (Silent Background Tasks)]
                 │
                 ├── 3. syncMissedMessages:
                 │        ├── Nếu Socket đang Connected & đã sync trong <60s ──> BỎ QUA
                 │        └── Nếu đang có Request cùng convId ──> Tái sử dụng In-Flight Promise
                 │
                 ├── 4. POST /read (Backend):
                 │        └── Bỏ FOR UPDATE trên bảng `conversations`
                 │            Chỉ lock `conversationParticipants` row (<20ms response)
                 │
                 └── 5. Mark Read Success / Socket `read:updated`:
                          └── Optimistic Cache Update: set unreadCount = 0
                              (KHÔNG invalidate danh sách conversations, 0 network call)
```

---

## 4. Kế hoạch Thực thi Chi tiết Từng bước (Step-by-Step Implementation)

### Bước 1: Tách Socket Lifecycle khỏi Active Conversation (Frontend)
- **File:** `lemon-ui/src/features/messaging/hooks/useMessagingRealtime.ts`
- **Nội dung:**
  1. Tạo `activeConversationIdRef = useRef(activeConversationId)`. Luôn cập nhật ref này trong render hoặc effect nhẹ.
  2. Tách `useEffect` đăng ký socket event listeners thành effect độc lập với dependencies: `[status, userId, queryClient]`. Các handlers (`handleNewMessage`, `handleMemberRemoved`, v.v.) sẽ đọc `activeConversationIdRef.current` thay vì re-bind cả effect.
  3. Tạo effect riêng chỉ phụ thuộc `[activeConversationId]` để quản lý trạng thái `conversationAccess` và xác thực khi chuyển tab.
  4. Sửa logic Direct Chat trong `verifyActiveConversation`:
     - Nếu là chat 1-1 đã có trong cache: set trust và `conversationAccess = 'allowed'` ngay lập tức.
     - **Không gọi** `verifyConversation(conversationId, { background: true })` khi switch bình thường.
  5. Xóa bỏ hoàn toàn lệnh ép refetch ở dòng 261:
     ```ts
     // XÓA: await queryClient.refetchQueries({ queryKey: CHAT_QUERY_KEYS.conversations, type: 'active' })
     ```

### Bước 2: Dập tắt Cascading Invalidation & Optimistic Mark Read (Frontend)
- **Files:**
  - `lemon-ui/src/features/messaging/hooks/useMessagingRealtime.ts`
  - `lemon-ui/src/features/messaging/hooks/useMessages.ts`
- **Nội dung:**
  1. Trong `useMessagingRealtime.ts`, tạo cơ chế debounced invalidation cho `conversations`:
     ```ts
     const conversationsInvalidateTimer = useRef<ReturnType<typeof setTimeout> | null>(null)
     const debouncedInvalidateConversations = (delayMs = 300) => {
       if (conversationsInvalidateTimer.current) return
       conversationsInvalidateTimer.current = setTimeout(() => {
         conversationsInvalidateTimer.current = null
         void queryClient.invalidateQueries({ queryKey: CHAT_QUERY_KEYS.conversations })
       }, delayMs)
     }
     ```
     Thay thế 7 chỗ gọi trực tiếp `queryClient.invalidateQueries({ queryKey: CHAT_QUERY_KEYS.conversations })` trong các socket handlers (`message:updated`, `message:revoked`, `message:hidden`, `conversation:changed`).
  2. Trong `useMessages.ts` (`useMarkReadMutation.onSuccess`):
     - Thay vì invalidate toàn bộ conversations list, thực hiện **Optimistic Cache Update**:
       ```ts
       onSuccess: (_, { conversationId }) => {
         queryClient.setQueryData<ConversationResponse[]>(CHAT_QUERY_KEYS.conversations, (old) =>
           old?.map((c) => (c.id === conversationId ? { ...c, unreadCount: 0 } : c)),
         )
       }
       ```
     - Bỏ lệnh invalidate `CHAT_QUERY_KEYS.conversation(conversationId)` vì đọc tin không làm thay đổi thông tin hội thoại.
  3. Trong `useMessagingRealtime.ts` (`handleReadUpdated`):
     - Nếu `event.userId === userId`: cập nhật trực tiếp `unreadCount: 0` vào cache thay vì invalidate.

### Bước 3: Thông minh hóa `syncMissedMessages` & In-flight Dedupe (Frontend)
- **File:** `lemon-ui/src/features/messaging/hooks/useMessagingRealtime.ts`
- **Nội dung:**
  1. Tạo `inFlightSyncRequests = useRef(new Map<string, Promise<void>>())`. Khi `syncMissedMessages(conversationId)` được gọi, nếu đã có promise đang chạy thì trả về promise đó.
  2. Tạo `lastSyncedMap = useRef(new Map<string, number>())`.
  3. Trong `verifyActiveConversation`:
     - Kiểm tra nếu `isChatSocketReady()` đang `true` VÀ `Date.now() - (lastSyncedMap.current.get(conversationId) ?? 0) < 60_000` $\rightarrow$ **Bỏ qua gọi `syncMissedMessages`**.
     - Khi socket disconnect (`handleDisconnect`) $\rightarrow$ clear `lastSyncedMap`.

### Bước 4: Khắc phục Double Fetch Prefetch (Frontend)
- **Files:**
  - `lemon-ui/src/features/shell/components/MiddlePanel.tsx`
  - `lemon-ui/src/features/friend/components/MainPane.tsx`
  - `lemon-ui/src/features/messaging/hooks/useMessages.ts`
- **Nội dung:**
  1. Trong `MiddlePanel.tsx` và `MainPane.tsx`: khi `onPointerDown` được kích hoạt, gọi ngay `clearTimeout(prefetchTimerRef.current)` để hủy debounce của `onMouseEnter`.
  2. Trong `prefetchConversationMessages`: kiểm tra trạng thái cache, nếu tin nhắn đã tồn tại và chưa stale thì không cần fetch thêm.

### Bước 5: Caching Story Reply Quote & Story Media (Frontend)
- **Files:**
  - `lemon-ui/src/features/messaging/components/conversation/MessageList.tsx`
  - `lemon-ui/src/features/story/hooks/useStories.ts`
- **Nội dung:**
  1. Trong `MessageList.tsx`, cấu hình cho `StoryReplyQuote`:
     - `staleTime: 10 * 60_000` (10 phút)
     - `gcTime: 30 * 60_000` (30 phút)
  2. Trong `useStories.ts`, cấu hình cho `useStoryMedia`:
     - `staleTime: 4 * 60_000` (4 phút, an toàn dưới TTL 5 phút của S3 presigned URL).

### Bước 6: Loại bỏ Row Lock Contention trong `updateLastRead` (Backend)
- **File:** `lemon-backend/src/domain/conversation/pg.ts:1235-1240`
- **Nội dung:**
  1. Xóa bỏ đoạn mã khóa dòng bảng `conversations`:
     ```ts
     // XÓA:
     await tx
       .select({ id: conversations.id })
       .from(conversations)
       .where(eq(conversations.id, conversationId))
       .for("update")
       .limit(1);
     ```
  2. Giữ nguyên row lock trên bảng `conversationParticipants` (`.for("update")` ở dòng 1256), đảm bảo trạng thái đọc của người dùng được serialize chuẩn xác và tuyệt đối không gây nghẽn bảng `conversations`.

### Bước 7: Cập nhật Unit Test Mocks Backend (Backend)
- **File:** `lemon-backend/test/unit/domain/conversation/pg.spec.ts:2228, 2243`
- **Nội dung:**
  1. Cập nhật các mock `txSelect.limitMock` trong describe block `ConversationPgRepository.updateLastRead guards`: giảm 1 lần mock select tương ứng với việc đã loại bỏ select trên `conversations`.

---

## 5. Danh mục File Thay đổi (Files Affected)

| Repository | Đường dẫn File | Mục đích thay đổi |
|---|---|---|
| **lemon-ui** | `src/features/messaging/hooks/useMessagingRealtime.ts` | Tách socket listeners effect, debounced invalidation, in-flight dedupe cho syncMissed |
| **lemon-ui** | `src/features/messaging/hooks/useMessages.ts` | Optimistic update unreadCount trong markReadMutation, tối ưu prefetch |
| **lemon-ui** | `src/features/shell/components/MiddlePanel.tsx` | Hủy hover debounce timer trong onPointerDown để chống double fetch |
| **lemon-ui** | `src/features/friend/components/MainPane.tsx` | Hủy hover debounce timer trong onPointerDown tương tự |
| **lemon-ui** | `src/features/messaging/components/conversation/MessageList.tsx` | Caching story reply quote metadata (10 phút) |
| **lemon-ui** | `src/features/story/hooks/useStories.ts` | Caching story media presign (4 phút) |
| **lemon-ui** | `src/features/messaging/hooks/useChatRealtime.integration.test.tsx` | Cập nhật/bổ sung test case kiểm chứng triệt tiêu duplicate calls |
| **lemon-backend** | `src/domain/conversation/pg.ts` | Bỏ `FOR UPDATE` trên bảng `conversations` trong `updateLastRead` |
| **lemon-backend** | `test/unit/domain/conversation/pg.spec.ts` | Cập nhật transaction select mocks trong test guards |

---

## 6. Kế hoạch Kiểm thử & Tiêu chuẩn Nghiệm thu (Verification & Acceptance Criteria)

### 6.1. Tiêu chuẩn Nghiệm thu Kỹ thuật (Technical Acceptance Criteria)
1. **Network Tab khi Chuyển Hội thoại Warm Cache:**
   - Số API calls giảm từ **7–10 calls** xuống còn **0–1 call** (`POST /read` chạy ngầm).
   - Tuyệt đối **không gọi** `conversations?limit=50` khi chuyển tab qua lại.
   - Tuyệt đối **không gọi** `syncMissedMessages` lặp lại khi socket đang online ổn định.
2. **Thời gian Phản hồi:**
   - Chuyển tab render tức thì: **< 50ms**, loại bỏ hoàn toàn độ trễ 1-1.5s.
   - API `POST /read` backend: thời gian xử lý **< 30ms** (không còn lock contention 800-1200ms).
3. **Bảo tồn Nghiệp vụ & Bảo mật:**
   - Group Chat epoch mismatch (`joinGeneration` đổi) vẫn purge cache tin nhắn cũ ngay lập tức trước khi render.
   - Khi socket reconnect sau offline, toàn bộ inbox và tin nhắn bị lỡ vẫn được đồng bộ chính xác.
   - Không gây mất tin nhắn live hay thông báo typing.

### 6.2. Lệnh Kiểm thử Bắt buộc (Quality Gates)
- **Frontend (`lemon-ui`):**
  - `bun run fmt:check`: Đạt 100% format chuẩn.
  - `bun run lint`: 0 warnings, 0 errors.
  - `bun run test`: Toàn bộ 32 test suites (224 tests) vượt qua.
  - `bun run build`: `tsc -b && vite build` hoàn tất không lỗi.
- **Backend (`lemon-backend`):**
  - `bun run typecheck`: 0 errors.
  - `bun run lint`: 0 errors.
  - `bun run test`: Toàn bộ 78 unit test suites (968 tests) vượt qua.
  - `bun run test:e2e`: Toàn bộ 8 e2e suites (54 tests) vượt qua.
  - `bun run build`: `nest build` hoàn tất không lỗi.
