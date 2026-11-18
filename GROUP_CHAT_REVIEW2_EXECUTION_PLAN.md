# Kế hoạch xử lý review2 cho group chat — BE và UI

> Trạng thái: **ĐÃ HOÀN THÀNH TOÀN DIỆN (100%)**. Ngày đối chiếu: 18/11/2026. Đã triển khai và kiểm thử thành công trên Backend (commit `a5e1dd4`) và UI (commit `b7cdb79`). Toàn bộ 78/78 suite unit test BE (936/936 tests), 8/8 suite E2E BE (53/53 tests bao gồm multinode), 26/26 suite test UI (183/183 tests) và UI production build đều pass 100%. Nguồn yêu cầu: [`../review2.md`](../review2.md).

> Bước lập kế hoạch này chỉ đọc code/tài liệu; **chưa** chạy test, migrate, benchmark hay sửa BE/UI. Các ca race và hiệu năng dưới đây là yêu cầu kiểm chứng khi triển khai, không phải kết quả đã đạt.

## 1. Mục tiêu và quyết định sản phẩm đã chốt

### Mục tiêu

1. Thành viên chỉ đọc được tin và metadata thuộc **lần tham gia hiện tại**; xóa rồi thêm lại không khôi phục lịch sử cũ, kể cả khi request, socket và cập nhật membership chạy song song hoặc sự kiện remove bị bỏ lỡ lúc offline.
2. Tin được gửi sau khi gia nhập không bị ẩn im lặng vì timestamp bằng hoặc nhỏ hơn `joinedAt` do thứ tự transaction. Đồng nhất timeline, reply target, preview, unread, cursor, read marker và realtime.
3. Ảnh nhóm chỉ được tải bởi thành viên hiện tại; chỉ Admin được đổi tên/ảnh. Upload hoặc dọn storage lỗi không làm DB trỏ vào ảnh đã xóa.
4. Mỗi lần gửi của client có một `clientMsgId` bắt buộc và một vị trí thử lại duy nhất **trên dòng tin**. Không báo gửi thất bại nếu server đã xác nhận; không mất text, reply, tệp hoặc bản nháp mới.
5. Bản nháp tạo nhóm trước khi bấm Tạo được khôi phục trong phiên. Tin thu hồi và thông báo VI/EN không để lại thông tin/thành phần UI thừa.

### Các quyết định không cần hỏi lại

| Chủ đề | Quyết định |
|---|---|
| Quyền nhóm | `ADMIN` đổi title/avatar, quản trị role/member, giải tán; `MEMBER` xem danh sách và rời nhóm, **không** đổi title/avatar. Người tạo là Admin đầu tiên; không tái lập role Owner. |
| Xem avatar | Chỉ tài khoản **đang là thành viên** được tải avatar nhóm. Có URL cũ không cấp quyền; không dùng redirect tới presigned GET URL có thể chia sẻ. |
| Wizard | Khôi phục tên và thành viên đã chọn sau điều hướng/reload trong cùng phiên. Hoàn tất tạo hoặc bấm Hủy rõ ràng thì xóa nháp. File ảnh chưa upload không lưu vào `sessionStorage`; sau reload cần chọn lại ảnh. |
| Gửi tin | `clientMsgId` bắt buộc với mọi client gọi endpoint gửi tin. Tin nội bộ được tạo qua service/DB có thể giữ cột nullable. UI retry trên dòng tin, không có banner thử lại thứ hai. |
| Re-add | Thành viên xem lịch sử từ lần tham gia hiện tại; không ghép các khoảng tham gia cũ. |

## 2. Phân tích sâu lần hai: điều gì thực sự còn lỗi

### 2.1. Ranh giới lịch sử: review đánh dấu MINOR 7 quá sớm

