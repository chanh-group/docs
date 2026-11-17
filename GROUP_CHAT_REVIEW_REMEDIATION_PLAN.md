# Kế hoạch xử lý review TechLead cho group chat (BE + UI)

> Trạng thái: **kế hoạch, chưa triển khai**. Nguồn yêu cầu: [`../review.md`](../review.md). Khảo sát nhánh `feat/group-chat` tại BE `031ce7a`, UI `96dbd87` ngày 17/11/2026. Đối chiếu lại commit trước khi sửa vì nhánh có thể tiếp tục thay đổi. Tài liệu này không thay thế [`GROUP_CHAT_PRE_MERGE_REVIEW_FIX_PLAN.md`](GROUP_CHAT_PRE_MERGE_REVIEW_FIX_PLAN.md) đã hoàn thành.

## 1. Mục tiêu, phạm vi và tiêu chí hoàn thành

- Xử lý **toàn bộ** M1–M5, MINOR 1–18, NIT và khoảng trống test của BE; P1 1–2, P2 3–7, P3 8–19 và khoảng trống test của UI. Những nhận xét đã được sửa hoặc cần quyết định sản phẩm vẫn được ghi rõ, không âm thầm bỏ qua.
- Không để lộ nội dung tin đã ẩn/ngoài lịch sử được phép, không hiển thị hoặc cấp sai quyền nhóm, không tạo trùng tin hay object khi retry, và bảo toàn trải nghiệm VI/EN.
- “Trước merge” gồm mọi lỗ hổng bảo mật, lỗi contract, lỗi làm sai dữ liệu/hành vi, và các bài test hồi quy tối thiểu. Các mục hiệu năng/chuẩn hóa còn lại có thể tách ticket nhưng phải có chủ sở hữu và điều kiện đóng; nếu dễ sửa an toàn thì làm trong cùng PR.
- Tài liệu là **đề xuất**; các kết luận trạng thái dựa trên code tại commit nêu trên. Không chạy test hoặc migrate trong bước lập kế hoạch này.

## 2. Bài học từ hệ thống tương tự và cách áp dụng