- `message/pg.ts::findTimeline` đọc `historyBoundaryMessageId`/`joinedAt` ở một `SELECT`, sau đó gắn các giá trị này vào truy vấn danh sách tin. `JOIN conversation_participants` ở truy vấn sau chỉ kiểm tra thành viên đang active, **không dùng mốc hiện tại**. Nếu A đọc mốc cũ, bị remove rồi re-add trước truy vấn thứ hai, A có thể nhận nội dung giữa hai lần tham gia. `findReplyTargets` có cùng dạng. Đây là **P1 bảo mật trước merge**, không phải mục đã đóng.
- `canAccessMessage`, `getMessageRecipientUserIds`, `resolveWriteMiss` phần lớn đã gắn membership vào chính truy vấn. Vẫn phải rà mọi đường dùng mốc và mọi nhánh `historyBoundaryMessageId IS NULL`, không chỉ hai hàm trên.
- **Phát hiện thêm khi reconnect:** `useMessagingRealtime::verifyConversation` tải lại chi tiết nhóm và roster nhưng không biết đó là **lần tham gia mới** nếu user bị remove rồi re-add trong lúc offline. Sự kiện remove có thể đã mất; UI vẫn giữ timeline/live/archive/reply cache cũ vì server hiện chỉ trả `myRole`, không có định danh lần tham gia. Cần `membershipVersion`/`joinGeneration` tăng atomically khi re-add, trả trong DTO inbox và detail. UI so sánh với version đã cache; khác version hoặc version cũ không rõ thì dọn toàn bộ dữ liệu nhóm trước khi hiển thị/sync missed. Trường hợp không còn membership vẫn dùng luồng revoke hiện tại. Đây cũng là **P1 bảo mật trước merge**, độc lập với SQL race.
- Postgres mặc định `Read Committed`: hai `SELECT` có thể nhìn thấy hai snapshot khác nhau; thêm transaction bao ngoài hai câu đọc **không đủ**. Một truy vấn SQL dùng membership và boundary hiện tại cùng lúc là hướng phù hợp. [PostgreSQL 16: Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html).
- Giới hạn cần ghi rõ: quyền được xét tại snapshot của truy vấn. Nếu yêu cầu thu hồi cả một response đã bắt đầu trước thao tác remove, phải giữ khóa xuyên suốt response/storage, không phù hợp request path hiện tại. Test bảo mật phải chứng minh không trộn **mốc cũ + membership mới**, không tuyên bố bảo đảm ngoài phạm vi snapshot.

### 2.2. Mốc rỗng: chỉ đổi `>` thành `>=` là sai

- Khi boundary ID rỗng, code hiện dùng `messages.createdAt > joinedAt` ở timeline, reply, inbox, unread, read marker. Tin có timestamp bằng `joinedAt` bị loại. Nghiêm trọng hơn, `joinedAt` khi re-add dùng `clock_timestamp()` còn tin dùng `defaultNow()` tức `now()` của **đầu transaction**. Transaction gửi bắt đầu trước nhưng chờ khóa conversation có thể chèn tin sau re-add với timestamp nhỏ hơn mốc gia nhập. Tin vẫn ở DB nhưng người mới vào không thấy.
- `>=` sẽ có nguy cơ mở lịch sử cũ có cùng timestamp. Dùng UUIDv7 làm chứng cứ thứ tự commit cũng không đủ vì ID được tạo ở ứng dụng, còn quyền được tuần tự hóa tại DB.
- So sánh phương án:

  | Phương án | Đánh giá |
  |---|---|
  | Đổi `>` thành `>=` hoặc nới vài microsecond | Bác bỏ: có thể lộ tin trước join. |
  | Tạo message hệ thống làm sentinel cho nhóm rỗng | Có thứ tự rõ nhưng phát sinh record/timeline/cleanup mới cho một mốc không có tin. Không cần ở cấu trúc hiện tại. |
  | Thêm số thứ tự tuần tự cho toàn bộ message/membership | Mạnh nhất về mô hình dài hạn, nhưng cần backfill và thay cursor, unread, preview trên toàn bộ hệ thống. Chi phí và rủi ro migration lớn cho đợt trước merge. |
  | **Chọn:** phân biệt “không có tin trước lúc vào” với “boundary cũ phải dùng timestamp”, đồng thời đảm bảo `createdAt` của tin mới tăng đơn điệu theo khóa conversation | Giải quyết dữ liệu mới và giữ fail closed cho bản ghi cũ chưa thể chứng minh; migration additive nhỏ. Cần audit dữ liệu legacy và kiểm thử DB thật. |

- Thêm cờ boolean có tên rõ nghĩa, ví dụ `history_boundary_was_empty`, mặc định `false`, và `join_generation` bigint mặc định 1. `boundary ID != null` nghĩa là chỉ thấy `(createdAt,id)` lớn hơn tin mốc. `boundary ID = null && was_empty = true` nghĩa là **đã chứng minh hội thoại chưa có tin trước lần vào này**, nên mọi tin hiện còn thuộc hội thoại đều mới hơn. `boundary ID = null && was_empty = false` là legacy/không chắc chắn, tiếp tục điều kiện timestamp fail closed cho tới khi kiểm chứng dữ liệu. Không gán `true` chỉ vì `lastMessageId` null; khi tạo DIRECT mới cũng ghi `true` cho hai participant vì hội thoại chưa có tin. `join_generation` tăng `+1` chỉ khi re-add một participant đã rời nhóm, không tăng khi sửa role/metadata hoặc retry tạo nhóm; serialize dạng string trong DTO để tránh giới hạn số nguyên JS.
- Trong transaction tạo nhóm hoặc add/re-add, khóa conversation trước, đọc tin mới nhất thật theo index `(conversationId,createdAt DESC,id DESC)`. Nếu không có tin thì ghi cờ `true`; nếu có thì ghi ID mốc và cờ `false`, cập nhật atomically cả `joinedAt`, read state và `join_generation`. Không dùng riêng `group.lastMessageId` vì FK `SET NULL` hoặc dữ liệu lịch sử có thể khiến trường này không đại diện cho max tin thật.
- Khi insert tin mới, sau khi lấy khóa conversation, gán `createdAt` bằng thời điểm DB ít nhất **1 microsecond lớn hơn** `MAX(messages.createdAt)` của conversation, hoặc `clock_timestamp()` nếu muộn hơn; tính trong SQL từ dữ liệu DB, không chuyển qua `Date` JS vì mất độ chính xác microsecond. Luồng tạo message tập trung tại `message/pg.ts::createMessageWithConversationUpdate`, nên có một điểm writer để kiểm soát. Test burst/chi phí index trước khi chốt triển khai. Đây là ràng buộc thứ tự cho **tin mới**; dữ liệu cũ được giữ nguyên và audit riêng.
- Bản ghi cũ có null boundary được backfill cờ `true` **chỉ khi chứng minh không có message trước hoặc bằng `joinedAt`** ở conversation; còn lại giữ `false` và thống kê. Nếu có bản ghi mơ hồ gây false denial, không tự nới quyền: cần truy vết hoặc chấp nhận legacy fail closed có ghi nhận. Không thể tái tạo chính xác thứ tự commit lịch sử chỉ từ hai timestamp bằng nhau.
- FK boundary `ON DELETE RESTRICT` ở migration `0024` vẫn là lưới an toàn. Không sửa migration đã áp dụng; mọi thay đổi schema phải tạo migration Drizzle mới. Kiểm tra ràng buộc boundary thuộc đúng conversation khi ghi.

### 2.3. Realtime: phân biệt lỗi thật với suy luận sai

- Nhận xét trong `review2.md` rằng member mới không nhận typing vì chưa vào `convRoom` **không đúng tại HEAD này**: `chat.gateway.ts::handleTypingStart/Stop` gửi tới `userRoom` của tất cả thành viên hiện tại. UI không emit `conv:join` nhưng đó không phải nguyên nhân mất typing. Không mở task “sửa một dòng join room để có typing”. Giữ test regression khi add thành viên lúc socket đang mở.
- `replyTarget` đã được redaction theo recipient, nhưng recipient list và quyền xem target được đọc ở hai câu riêng trước emit. Với remove/re-add xen giữa, payload target dựng từ epoch cũ vẫn có thể được đẩy tới `userRoom`. Phương án fail closed: **không đẩy nội dung `replyTarget` trong realtime**; chỉ gửi `replyToId` và dữ liệu tin mới. UI dùng target đã có trong cache được xác thực của **cùng `joinGeneration`**; nếu chưa có, gọi endpoint đọc reply target theo `(conversationId,messageId)` dùng cùng predicate DB quyền/mốc/hidden, 403/404 trả “không khả dụng”, tránh để quote hợp lệ biến mất vô thời hạn. Giữ response trực tiếp cho sender khi sender còn quyền. Kiểm thử thời điểm remove/re-add giữa việc dựng và phát event; không dựa vào membership của socket room làm chứng cứ quyền.
- Việc một event của tin được commit khi user còn là thành viên đến sau sự kiện remove là vấn đề thứ tự phân phối khác. Contract phải định nghĩa quyền tại thời điểm commit/query và UI phải xóa cache khi nhận remove; kiểm tra thứ tự event ở hai node. Không hứa đồng bộ tuyệt đối giữa DB và mạng chỉ bằng `socketsLeave`.

### 2.4. Avatar nhóm: ba lỗi cùng một luồng