| Tham khảo | Điều áp dụng cho LemonChat | Giới hạn |
|---|---|---|
| [Socket.IO Rooms](https://socket.io/docs/v4/rooms/) | Room broadcast gửi cùng payload cho mọi socket trong room; dữ liệu phụ thuộc người xem như `replyTarget` và `myRole` phải được dựng riêng theo người nhận hoặc bỏ khỏi event chung. Nhiều node cần Redis adapter và test thực tế. | Room chỉ là kênh truyền, không thay thế kiểm tra membership ở DB. |
| [PostgreSQL FK constraints](https://www.postgresql.org/docs/current/ddl-constraints.html) và [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html) | `SET NULL` có thể làm mất mốc quyền; FK không tự tạo index cho cột tham chiếu; index đồng thời cần chạy ngoài transaction. | Quyết định giữ/xóa lịch sử là nghiệp vụ, không suy ra chỉ từ FK. |
| [PostgreSQL isolation](https://www.postgresql.org/docs/current/transaction-iso.html) | Ở Read Committed, hai `SELECT` trong một transaction có thể thấy hai snapshot khác nhau; gộp membership và timeline vào một statement loại bỏ cửa sổ TOCTOU giữa hai câu lệnh. | Một statement vẫn có snapshot tại thời điểm bắt đầu; nếu yêu cầu thu hồi tuyệt đối trước lúc response phát ra thì cần đồng bộ/kiểm tra lại và test race riêng. |
| [TanStack Query optimistic updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) | Mỗi lần gửi cần một định danh bất biến, trạng thái pending/confirmed/uncertain/failed rõ ràng, và cache reconciliation theo ID đó. | Không mặc định rollback khi kết quả mạng không chắc chắn; cần tra lại server theo `clientMsgId`. |
| [Stream Chat typing indicators](https://getstream.io/chat/docs/sdk/react/features/messaging/typing-indicators/) | Trạng thái typing theo user ID, throttling và expiry để không kẹt khi mất `stop`; một vị trí hiển thị nhất quán. | Chỉ tham khảo UX/protocol, không sao chép quyền hay cấu trúc API. |

## 3. Khảo sát hiện trạng và phân loại phát hiện

### 3.1. BE: năm phát hiện chính

| ID | Bằng chứng ở code hiện tại | Mức/đích |
|---|---|---|
| M1 | `message/service.ts` dựng `replyTarget` theo quyền người gửi; nhánh DIRECT gọi `emitNewMessage(conversationId, response)` chung. `conversation/pg.ts::canAccessMessage` không loại `message_hidden_for_users`. Nhóm đã fanout theo người nhận. | **P0 trước merge**: lộ nội dung đã “xóa ở phía tôi”. |
| M2 | `conversation/pg.ts::addGroupMembers` trả `myRole: MEMBER` cố định; `service.ts` trả trực tiếp, thiếu trường contract. | **P1 trước merge**. |
| M3 | `createGroup` emit `conversation:created` với `toConversationResponse(result.conversation)` mang role ADMIN của creator cho mọi người. | **P1 trước merge**. |
| M4 | Migration `0021` và schema dùng `history_boundary_message_id ON DELETE SET NULL`, trong khi `messages.sender_id` cascade khi hard-delete user. `NULL` hiện được hiểu là thấy toàn bộ lịch sử. | **P0 trước merge**: fail-open lịch sử. |
| M5 | `GET /files/group-avatars/...` có `@Public()`, không kiểm membership. **Nhận xét upload trước kiểm tra quyền đã lỗi thời**: `file/service.ts::uploadGroupAvatar` gọi `conversations.findById(groupId,userId)` trước `putObject`; vẫn còn race giữa precheck, ghi object và finalize. | **P1 quyết định chính sách GET**; cleanup object P2. |

### 3.2. BE: MINOR và NIT, không bỏ sót

| ID trong review | Hiện trạng / xử lý đề xuất | Ưu tiên |
|---|---|---|
| 1 migration locks | `0020` build index không đồng thời; `0023` đổi enum qua text rồi enum. Trước triển khai đo thời gian/khóa trên bản sao dữ liệu, xác minh chính xác Drizzle migrator/transaction của lệnh `db:migrate`, đặt `lock_timeout` và maintenance window. **Không sửa `0020–0023` nếu đã áp dụng ở bất kỳ môi trường chung nào**; môi trường chưa áp dụng phải được kiểm theo đường nâng cấp riêng. Index lớn mới build ngoài transaction. | Trước deploy |
| 2 index | Thêm `message_hidden_for_users(user_id)` và `conversation_participants(history_boundary_message_id)` sau `EXPLAIN`; nếu bảng lớn dùng quy trình concurrent riêng. | P1 |
| 3 creator nullable | `creator_id SET NULL` làm yếu unique `(creator_id,create_request_id)` sau hard-delete. Chọn khóa idempotency bền vững độc lập vòng đời user; ưu tiên tombstone/creator key bất biến, vì `clientRequestId` chỉ do client sinh và không mặc nhiên unique toàn cục. Kiểm tra dữ liệu trùng và chính sách xóa user trước đổi constraint. | P1 |
| 4 partial index | Thêm `kind = 'GROUP'` vào cả hai precheck theo request ID; kiểm tra `EXPLAIN` dùng partial index. | P1 |
| 5 re-add/boundary | Code ghi đè joinedAt, read cursor, boundary khi re-add. **Không tự động “giữ mốc sớm nhất”**: hiện quy tắc chỉ thấy từ lần gia nhập hiện tại; giữ mốc cũ có thể lộ lịch sử giữa hai lần. Giữ quy tắc hiện tại, ghi quyết định vào contract/test; xác nhận mốc thuộc cùng conversation khi ghi, xem xét composite FK nếu migration khả thi. | P1 quyết định + test |
| 6 tên FK | PostgreSQL cắt tên dài; dùng tên ngắn ổn định ở migration/schema mới nếu cần thay FK, đối chiếu `pg_constraint` trước drop. | P3 |
| 7 membership TOCTOU | `findTimeline`/`findReplyTargets` đọc membership rồi query; dời điều kiện membership + boundary vào query thực, fail closed. | **P0 trước merge** |
| 8 lock order | `reactMessage` khóa message trước participant; chuẩn hóa `conversation → participant → message`, test song song với remove/leave. | P1 |
| 9 hide after block | `hideForMe` gọi `assertDirectConversationAllowed`; đổi sang kiểm membership/canAccess phù hợp, cho phép ẩn tin cũ sau unfriend/block mà không cho gửi tin. | P2 |
| 10 error oracle | Chuẩn hóa 404 cho ID message không tồn tại/không thuộc quyền ở read/edit/revoke/react/hide, không phân biệt qua body/status/timing dễ thấy. | P1 |
| 11 query perf | Đo `EXPLAIN (ANALYZE, BUFFERS)` với inbox nhiều hội thoại, nhiều tin ẩn, paginate sâu; tối ưu tuple predicate và fallback lặp chỉ khi có kết quả định lượng. | P2 |
| 12 TTL duplicate | **Đã lỗi thời**: gateway hiện import `TYPING_TTL_SECONDS` từ `utils/cache/constants.ts`; không thấy `domain/realtime/constants.ts` thứ hai. Chỉ chạy `rg`/test bảo vệ, không tạo thay đổi vô ích. | Đã xử lý |
| 13 typing DB load | Gateway vẫn cần DB cho quyền; thêm rate limit/throttle theo socket/user/conversation, cache membership TTL ngắn nếu có invalidation ngay lúc remove/leave. Không thay bằng room-only check. | P2 |
| 14 typing lifecycle | Chống `stop` hoàn tất trước `start` bằng sequence/generation; disconnect phát stop hoặc xóa state, TTL vẫn là lưới an toàn; kiểm tra Redis key có được đọc, bỏ nếu chỉ ghi vô ích. | P2 |
| 15 avatar storage | Xóa object cũ sau khi URL mới commit; khi upload/finalize lỗi xóa object mới; purge avatar nhóm khi disband theo job có retry. Không xóa ảnh còn được tham chiếu. | P2 |
| 16 active user | `addGroupMembers` cần validate `isActive` giống create, ngay trước transaction nếu có thể kiểm lại khi ghi. | P1 |
| 17 DTO i18n | Áp dụng `vmsg(...)` cho các schema nhóm, kiểm message/code ở VI/EN; lỗi auth không phát raw Zod tiếng Anh. | P1 |
| 18 inbox metadata | `findUserConversations` tính `visibleMessageAt` cho response nhưng vẫn dùng `COALESCE(conversations.lastMessageAt,createdAt)` khi sort **và khi lọc cursor**. Dùng cùng một activity expression hiển thị theo viewer ở SELECT, ORDER BY và cursor predicate, với tie-breaker ID ổn định; xác định thứ tự khi viewer chưa thấy tin nào. Test ẩn-for-me, mốc gia nhập và phân trang không lặp/không bỏ sót. | **P1 trước merge** |
| NIT hash | `timingSafeEqual` chỉ cần nếu có yêu cầu chống side-channel; hash payload không phải secret. Kiểm độ dài/format trước nếu đổi; theo dõi ticket, không chặn merge. | P3 |
| NIT replay status | Chọn 201 ổn định cho compatibility hoặc 200 + `replayed` theo API version; cập nhật UI/test/docs cùng lúc, không đổi status tùy tiện. | P3 quyết định API |
| NIT DIRECT error | Đồng bộ 400/404 của `leaveGroup`/`disbandGroup` với các thao tác nhóm và cập nhật test contract. | P3 |
| NIT update schema | `title` bắt buộc ngay ở DTO nếu endpoint chỉ đổi tên; test empty/whitespace/VI-EN. | P2 |
| NIT revoked story | Strip `storyId` trong mapper tin thu hồi; test JSON và UI refetch. | P1 |
| NIT viewerId | `viewerId` thành bắt buộc cho `findTimeline`/`findReplyTargets`; rà hết callsite/test. | **P1 trước merge** |
| NIT read cursor | Không cho `updateLastRead` nhảy tới message ẩn; trả lỗi nhất quán và kiểm unread. | P2 |

### 3.3. UI: phát hiện chính

| ID | Bằng chứng ở code hiện tại | Ưu tiên |
|---|---|---|
| P1-1 | `ConversationView.submit(..., replyToId = replyingTo?.id)` dùng default parameter; retry truyền `undefined` từ snapshot cũ nên lấy reply đang chọn mới. | **P0 trước merge** |
| P1-2 | `useMessages` giữ bubble `uncertain`, `MessageComposer` cũng giữ failed banner; `uploadedAttachment` được đọc nhưng chưa được gán sau upload, làm retry tải lại. | **P0 trước merge** |
| P2-3 | **Đã sửa tại UI `96dbd87`**: `ConversationView` chỉ truyền typing vào `MessageList`, không còn indicator thứ hai cạnh composer. Giữ test hồi quy. | Đã xử lý |
| P2-4 | `chat.mapper.ts::mapMessageResponse` và `MessageList` vẫn có đường hiển thị reactions/isForwarded cho message thu hồi khi refetch. | **P1 trước merge** |
| P2-5 | `GroupWizard.toggle` không chặn > giới hạn, `handleCreate` return lặng lẽ. | P1 |
| P2-6 | Confirm remove/add member không khóa khi request đang chạy; double click có thể gây toast lỗi sau thành công. | P1 |
| P2-7 | `useMessages.onError` nhận ra message đã confirmed nhưng `mutateAsync` vẫn reject; composer báo gửi hỏng. | P1 |
| P3-8 | Voice note không giữ `clientMsgId` qua retry. | P1 do nguy cơ trùng |
| P3-9 | `wasUncertain || isUncertainSendError(error)` giữ trạng thái không chắc chắn cả khi retry 400/403. | P1 |
| P3-10 | Blob URL có thể bị revoke khi bubble còn dùng; `failedAttemptStore` chưa dọn khi mất quyền nhóm. | **P1 trước merge** cho cleanup quyền/cache |
| P3-11 | Upload lâu nhưng composer không có progress/label; thêm trạng thái upload rõ ràng, còn bubble vẫn optimistic. | P2 |
| P3-12, 13 | **Đã sửa tại `96dbd87`**: grammar VI/EN và đếm tên thất bại; xác minh thêm 1/2/3 người và expiry. | Đã xử lý |
| P3-14 | `updateConversationLastMessage` **có set `lastMessageSender`**, mapper sau đó suy ra `lastMessageSenderName`; đường optimistic `applyOptimisticConversationPreview` chỉ set sender ID/detail. Nhận xét review không đúng cho event có profile, nhưng vẫn cần test event thiếu `sender` và optimistic rồi sửa đúng đường gây mất tên nếu tái hiện được. | Cần kiểm chứng |
| P3-15 | `MessageList` hiển thị sender name cho tin nhận mà không gate `isGroup`; direct có thể bị thêm nhãn. | P2 |
| P3-16 | Panel đọc `conversation.members`, mapper chủ yếu có `memberCount`; dùng count đã map/locale, test direct/group. | P2 |
| P3-17 | `clientRequestId` của wizard nằm trong ref và mất khi unmount. Lưu theo chữ ký payload có scope user/session, xóa khi confirmed hoặc user đổi payload, đặt thời hạn. | P2 |
| P3-18 | `runGroupAction` refresh chỉ khi thành công; invalidate/refetch roster cả khi lỗi liên quan stale state, nhưng không che lỗi thật. | P2 |
| P3-19 | Gom validator avatar chung; bỏ fallback hardcode VI; kiểm toast group-created chỉ một lần; sửa comment cũ; bổ sung `chat.newMessageCountSingle` ở VI và kiểm key đối xứng. | P2/P3 |

Ba phát hiện UI của **review vòng 1** (mất draft, xóa draft mới khi success, reconnect thiếu revalidation) đã được sửa theo `review.md`; giữ regression tests, không mở lại kiến trúc đã ổn nếu không có bằng chứng mới.

## 4. Thiết kế triển khai theo luồng công việc

### A. Quyền đọc, lịch sử và rò dữ liệu (BE trước, UI sau)

1. **M1:** Tạo một hàm eligibility cho reply target theo **recipient** (membership + boundary + hidden + revoked) và dùng cho cả DIRECT/GROUP. Hiện bước kiểm tra `replyToId` lúc **gửi** cũng dùng `canAccessMessage` không biết hidden; phải chặn người gửi reply vào tin chính họ đã ẩn, đồng thời lọc response HTTP theo người gửi. Đối với event chung không chứa `replyTarget`; tốt nhất fanout user-scoped có payload đã redact từng người. Đảm bảo người gửi thấy target hợp lệ, người nhận đã hide nhận `replyTarget:null`; nhiều tab cùng user nhất quán. Rà `message:new`, timeline, forward và push/preview để không có kênh khác dùng payload chưa lọc. Test E2E hai user/two-node, hide rồi reply, response HTTP người gửi và realtime người nhận khác nhau.
2. **M4 + MINOR 5:** Ưu tiên sửa bằng mốc lịch sử **bất biến** lưu snapshot thứ tự `(created_at,id)` hoặc cơ chế tương đương không biến `NULL` thành không giới hạn. Mô hình phải phân biệt tường minh quyền “từ đầu hội thoại” với “chỉ sau mốc”; trạng thái thiếu snapshot của người gia nhập sau là **deny**, không suy diễn thành thấy tất cả. **Trước khi backfill**, tạm chặn hard-delete có thể làm `SET NULL` (hoặc đổi FK sang `RESTRICT` nếu chính sách retention cho phép), để không phát sinh leak mới trong cửa sổ chuyển đổi. Backfill từ message mốc **trước khi bỏ FK**. **Các row legacy có boundary `NULL` là mơ hồ**: có thể là thành viên ban đầu, gia nhập khi nhóm trống, hoặc mốc đã bị `SET NULL`; không được gán quyền “từ đầu” cho tất cả. Phân loại bằng dữ liệu/audit đáng tin cậy; row không phân loại được dùng mốc `joined_at` theo DB để fail closed, chấp nhận tạm ẩn một phần lịch sử hợp lệ và ghi nhận để xử lý thủ công. Khi backfill đang chạy, writer create/re-add phải dual-write snapshot hoặc có trigger/khóa ngắn để không sinh row chỉ có mốc cũ; triển khai rolling deploy phải giữ được tương thích hai phiên bản đến lúc switch. Tất cả timeline, unread, inbox preview/order, reply, forward, read cursor phải dùng cùng predicate. Không để `SET NULL` giữ nguyên sau switch. Thêm validation mốc thuộc cùng conversation. Thiết kế migration theo expand/backfill/verify/switch, không chỉnh migration đã áp dụng.
3. **MINOR 7, NIT viewerId/read cursor, MINOR 18:** Chuyển query timeline/reply sang join/EXISTS membership **cùng statement** và dùng boundary ngay trong statement; không có membership thì không trả dữ liệu. `viewerId` bắt buộc ở type và runtime. Đồng bộ sort inbox theo thời điểm **tin hiển thị**; read cursor phải là tin hiển thị. Test forced interleaving remove/re-add giữa bước auth và query, cộng kiểm thử ẩn/ranh giới trên DB thật.
4. **M5:** Chốt chính sách avatar. Đề xuất avatar nhóm là private: GET cần xác thực + membership. UI hiện dùng Bearer token trong Axios còn ảnh `<img>` không gửi header này; phương án khả thi là fetch có auth → Blob URL (kể cả refresh token), hoặc backend stream ảnh sau auth. Rà **mọi nơi hiển thị avatar nhóm**: inbox, header, info panel, wizard/preview trước khi đổi route. **Redirect tới presigned URL không bảo đảm thu hồi ngay**: URL đã cấp còn dùng được đến hết hạn; nếu chọn URL ký hạn ngắn phải ghi rõ cửa sổ rủi ro và cache policy. Giải phóng Blob URL khi rời nhóm/logout. Nếu sản phẩm muốn public avatar như user avatar, ghi rõ đó là capability URL trong API/PR, chỉ current file name, không suy ra membership; test URL cũ ở endpoint bị chặn, đồng thời kiểm tra giới hạn của presigned URL đã phát. Dù chọn cách nào, kiểm `findById` trước `putObject` hiện có phải được giữ và bổ sung cleanup khi finalize lỗi/race. Quyết định này cần chốt trước implementation vì ảnh hưởng BE/UI/cache.
5. **UI P2-4/P3-10:** Mapper strip toàn bộ metadata không được phép trên revoked message (`reactions`, `attachments`, `isForwarded`, `storyId`, `storyPreview` tùy contract); component gate lần nữa theo `deletedAt`. Khi mất membership/disband, xóa timeline, live messages, preview URL, failed attempts và revoke Blob URLs; không để tab cũ truy cập cache.

### B. Contract nhóm và quản lý thành viên

1. **M2:** Trong transaction add member trả dữ liệu đúng actor, hoặc sau commit query conversation theo `actorId` và map `toConversationResponse`; response phải có `myRole:ADMIN`, `unreadCount`, `lastMessageSenderId`, `memberCount` và các field của schema. Kiểm race nếu actor mất quyền sau commit: chỉ trả kết quả của thao tác thành công hoặc lỗi rõ, không rò. Test schema/consumer UI.
2. **M3:** Event `conversation:created` chung chỉ chứa `{conversationId}` rồi UI refetch theo token; hoặc fanout per-user với response cá nhân. Đề xuất ID-only để giảm bề mặt rò quyền/unread. UI hiện refetch inbox nên tương thích nhưng rà các client khác, event types và test hai tài khoản creator/member, hai node. Không gửi role/unread của A cho B.
3. **MINOR 16, UI P2-5/6, P3-18:** Validate user active và accepted friendship trước add, kiểm lại phù hợp trong transaction; giới hạn tổng 10 gồm creator ngay tại bước chọn (tối đa 9 invite), hiển thị `n/9` và toast VI/EN khi vượt. Disable và guard bằng ref/mutation state cho Confirm/Cancel/Add trong khi pending; idempotent error handling cho double click. Khi lỗi stale 403/404/409, refetch group + members và thông báo có nghĩa.
4. **UI P3-16/17/19:** Panel dùng `memberCount`, tên nhóm/placeholder qua locale. Wizard persist request ID theo `(userId, payloadSignature)` trong sessionStorage với TTL; tạo ID mới khi thay title/member IDs. Khi server đã tạo nhóm, lưu `groupId` riêng và **không gọi createGroup lại** nếu chỉ upload avatar thất bại, kể cả sau unmount. Giữ request ID cho response tạo nhóm chưa rõ kết quả; chỉ xóa trạng thái sau khi đã mở nhóm/hoàn tất luồng hoặc người dùng chủ động hủy, có kiểm tra TTL và đổi tài khoản. `File` không thể phục hồi chỉ từ sessionStorage: sau reload cho mở nhóm và chọn lại avatar, không giả định ảnh đã chọn còn tồn tại. Tránh lưu dữ liệu nhạy cảm ngoài cần thiết. Test reopen wizard sau mất response và sau avatar-upload-failed.

### C. Gửi tin, retry và optimistic UI

1. **UI P1-1:** Loại default parameter phụ thuộc state hiện tại. Gói send attempt bất biến `{clientMsgId, conversationId, body, replyToId: string|null, files, uploadedAttachments}` ngay khi bấm Send; retry đọc đúng snapshot cũ. Tin mới đọc `replyingTo` tại lúc tạo attempt. Test gửi không reply → chọn reply khác → retry không gắn reply; và gửi có reply → đổi lựa chọn → retry giữ reply gốc.
2. **UI P1-2/P2-7/P3-8/9/10/11:** Một outbox state per `(accountId,conversationId,clientMsgId)`; bubble và banner là hai cách **hiển thị cùng một attempt**, chỉ một nút retry có hiệu lực. Phân biệt `uploading`, `sending`, `uncertain`, `confirmed`, `failed`. Sau upload lưu `uploadedAttachment`/objectKey vào attempt **và retry payload**, với snapshot file được giữ qua **component unmount trong cùng tab**; retry tin dùng cùng objectKey + clientMsgId, không upload lại. Nếu upload chưa xong mới tiếp tục/khởi động lại, dọn object orphan theo chính sách storage. Sau refresh, `File`/Blob URL trong RAM mất; chỉ tự retry khi đã có objectKey bền vững, còn không yêu cầu chọn lại file và không tái dùng key cho payload khác. Voice note cũng dùng ID giữ qua retry. Khi HTTP lỗi nhưng realtime/GET đã confirmed thì coi thành công ở composer, không dựng banner; khi lỗi xác định (400/403) rollback optimistic và hiển thị lỗi có hành động phù hợp. Không dùng `wasUncertain` để ghi đè kết quả lỗi hiện tại. Dọn outbox khi logout/đổi account/mất quyền, nhưng giữ attempt của hội thoại hợp lệ khi đổi màn hình. Blob URL quản lý theo reference/owner, revoke khi cả bubble, banner, composer đều không dùng. Hiển thị progress hoặc `Đang tải tệp lên…`/`Uploading…` trong composer và bubble theo cùng attempt.
3. **BE hỗ trợ:** Xác minh dedupe `clientMsgId` với message text, attachments, voice, replay sau HTTP timeout và event đến trước response. Nếu objectKey upload xong nhưng send thất bại, TTL cleanup không được xóa object đang được message tham chiếu. Test với lỗi mạng sau server commit, upload fallback, nhiều tab.

### D. DB, realtime và UX còn lại

- **Migration/DB (MINOR 1–4, 6, 11):** Thêm migration mới cho constraint/index/boundary; đo số row, bloat, thời gian khóa và plan query trên staging dữ liệu giống production. Script index concurrent riêng ngoài Drizzle transaction; idempotent, kiểm invalid index sau thất bại. Backfill từng batch có checkpoint, verify số row + invariant trước deploy app đọc schema mới. `creator_id` null/hard-delete phải giữ uniqueness theo quyết định retention. Thử rollback app cũ khi schema mới đã deploy; migration dữ liệu không rollback mù.
- **Realtime (MINOR 8, 12–14):** Sửa lock order reaction. Typing: chỉ một TTL, throttle theo user, state sequence để stop cũ không ghi đè start mới, dọn khi disconnect, membership cache invalidation khi remove/leave; test hai node và load burst. Không biến room membership thành chứng cứ quyền.
- **Quy tắc nhỏ BE (MINOR 9/10/17; NIT):** Hide-for-me sau block; lỗi truy cập thống nhất; DTO i18n; revoked `storyId`; title schema; replay status và DIRECT error code được đóng bằng contract test. Hash timingSafeEqual chỉ làm nếu threat model cần.
- **UI còn lại (P3-14/15/16/19):** Test và sửa preview sender nếu event thiếu profile, gate tên tác giả theo group, subtitle đúng member count, validator avatar dùng chung, không hardcode VI, toast không lặp, locale key đối xứng; sửa stale comment khi đụng file.

## 5. Thứ tự triển khai và phụ thuộc

| Bước | Việc làm | Điều kiện chuyển bước |
|---|---|---|
| 0. Baseline | Ghi commit BE/UI, snapshot schema/migration đã chạy, chạy test baseline, xác nhận chính sách avatar và re-add; ghi test đang fail sẵn. | Có quyết định sản phẩm và dữ liệu mẫu. |
| 1. Schema an toàn | Migration additive cho boundary/uniqueness/index; dual-write hoặc chặn writer liên quan, backfill và verify; không đổi logic đọc trước khi dữ liệu mới sẵn sàng. | DB invariant đạt cả khi ghi đồng thời; rolling deploy/rollback được thử trên staging. |
| 2. BE bảo mật | M1, M4, MINOR 7/18, viewerId, M5 policy; E2E DB + hai node. | Không leak trong negative/race tests. |
| 3. BE contract/quality | M2/M3, active user, lock order, API errors/i18n, avatar cleanup, typing. | Contract tests và perf budget đạt. |
| 4. UI dữ liệu | Mapper revoked, group responses/events, mất quyền/cache cleanup. | VI/EN và refetch/reconnect test đạt. |
| 5. UI gửi tin/UX | Outbox thống nhất, reply snapshot, voice ID, wizard/panel/preview. | Không duplicate/retry sai, không mất draft, keyboard/multi-tab đạt. |
| 6. Tích hợp | Chạy test toàn diện, migrate DB staging mới + DB nâng cấp, smoke hai node, rà diff/changelog, mở ticket P2/P3 chưa làm. | Các gate mục 6 đạt; không còn P0/P1. |

## 6. Ma trận kiểm thử và gate merge

| Tầng | Ca bắt buộc |
|---|---|
| BE unit/contract | `toConversationResponse` sau add member, event created cho creator/member, DTO VI/EN, revoked story/reactions, active/inactive add, non-admin 403, last-admin demote/leave, hide after block, read cursor hidden, error oracle. Assert **payload từng user**, thay `expect.any(Array)` yếu. |
| BE DB integration | Postgres thật: boundary null/không null, hard-delete boundary, re-add, wrong-conversation boundary, hidden-for-me direct/group, unread/preview/order và cursor qua nhiều trang, cursor before boundary, membership revoked giữa hai bước. Backfill trong lúc add/re-add đồng thời và old/new app cùng chạy; record boundary hỏng phải fail closed. Không chỉ match chuỗi SQL. |
| BE E2E | Một node + hai node Redis: reply tới tin đã hide ở direct; reply target trước join ở group; remove member → POST 403 và không nhận event; create replay sau rename; avatar GET với/không quyền theo policy; double admin race; typing stop/start/disconnect. Tách `group-chat.e2e` thành case độc lập, reset fixture mỗi test; đặt timeout hợp lý cho `chat-1-1.e2e`. Với negative event dùng cửa sổ kiểm quan sát rõ ràng + đồng bộ barrier/ack, không coi 500ms là bằng chứng tuyệt đối. |
| UI unit/component | Deferred promise: retry giữ `replyToId:null`, retry đúng file/key/ID; server-confirmed-before-HTTP-error; 403 sau uncertain; một retry control; voice note retry; revoke trên refetch; wizard 10/11 thành viên; reopen wizard sau create success + avatar failure; double click confirm; memberCount, direct sender name, group preview; typing VI/EN 1/2/3 người, tên thiếu, expiry. |
| UI integration/E2E | Hai user/two-tab; upload chậm/fallback; refresh sau revoke; unmount/reopen wizard sau timeout; remove khỏi group lúc offline rồi reconnect; clear failed attempts/Blob URL; accessibility nút pending và thông báo lỗi. UI `package.json` hiện chỉ có Vitest, chưa có `test:e2e`; bổ sung harness browser có thể chạy lặp lại (ví dụ Playwright) **hoặc** ghi kịch bản smoke có bằng chứng, môi trường và account test rõ ràng; không báo đã chạy E2E nếu chỉ chạy unit. |
| Static/build | BE `typecheck`, `lint`, `fmt:check`, `build`, `test`, `test:e2e`; UI `lint`, `fmt:check`, `build`, `test`; đối xứng key i18n VI/EN; kiểm diff migration và generated meta. |

**Gate merge:** mọi P0/P1 đóng bằng test hồi quy; migration chạy được từ DB sạch **và** DB đã áp `0020–0023`; không có leak quyền trên direct/group; không sai role/response; không tạo trùng tin/upload khi retry; không hardcoded VI trên UI EN; không test flake/race bị che bằng timeout tùy tiện. Performance của inbox/timeline/typing không suy giảm đáng kể theo số liệu baseline, có rollback plan.

## 7. Điểm cần chốt và cách ghi nhận sau triển khai

1. **Avatar nhóm:** đề xuất private GET; nếu chọn public capability URL, ghi ngoại lệ bảo mật minh bạch trong API, PR và test. Không trộn hai cơ chế giữa client và BE.
2. **Re-add:** đề xuất chỉ xem lịch sử từ **lần vào lại**. Nếu sản phẩm muốn giữ quyền xem lịch sử từng được phép, cần mô hình nhiều membership interval; một boundary sớm nhất không diễn tả được khoảng rời nhóm.
3. **Hard-delete user/message:** xác nhận nhu cầu retention/legal và đường xóa thật trước chọn `RESTRICT` hay snapshot. Dù lựa chọn nào, mất mốc không được tự chuyển thành xem toàn bộ.
4. **Replay HTTP status:** giữ 201 để không phá client hiện có trừ khi có phiên bản contract mới; idempotency được chứng minh bằng body và key, không dựa vào status.
5. **Theo dõi:** đóng từng ID trong `review.md` bằng commit + test/đo đạc; mục “đã sửa” chỉ cần regression; mục “cần kiểm chứng” phải có test tái hiện trước khi sửa. Tách ticket P2/P3 có owner, deadline và tiêu chí hoàn thành nếu không đi cùng PR này.