- GET hiện `@Public()` rồi redirect tới presigned URL. Presigned URL là bearer token cho bất kỳ ai giữ URL đến lúc hết hạn, nên kiểm membership **trước redirect** vẫn không đáp ứng chính sách “chỉ thành viên hiện tại xem” sau khi remove. Cần GET có auth + active membership **và URL ảnh đang dùng trong cùng một câu đọc**, kiểm filename, rồi trả bytes ảnh qua BE (hoặc cơ chế tương đương kiểm quyền cho từng request, không phát presigned URL). Ảnh tối đa 5 MB; mở rộng `StorageService`, S3 và adapter/test stub để lấy object có kiểm size/MIME, `Content-Type` an toàn và `Cache-Control: private, no-store`; đo bộ nhớ/throughput nếu chọn buffer thay vì stream. [AWS S3: Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html).
- Quyền sửa đang là `requireGroupMember` ở service/repository và `Boolean(myRole)` ở UI. Đổi thành Admin-only ở **transaction cuối cùng** của repository nữa, không chỉ precheck hoặc ẩn nút. Member vẫn xem ảnh và danh sách nhóm. Admin mất quyền giữa precheck/upload/finalize phải bị từ chối ở finalize.
- Controller hiện đặt `updateGroupAvatar` **và** xóa avatar cũ trong cùng `try`. Nếu DB đã trỏ ảnh mới nhưng xóa ảnh cũ ném lỗi, `catch` xóa luôn ảnh mới và tạo ảnh hỏng. `deleteObjects` còn có thể trả `{failed:[...]}` mà helper hiện bỏ qua. Hai Admin upload đồng thời cũng cần nhận chính xác URL **bị thay thế trong transaction**, không dựa vào `current.avatarUrl` đọc trước upload.
- Thiết kế: ghi object mới → transaction khóa nhóm, xác nhận Admin active, atomically thay URL, **ghi job dọn URL cũ trong cùng transaction**, trả `replacedAvatarUrl` + DTO → worker dọn sau commit. Chỉ dọn object mới nếu **chắc chắn** chưa commit thay URL; nếu kết quả DB không chắc chắn thì đọc lại URL trước khi xóa. Lỗi dọn ảnh cũ không được biến request đã commit thành lỗi “upload thất bại”. Giải tán nhóm cũng enqueue avatar hiện tại trong transaction soft-delete.
- Dọn ảnh cũ, ảnh mới orphan sau thất bại, và ảnh khi giải tán nhóm cần retry bền vững. Ưu tiên bảng cleanup/outbox nhỏ do migration tạo, object key unique + trạng thái/attempt, worker purge hiện có xử lý batch; chỉ xóa job khi storage xác nhận key nằm trong `deleted`, ghi log/metric cho `failed`. DB và object store không có transaction chung: nếu DB mất hoàn toàn ngay sau upload vẫn có orphan khó tránh; ghi nhận, đo và có tác vụ đối soát thay vì nuốt lỗi. Không tự xóa object vẫn được `conversations.avatarUrl` tham chiếu.
- UI đang đặt `avatarUrl` thẳng vào `<img>` qua `UserAvatar`/`Avatar`; browser không gắn access token trong Axios interceptor. Cần loader avatar nhóm chuyên biệt gọi GET bằng `api` với `responseType:'blob'`, tạo object URL, phân biệt avatar user public và avatar nhóm private. Rà inbox, header, info panel, modal forward, danh sách nhóm/call. Chia sẻ một cache theo `(accountId,groupId,joinGeneration,avatarUrl)`, hủy request/revoke object URL khi avatar đổi, user logout, rời/bị remove/re-add/giải tán; fallback initials nếu 403/404 hoặc tải lỗi. Không log token hay blob URL. Quyền Admin trên panel phải được làm mới sau reconnect/role-change trước khi bật thao tác sửa.

### 2.5. Gửi tin và cache UI

- `useMessages.onError` kiểm confirmed trong cả live + timeline, còn `ChatShell.handleSend` kiểm live từ render snapshot. Trạng thái “tin server đã nhận nhưng UI báo lỗi” vẫn có cửa sổ. Chuyển quyết định `confirmed|uncertain|failed` về **một nơi** trong mutation/outbox; `ChatShell` không tự suy đoán. Nếu xác nhận đến trước HTTP error, giữ server row và không toast lỗi.
- Composer hiện giữ `failedAttemptStore` module-level, timeline giữ temp row. Hai nơi có retry; ở lỗi xác định `onError` xóa temp row nên nếu bỏ banner mà không đổi state thì mất nội dung/tệp thất bại. Outbox/attempt theo `(accountId,conversationId,clientMsgId)` phải giữ snapshot text, `replyToId`, pending files, object keys upload xong, preview URL và trạng thái; hiển thị retry/loại bỏ **chỉ trên dòng tin**. Retry giữ đúng ID và file đã upload, không upload lại; kiểm confirmed trước khi retry. UI không cho chỉnh nội dung attempt cùng ID; muốn sửa tạo attempt/ID mới. Chuẩn hóa `SendMessageArg` về object mang ID bắt buộc; nhánh `string` hiện tự tạo ID ở `onMutate` và `mutationFn` riêng, dù chưa có caller, không được trở thành đường gửi thiếu tương quan.
- Cần phân biệt `uploading`, `sending`, `uncertain`, `failed`, `confirmed`; `400/403` xác định không nên bấm retry mù, nhưng dòng tin phải cho xem lỗi và chọn sửa/xóa. Đừng biến lỗi mạng sau server commit thành rollback. Giữ bản nháp đang gõ mới trong composer khi attempt cũ lỗi. Voice note đi qua cùng trạng thái. Dọn outbox, object URL và query cache khi mất quyền/logout/đổi account; đổi hội thoại hợp lệ thì giữ attempt trong phiên. Sau reload File/Blob trong RAM không còn: chỉ phục hồi phần có object key bền vững, còn file chưa upload yêu cầu chọn lại. Thêm query key quote theo `(conversationId,joinGeneration,messageId)` để không tái sử dụng target của lần tham gia cũ.
- BE `clientMsgId` hiện optional trong DTO nhưng UI type đã required. Bắt buộc UUID ở `POST /conversations/:id/messages`; giữ `(conversationId,senderId,clientMsgId)` idempotency hiện có. Hợp đồng client: một ID chỉ dành cho một **request bất biến**; đổi body/reply/file phải tạo ID mới. BE hiện trả message cũ cả khi payload mới dùng trùng ID; nếu muốn server trả `CONFLICT` cần lưu hash của **request gốc** (không so nội dung đã biến đổi khi forward), migration và xử lý legacy. Đó là hardening riêng, không giả vờ rằng unique index hiện kiểm được payload. Cột DB có thể nullable cho tin hệ thống/story nội bộ; không cần sửa schema chỉ để bắt buộc ở HTTP. Cập nhật contract và test VI/EN cho missing/malformed ID. Đây là thay đổi breaking với client cũ: triển khai UI sẵn gửi ID trước khi BE siết DTO, thông báo API client khác; không âm thầm chấp nhận thiếu ID.

### 2.6. Các mục UI khác

- Tin thu hồi: BE mapper còn `storyId`, `isForwarded`, `replyToId`; UI mapper giữ `replyToId/replyTarget`, `MessageList` render quote trước placeholder. Trả về DTO tối thiểu cho tin thu hồi (giữ ID, sender, thời gian, `deletedAt`, các trường schema bắt buộc; metadata nhạy cảm/null), UI có guard `!deletedAt` khi render quote, story, reactions, forwarded, và scrub cả cache realtime.
- Wizard chỉ ghi `sessionStorage` khi bấm Tạo. Lưu `{userId, step, title, memberIds, savedAt}` sau mỗi thay đổi phù hợp, debounce nếu cần; tách `draft` trước submit khỏi `createAttempt` với `clientRequestId` bất biến theo signature. Lưu theo account, TTL hiện 10 phút hoặc giá trị cấu hình; explicit cancel/success xóa, điều hướng thông thường giữ. Avatar File không nằm trong sessionStorage.
- `runGroupAction` chỉ refresh sau thành công. Nếu server commit nhưng HTTP timeout, quyền và roster cũ sẽ giữ nguyên. Sau cả success/error, revalidate chi tiết nhóm/member/inbox; khi 403/404 thì đóng panel và dọn cache như luồng mất quyền. Không hiện toast “thất bại” chắc chắn khi kết quả chỉ uncertain nếu refresh chứng minh đã áp dụng.
- `chat.mapper.ts` còn fallback cứng `'Nhóm trò chuyện'`/`'Người dùng Lemon'`; chuyển sang localized fallback ở tầng có `t` hoặc truyền translator vào mapper, test cả VI/EN. Không để namespace/core mapper phụ thuộc global language mutable một cách khó test.

## 3. Danh mục review2 và phân loại thực thi

### BE

| Mục | Đánh giá lần hai | Xử lý |
|---|---|---|
| M1 | Fanout per-recipient đã xử lý leak thông thường; còn cửa sổ thay membership giữa các query và emit. | P1: redacted realtime target + test epoch. |
| M2, M3 | Response sau add và event create ID-only đã đúng shape; giữ regression. | Không viết lại. |
| M4, MINOR 7 | FK fail closed nhưng timeline/reply target còn mốc snapshot cũ; fallback thời gian còn lỗi. | **P1 trước merge**, mục 2.1–2.2. |
| Reconnect remove/re-add | Review2 chưa nêu: UI không phân biệt membership cũ và mới nếu missed remove event. | **P1 trước merge**: `join_generation` trong DTO, clear cache theo epoch. |
| M5, MINOR 15 | Precheck trước lưu có; GET public, role sai, cleanup sau commit lỗi, disband chưa dọn. | **P1/P2 trước merge**, mục 2.4. |
| MINOR 1 | Migration/index không `CONCURRENTLY` có nguy cơ khóa bảng lớn; không sửa migration đã áp dụng. | Predeploy: đo lock, dữ liệu, chạy migration mới đúng cửa sổ; ticket vận hành cho migration cũ. |
| MINOR 2, 4, 9, 16, 17 | Index, partial lookup, hide after block, active invited user, localized DTO đã có. | Regression và kiểm e2e. |
| MINOR 3, 5, 6 | `creatorId` nullable làm yếu unique sau hard-delete; re-add ghi đè boundary/read state là **đúng policy** nhưng cần atomically set cờ mới; tên FK dài. | Kiểm invariant và ticket hard-delete; chỉ đổi FK khi migration mới cần. Không “giữ lịch sử lần cũ”. |
| MINOR 8, 10 | Reaction khóa message trước participant; pre-read còn phân biệt lỗi. | Test deadlock/error oracle trước; chuẩn hóa lock order/error nếu tái hiện hoặc có đường lộ rõ. Không suy diễn deadlock chỉ từ thứ tự khác nhau. |
| MINOR 11, 18, P2-2 | Visible helper lặp; inbox ordering đã viewer-aware nhưng cursor hoạt động biến động giữa hai query. | Đo EXPLAIN; P2 follow-up cursor `{activityAt,id}` bất biến và UI dedupe, ghi rõ không thể có snapshot feed đầy đủ nếu không có version/snapshot. |
| MINOR 12–14 | Hằng TTL trùng thật; typing mỗi event tra DB, Redis write fire-and-forget, chưa serialize start/stop. Claim “new member không nhận typing” là sai. | Dọn hằng/đo query; thêm throttle/lifecycle khi test tải chứng minh cần. Giữ quyền DB, không tin room. |
| NIT | Hash compare, HTTP replay status, DIRECT error codes, revoked `storyId`. | Redact metadata ngay đợt này; hash/status/error code theo contract và threat model riêng, không đổi HTTP ngoài yêu cầu. |
| P2-1, P2-2, P3-1, P3-2 mới của review | P2-1 đúng; P2-2 race feed có thật nhưng không bảo đảm snapshot đầy đủ chỉ bằng sửa cursor; P3-1 latent vì chưa có hard-delete production; P3-2 đúng, kèm bug xóa avatar mới nghiêm trọng hơn. | Mục 2.2, 2.4; test/đo và ticket có owner cho phần vận hành chưa cần merge. |

### UI

| Mục | Đánh giá lần hai | Xử lý |
|---|---|---|
| P1 reply retry | Đã snapshot `replyToId`; giữ test. | Regression. |
| P2-1 confirmed predicate | Lỗi thật, thêm rủi ro closure cũ của React. | Một kết quả mutation/outbox, test event trước HTTP error. |
| P3-1 revoked quote | Lỗi thật. | Redact BE + guard UI + test. |
| P3-2 hai retry | BE dedupe ngăn trùng message, nhưng hai state UI phân kỳ; lỗi xác định hiện xóa row. | Một outbox/row, bỏ banner sau khi row giữ được payload. |
| P3-3 wizard | Chỉ persist sau submit. | Persist nháp trước submit; tách ID idempotency. |
| P3-4 action refresh | Lỗi hậu response không chắc chắn. | Revalidate sau error/success, guard mất quyền. |
| P3-5 failedAttemptStore | Map module vượt vòng đời account/conversation. | Loại khỏi composer, state theo account và cleanup. |
| P3-19 i18n | Fallback cứng còn. | Localized fallback + test VI/EN. |
| Policy | Review giả định Member sửa title/avatar là tùy chọn; **đã chốt Admin-only**. | BE auth transaction + UI chỉ Admin thấy controls; Member vẫn xem. |
| `conv:join`/typing | Review suy ra lỗi typing từ room nhưng code phát qua user-room. | Không sửa theo nhận xét này; giữ test socket add giữa phiên. |

## 4. Trình tự triển khai có phụ thuộc rõ ràng

| Giai đoạn | Repo / thay đổi | Điều kiện chuyển bước |
|---|---|---|
| 0. Baseline | Ghi lại HEAD, status, migration đã áp, test đang fail sẵn; audit số participant null-boundary, số tin có `createdAt <= joinedAt`, độ lệch `lastMessageId`/max message, số avatar orphan; lưu số liệu không chứa PII. | Có inventory dữ liệu và test tái hiện P1/P2. |
| 1. DB additive | BE `conversation/schema.ts`, migration mới do `db:generate`: cờ empty-boundary, `join_generation`, bảng cleanup avatar. Không chỉnh `0024`. Migrate DB sạch và DB nâng cấp; backfill an toàn theo mục 2.2, thống kê các hàng mơ hồ. | Migration chạy/rollback app được; legacy mơ hồ vẫn fail closed. |
| 2. BE quyền lịch sử | `message/pg.ts`, `conversation/{pg,service,dto}.ts`, `message/{service,controller}.ts`: writer timestamp đơn điệu, join boundary từ max message, `joinGeneration` trong DTO, cùng một SQL statement cho timeline/reply target, helper visibility dùng cả cờ, redacted realtime target + endpoint tải quote có quyền, DTO thu hồi. | DB race tests không lộ lịch sử, tin sau join hiển thị ở timeline/unread/inbox/reply; UI có dữ liệu xác định epoch. |
| 3. BE avatar/contract | `conversation/{service,pg}.ts`, `file/{controller,service}.ts`, storage interface + S3, cleanup worker: Admin-only, private GET bytes, swap+cleanup đúng trạng thái commit; `clientMsgId` required ở DTO + VI/EN. | Member bị 403 khi sửa, removed bị 403/404 khi tải; lỗi xóa ảnh cũ không làm hỏng ảnh mới; HTTP contract ổn. |
| 4. UI chuẩn bị tương thích | `conversation.api.ts`, avatar loader/hook + các màn hình dùng ảnh; ẩn controls của Member; bảo đảm mọi send payload có ID. **Đưa UI hỗ trợ private GET lên trước khi BE bỏ public GET** hoặc phát hành atomically. | UI hiển thị avatar nhóm và fallback khi BE private; không request ảnh khi mất quyền. |
| 5. UI gửi tin/UX | `useMessages`, `ChatShell`, `MessageComposer`, `MessageList`, mapper, realtime cleanup, `GroupWizard`, `ConversationInfoPanel`, locale: so sánh `joinGeneration` trước khi hiển thị nhóm/reconnect, single attempt/row, shared reconciliation, revoked guard, wizard draft, refresh action, VI/EN. | Retry chỉ trên row; xác nhận không lỗi giả, không mất draft/file; cache cũ bị xóa khi đổi epoch/account. |
| 6. Tích hợp và audit | Chạy suite BE/UI, DB E2E, hai node Redis, browser smoke hai account, migration upgrade, kiểm performance/lock, đối chiếu review2 từng mục; tạo ticket có owner cho các P2/P3 chưa làm. | Gate mục 6 đạt và release order được ghi trong PR. |

## 5. Ma trận kiểm thử bắt buộc

| Lớp | Ca phải có và kỳ vọng |
|---|---|
| BE SQL/DB race | Tái hiện pre-read boundary → remove/re-add → timeline; sau sửa không thể lấy tin giữa hai lần. Tương tự reply target. `cursor/after` cũ bị từ chối theo membership mới. Kết quả không dựa vào sleep ngẫu nhiên: dùng transaction barrier/hook test. |
| BE boundary | Nhóm rỗng → join → gửi tin cùng microsecond; re-add rỗng; re-add nhóm có tin; send transaction bắt đầu trước nhưng insert sau join; 10+ tin trong burst; tất cả timeline/preview/unread/read marker/canAccess đồng nhất. Legacy null mơ hồ fail closed, FK boundary bị hard-delete từ chối. |
| BE realtime | Sender reply target đã hide; recipient vào muộn; remove/re-add giữa lookup và emit; event không chứa body target trước join. Endpoint lấy quote trả đúng body với người được phép, 403/404 cho hidden/below-boundary/removed. Add member lúc socket đang mở vẫn nhận typing. Hai node và user-room không phát nội dung sang người đã mất quyền theo semantics commit/snapshot. |
| BE role/avatar | Member PATCH title và POST avatar bị 403; Admin thành công; Admin mất quyền giữa upload/finalize bị từ chối và object mới được dọn; GET avatar với token Admin/Member 200, người lạ/removed/anonymous 403/404, URL cũ không tải được sau thay; no redirect/presigned URL. Hai Admin upload song song, old-delete throw/`failed`, disband, retry worker, không xóa object còn được DB tham chiếu. |
| BE API/i18n | POST message thiếu/sai `clientMsgId` trả validation VI/EN; cùng ID cùng payload trả cùng message; cùng ID khác payload **không tạo tin thứ hai** (behavior hiện là trả tin cũ, ghi contract rõ); upload đã xong retry không tạo attachment/message trùng. Tin nội bộ/story không bị DTO HTTP làm gãy. |
| UI component | Member thấy roster nhưng không thấy/sử dụng edit title/avatar; Admin thấy và thao tác được. Ảnh nhóm tải qua bearer API, caching/refresh/403/logout/re-add revoke blob URL; user avatar public vẫn bình thường. Tin thu hồi không render quote/story/reactions/forwarded dù response/cache cũ còn field. |
| UI membership epoch | Remove khi offline → re-add trước reconnect: `getConversationById` trả `joinGeneration` mới; clear timeline/live/archive/quote/avatar và chặn render cache cũ trước `syncMissedMessages`. Tương tự khi nhóm chưa mở nhưng inbox refresh thấy version mới. Không xóa dữ liệu của cuộc hội thoại khác. Test event đến trước/sau verify. |
| UI send | Deferred HTTP + websocket confirm trước reject, confirm chỉ ở timeline, upload lỗi xác định, mạng uncertain, retry text/file/voice, hai lần gửi cùng nội dung, reply thay đổi sau lần gửi đầu, đổi hội thoại, logout, bị remove. Một retry trên row; draft mới không bị ghi đè; URL preview không rò rỉ. |
| UI wizard/action | Chọn người/tên → rời trang/reload → khôi phục; đổi payload tạo `clientRequestId` mới; bấm Tạo → timeout → retry cùng ID; avatar upload lỗi sau nhóm đã tạo; Hủy/success xóa nháp; action server commit rồi HTTP timeout vẫn làm mới quyền/roster. VI/EN có cùng key. |
| Tích hợp | `bun run typecheck`, `bun run lint`, `bun run fmt:check`, `bun run build`, `bun run test`, `bun run test:e2e` tại BE; UI `bun run lint`, `bun run fmt:check`, `bun run build`, `bun run test`. Dùng `.env` test phù hợp nhưng không in secrets. BE E2E trên Postgres/Redis thật và `group-chat-multinode`; UI browser smoke hai tài khoản/2 tab. UI hiện chưa có script `test:e2e`: nếu chưa bổ sung harness browser, báo rõ smoke thủ công và bằng chứng, không gọi Vitest là E2E browser. |

## 6. Migration, release và các rủi ro còn lại

1. **Migration:** cờ mới default `false` để app cũ và dữ liệu mơ hồ tiếp tục fail closed. Generate migration từ schema; chạy từ DB rỗng và DB đã có `0024`, kiểm `pg_constraint`, số row và thời gian khóa. Không sửa migration cũ/không đổi `ON DELETE RESTRICT` trong đợt này. Nếu cleanup outbox tạo bảng, worker có thể deploy trước writer. Backfill theo batch và khóa ngắn; rollback ứng dụng mới sang app cũ vẫn đọc cột cũ, nhưng app cũ sẽ có lại false denial và GET avatar public, vì vậy rollback cần đánh giá quyền riêng tư riêng.
2. **Thứ tự release:** UI sẵn gọi private avatar API và luôn gửi `clientMsgId` trước, sau đó BE bật private GET/Admin-only/required DTO; hoặc triển khai atomically khi không có rolling client cũ. Không mở URL presigned GET cho client mới. Kiểm lại CDN/proxy cache và `Cache-Control`. **URL presigned đã phát trước cutover** còn dùng được đến TTL thực tế (`.env.example` là 300 giây); trước khi tuyên bố chính sách private có hiệu lực tuyệt đối cho request mới, chờ hết TTL lớn nhất đã cấu hình hoặc xoay object key/thu hồi credential theo khả năng vận hành. Ảnh đã tải trong trình duyệt không thể thu hồi khỏi trí nhớ người xem.
3. **Hiệu năng:** đo thêm lookup max message mỗi send, timeline join membership, inbox viewer-aware subquery, GET avatar qua BE, và query typing. Dữ liệu benchmark: nhóm tối đa 10 thành viên, hội thoại dài, nhiều message đã hide, 2 node; không giả định index hiện có đủ nếu `EXPLAIN` cho thấy scan lớn.
4. **Legacy ambiguity:** không có thuật toán luôn khôi phục được thứ tự join/send cho record cũ cùng timestamp nếu không có event log. Đếm, báo cáo và xử lý thủ công/giữ fail closed; không biến mốc null thành quyền xem tất cả trước khi audit.
5. **Inbox mutable:** cursor `{activityAt,id}` giúp ổn định mốc giữa hai page nhưng hoạt động của row khác vẫn có thể đổi. UI dedupe, refresh inbox trên realtime; nếu sản phẩm đòi snapshot phân trang tuyệt đối, đó là dự án feed version riêng. Không tuyên bố đã giải quyết hoàn toàn trong PR này.
6. **Hard-delete tương lai:** FK RESTRICT sẽ chặn xóa user/message được dùng làm boundary. Hiện chưa có production hard-delete; mở ticket chứa contract 409/domain error, quy tắc retention và đường purge, không âm thầm đổi FK sang `SET NULL`.
7. **Mục deferred có điều kiện:** migration cũ không concurrent, nullable creator unique, tên FK, timing-safe hash, replay HTTP 200, error oracle, typing rate/lifecycle và tối ưu inbox. Mỗi mục cần ticket + owner + tiêu chí đóng; không gọi là fixed chỉ vì không sửa trong đợt này. Nếu test cho thấy leak/deadlock/áp lực DB thực tế, nâng lên gate trước merge.

## 7. Tiêu chí hoàn thành

- Không tái hiện được leak lịch sử remove/re-add qua timeline, reply target, realtime, preview, cursor **hoặc cache UI sau reconnect** tại các barrier test; không có tin gửi sau join bị ẩn bởi timestamp. Legacy chưa chứng minh được luôn fail closed.
- GET avatar nhóm đòi token và membership ở mọi request, không redirect bearer URL; chỉ Admin đổi title/avatar; object bị thay/disband được dọn có retry mà không xóa object hiện hành.
- POST message của client thiếu `clientMsgId` bị từ chối rõ VI/EN; retry cùng ID idempotent; UI có đúng một điểm retry trên dòng tin và giữ nội dung/file/draft qua lỗi.
- Wizard khôi phục nháp trong phiên; tin thu hồi không có quote/metadata hiển thị; các nhánh success, timeout và quyền đổi trong lúc request đều có test.
- BE/UI build, lint, format và unit/integration pass; DB E2E + hai node pass; browser smoke có bằng chứng; số liệu migration/performance và ticket deferred ghi trong PR. Chỉ báo “đã kiểm thử toàn diện” khi thật sự chạy đủ các gate này.

## Tài liệu tham chiếu

- [PostgreSQL 16 — Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html): hai `SELECT` ở Read Committed có thể thấy trạng thái khác nhau.
- [PostgreSQL — Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html): `now()` là đầu transaction, `clock_timestamp()` là thời gian thực.
- [AWS S3 — Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html): presigned URL là bearer token còn hiệu lực theo thời hạn.
- [TanStack Query — Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates): snapshot, rollback/reconciliation và cache mutation.
