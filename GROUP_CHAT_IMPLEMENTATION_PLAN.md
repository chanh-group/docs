# Kế hoạch triển khai Group Chat — LemonChat

> Trạng thái: phần triển khai BE/UI đã hoàn tất trên branch `feat/group-chat`; migration 0021–0022 đã áp dụng thành công vào database development đang cấu hình. Unit test, quality gate và từng E2E suite đều đạt khi chạy riêng; lượt E2E tổng hợp gần nhất bị Redis remote timeout ở bước khởi tạo direct-chat. Vẫn cần nghiệm thu realtime đa tab/reconnect, mobile, media trước phát hành. Đối chiếu ban đầu ngày 16/11/2026, sau các commit BE `8a36958` và UI `a757953` về xóa/sửa tin nhắn.
>
> Phạm vi: chat nhóm trên web và API/realtime tương ứng. Cuộc gọi nhóm, lời mời bằng liên kết, thông báo đẩy và nhóm công khai là hạng mục riêng.

## 1. Mục tiêu và tiêu chí hoàn thành

Người dùng tạo nhóm từ bạn bè, đặt tên, nhắn tin và gửi tệp, thấy đúng tên/avatar người gửi, xem và quản lý thành viên theo vai trò. Người mới chỉ xem lịch sử từ lúc vào; người rời hoặc bị xóa không đọc/gửi tin hay nhận sự kiện mới. Chat trực tiếp hiện tại hoạt động như trước.

Hoàn thành khi luồng ba tài khoản chạy được từ tạo nhóm đến thêm/xóa/rời nhóm trên desktop và mobile, kể cả tải lại trang, nhiều tab và reconnect; các kiểm thử quyền ở API, socket và cơ sở dữ liệu đều đạt.

## 2. Khảo sát mã hiện tại

| Khu vực | Điểm có sẵn | Khoảng trống cần xử lý |
| --- | --- | --- |
| Schema BE `src/domain/conversation/schema.ts` | `conversation_kind` có `DIRECT/GROUP`; `conversations` có `title`, `avatarUrl`; `conversation_participants` có `OWNER/ADMIN/MEMBER`, `joinedAt`, `leftAt`; migration mới nhất là `0020_chubby_stingray.sql` | Chưa có nghiệp vụ tạo/quản lý nhóm, ràng buộc quy mô nhóm, truy vấn metadata nhóm; migration nhóm tiếp theo phải sinh sau snapshot `0020` |
| Conversation BE `controller.ts`, `service.ts`, `pg.ts`, `dto/*` | API direct, inbox, detail, read; inbox phân trang; thành viên đang hoạt động được lọc | Chưa có API nhóm; response không trả title/avatar/role/count; `findById` join một đối tác nên không dùng để dựng danh sách nhóm |
| Message BE `service.ts`, `pg.ts`, `dto/*` | Gửi, timeline, sửa tin `TEXT`, thu hồi, xóa ở phía mình qua `message_hidden_for_users`, reaction, reply, forward, tệp và unread; tin thu hồi vẫn là placeholder trong timeline | Chưa trả tên/avatar người gửi; chưa áp dụng mốc `joinedAt` cho lịch sử; kiểm quyền trước transaction gửi tin có race với xóa thành viên; kiểm cursor/mark-read/forward chưa biết ranh giới lịch sử nhóm |
| Realtime BE `chat.gateway.ts`, `realtime.service.ts` | Tự join các conversation đang tham gia khi kết nối; phát tin/typing/read vào `conv:*`; có phòng `user:*` và event `message:hidden` riêng cho một người | Chưa có event thành viên/metadata; chưa buộc socket của người bị xóa rời phòng; `typing:start/stop` hiện chỉ kiểm socket còn trong phòng, cần kiểm lại membership sau thay đổi quyền và xử lý nhiều node |
| UI `src/features/friend/components/GroupWizard.tsx` | Chọn bạn bè và đặt tên theo hai bước | Gọi `POST /groups` trong `src/features/friend/api/friend.api.ts`, nhưng BE chưa có endpoint; hiện cho qua bước tiếp với **1 bạn**, khác quy tắc tối thiểu 2; thành công chưa mở nhóm vừa tạo |
| UI `src/features/conversation/*` | Có `isGroup` và truy vấn danh sách/chi tiết | Mapper dùng tên cứng “Nhóm trò chuyện”; thiếu avatar, memberCount, role và invalidation nhóm |
| UI `src/features/messaging/*` | Có bubble, typing theo số người, xóa ở phía mình/thu hồi, sửa tin văn bản của mình, panel thông tin và danh sách ảo hóa | `mapMessageResponse` luôn để `sender: undefined`; `MessageList` nhóm chưa có `senderId` ổn định để tách cụm, avatar mỗi tin đang nhận từ đối tác trực tiếp; typing nhóm chỉ hiện số người, chưa hiện tên; nút rời nhóm chỉ báo “Coming soon” |
| UI `src/features/call/*` và `ConversationHeader.tsx` | Có một số nhánh `isGroup`; header hiện nút gọi thoại/video cho cả nhóm | Chưa được coi là chức năng gọi nhóm đã hoàn thiện; phải ẩn/vô hiệu hành động gọi nhóm trước khi mở chat nhóm |

Các file đọc chính: `lemon-backend/src/domain/conversation/{schema,repository,pg,service,controller}.ts`, `lemon-backend/src/domain/message/{pg,service}.ts`, `lemon-backend/src/domain/realtime/{chat.gateway,realtime.service}.ts`, `lemon-ui/src/features/friend/{api,components}`, `lemon-ui/src/features/conversation`, `lemon-ui/src/features/messaging`, `lemon-ui/src/features/shell/components/ChatShell.tsx`.

## 3. Quy tắc sản phẩm đề xuất

Các giá trị dưới đây là mặc định để có thể triển khai; xác nhận với chủ sản phẩm trước khi chốt hợp đồng API.

- Tạo **nhóm mới độc lập**, không chuyển chat 1–1 sang nhóm hoặc tự chia sẻ lịch sử 1–1. Người tạo chọn tối thiểu **2 bạn bè khác** (tổng 3 người); tối đa **10 người kể cả chủ nhóm**. Đặt giới hạn trong constants BE và dùng cùng hợp đồng ở UI. Tên trim, dài 1–80 ký tự. `memberIds` không được trùng, chứa chính mình, tài khoản không hoạt động hoặc không phải bạn bè; không cho thêm người có quan hệ chặn theo quy tắc sản phẩm.
- Tạo nhiều nhóm có cùng tập thành viên được phép; mỗi lần tạo là một cuộc hội thoại riêng. Chống tạo trùng do double submit bằng idempotency key dành cho thao tác tạo nhóm (khác `clientMsgId` của tin nhắn).
- Hủy kết bạn sau khi vào nhóm không tự động xóa thành viên khỏi nhóm; quyền nhóm dựa trên membership. Theo quyết định của chủ sản phẩm ngày 16/11/2026, nếu hai người chặn nhau sau khi đã cùng nhóm thì họ vẫn ở trong nhóm, xem và gửi tin nhóm. Kiểm tra chặn vẫn áp dụng cho chat trực tiếp; không cho thêm người chưa phải bạn bè vào nhóm.
- Người tạo là `OWNER`; người được thêm là `MEMBER`. `OWNER` và `ADMIN` được đổi tên/avatar và thêm thành viên; chỉ `OWNER` được xóa người khác, đổi vai trò, chuyển quyền sở hữu. Thành viên được tự rời. Không cho tự xóa `OWNER`; nếu owner rời khi còn người khác, phải chuyển quyền trước. Khi chỉ còn owner, `leave` sẽ đóng mềm nhóm trong cùng transaction để không để nhóm không chủ. Chuyển quyền sở hữu hạ owner cũ thành `ADMIN`.
- Người mới hoặc được thêm lại chỉ thấy tin nhắn từ lần tham gia hiện tại. `joinedAt` hiện chỉ là timestamp; mốc A phải chọn ranh giới lịch sử **bền vững, có thứ tự toàn phần** (ví dụ sequence tăng theo conversation lưu trong participant) và đồng bộ nó với transaction gửi/thêm thành viên. Không dùng thời gian ở client hoặc `now()` của transaction làm ranh giới duy nhất; kiểm thử tin gửi đồng thời và tin trùng timestamp. Người rời hoặc bị xóa mất quyền xem nhóm ngay, bao gồm lịch sử cũ, reply/forward từ nhóm, API tệp và socket. Thành viên đang hoạt động không mất quyền xem tin cũ khi người gửi rời nhóm.
- Sau khi nhóm co lại còn 1–2 người, vẫn giữ `GROUP`, không tự đổi sang `DIRECT`. Khi chỉ còn owner, UI hiện trạng thái nhóm trống và cho thêm người hoặc đóng nhóm.
- Tin nhắn nhóm hiển thị tên/avatar theo **người gửi**, không theo tên/avatar nhóm. Typing có thể hiện tên tối đa 2 người và “N người đang nhập” khi nhiều hơn; không cần hiển thị trạng thái online tổng hợp của nhóm.
- Đọc tin trong nhóm là trạng thái **riêng từng thành viên**; trước mắt inbox hiển thị unread cá nhân. Quy ước dấu đã đọc trong bubble nhóm cần nêu rõ: hoặc ẩn dấu “đã đọc bởi tất cả”, hoặc có danh sách người đã đọc trong giai đoạn sau. Không suy luận đã đọc của cả nhóm từ một thành viên.
- Giữ đúng hành vi tin nhắn đã phát hành: người gửi có thể xóa ở phía mình hoặc thu hồi với mọi người; người nhận chỉ xóa ở phía mình; chỉ người gửi sửa được tin `TEXT` chưa thu hồi. Xóa ở phía mình tiếp tục ghi vào `message_hidden_for_users` và phát `message:hidden` tới phòng `user:*`; thu hồi giữ placeholder cho thành viên **có quyền xem tin đó**. Mọi hành động, reply quote, inbox preview và media archive phải tôn trọng cả `joinedAt`/ranh giới tham gia, `leftAt` và trạng thái ẩn riêng từng người.
- Avatar nhóm tùy chọn: nếu chưa có thì dùng biểu tượng/chữ cái tên nhóm. Upload có thể triển khai sau luồng tạo/nhắn tin cơ bản; không nhận URL tùy ý do client tự khai.

### Ma trận quyền dự kiến

| Thao tác | OWNER | ADMIN | MEMBER | Người ngoài/đã rời |
| --- | :---: | :---: | :---: | :---: |
| Xem nhóm, thành viên, lịch sử được phép; gửi/đọc tin | Có | Có | Có | Không |
| Thêm bạn bè, sửa tên/avatar | Có | Có | Không | Không |
| Xóa thành viên khác, đổi vai trò, chuyển owner | Có | Không | Không | Không |
| Tự rời nhóm | Theo quy tắc chuyển chủ/đóng nhóm | Có | Có | Không |

## 4. Hợp đồng API dự kiến

Giữ base `/api` và response envelope hiện có (`docs/RESPONSES.md`). Tên endpoint là đề xuất để khớp module Conversation; không triển khai `/groups` song song.

| Method/path | Input chính | Output/hiệu ứng |
| --- | --- | --- |
| `POST /conversations/groups` | `{ title, memberIds, clientRequestId }` | `201` + `ConversationResponse`; transaction tạo conversation, owner và members |
| `GET /conversations/:id` | ID | Metadata nhóm, `memberCount`, `myRole` (direct giữ schema tương thích) |
| `GET /conversations/:id/members` | ID | Danh sách thành viên hiện tại có ID, tên, avatar, role, joinedAt; paginate nếu nâng giới hạn |
| `PATCH /conversations/:id` | `{ title? }`, sau đó `{ avatarObjectKey? }` | Metadata mới, chỉ owner/admin |
| `POST /conversations/:id/members` | `{ memberIds }` | Danh sách thành viên mới; chỉ owner/admin; kiểm trùng/giới hạn/bạn bè |
| `DELETE /conversations/:id/members/:userId` | ID thành viên | Xóa thành viên, chỉ owner |
| `PATCH /conversations/:id/members/:userId/role` | `{ role: "ADMIN" | "MEMBER" }` | Đổi vai trò, chỉ owner |
| `POST /conversations/:id/transfer-ownership` | `{ userId }` | Chuyển owner atomically, owner cũ thành admin hoặc member theo hợp đồng |
| `POST /conversations/:id/leave` | Không có body | Ghi `leftAt`; owner phải chuyển quyền trước nếu còn thành viên khác; nếu là người cuối thì đóng mềm nhóm trong cùng transaction |

`ConversationResponse` nhóm bổ sung `title`, `avatarUrl`, `memberCount`, `myRole`; không cần gửi toàn bộ thành viên trong inbox. `MessageResponse` bổ sung `sender: { id, fullName, username, avatarUrl }` cho nhóm (có thể trả thống nhất cho direct), hoặc hợp đồng `senders` batch có định nghĩa rõ. Ưu tiên response chứa sender để tin realtime và timeline có cùng cấu trúc. Không lộ email/số điện thoại trong payload.

Các API tin nhắn hiện có vẫn dùng chung cho direct và group: `PATCH /messages/:id` (chỉ `TEXT` của chính mình), `DELETE /messages/:id` (thu hồi), `DELETE /messages/:id/for-me` (ẩn riêng), cùng timeline/send/reaction/forward/read. Khi mở quyền cho nhóm phải bổ sung kiểm tra thành viên và ranh giới lịch sử vào **từng** API này; không tạo đường tắt quyền chỉ vì UI đã ẩn nút. Với request trùng `clientRequestId`, trả lại nhóm cũ nếu payload khớp; nếu payload khác, trả `CONFLICT`.

Trả `NOT_PARTICIPANT`/`FORBIDDEN`/`BAD_REQUEST`/`CONFLICT` nhất quán với mã lỗi sẵn có; chỉ thêm mã nhóm cụ thể khi UI cần phân biệt (ví dụ `GROUP_FULL`, `OWNER_TRANSFER_REQUIRED`). Với ID nhóm không được phép xem, giữ cùng hành vi chống lộ thông tin như API hiện tại. Kiểm tra `kind === GROUP` cho mọi endpoint quản lý nhóm.

## 5. Thiết kế Backend theo thứ tự triển khai

### 5.1 Schema, migration, repository

1. Khảo sát snapshot Drizzle `0020`; giữ các cột `title`, `avatarUrl`, `role`, `joinedAt`, `leftAt` và bảng `message_hidden_for_users`. Thêm constraint/index thực sự cần sau khi xem query plan; không thêm bảng mới chỉ vì tính năng nhóm. Với `clientRequestId` tạo nhóm, thêm cột creator + request ID (nullable cho direct) và unique constraint theo cặp; kiểm tra cùng key nhưng khác payload trả `CONFLICT`.
2. Chọn và ghi rõ cơ chế mốc lịch sử trước migration: `joinedAt` phục vụ hiển thị nhưng không đủ chống race. Nếu dùng sequence theo conversation, cần backfill cho tin cũ, lưu mốc sequence lúc thêm/re-add thành viên và khóa cùng hàng conversation trong giao dịch gửi tin. Nếu dùng thứ tự khác, phải chứng minh không có tin gửi sau khi thêm bị xếp trước mốc. Dùng `db:generate` sinh migration sau `0020`, rà SQL/backfill và chạy migrate ở **DB test**. Không sửa tay file migration sinh tự động.
3. Mở rộng `ConversationRepository`: `createGroup`, `listMembers`, `getGroupMembership`, `addMembers`, `removeMember`, `updateGroup`, `transferOwner`, `leaveGroup`. Giao diện nên phân biệt dữ liệu direct/group bằng union có discriminator `kind` thay vì trường `partner` tùy chọn cho cả hai.
4. Transaction tạo nhóm: xác minh người dùng/bạn bè, insert một conversation `GROUP`, insert owner + thành viên, trả response. Xác minh lại điều kiện có thể đổi trong lúc giao dịch bằng khóa hoặc điều kiện DB phù hợp. Idempotency key phải có unique constraint theo creator và key; lặp request cùng payload trả cùng nhóm, không tạo nhóm thứ hai.
5. Thay `findById` khi là group để không join `partnerParticipants` rồi lấy ngẫu nhiên một người. Inbox lấy metadata nhóm trong truy vấn đã phân trang, memberCount theo batch/subquery phù hợp; tránh N+1. Dùng `leftAt IS NULL` thống nhất khi tìm thành viên hiện tại.
6. Thêm/xóa/chuyển quyền/đóng nhóm trong transaction, khóa conversation hoặc hàng membership theo thứ tự cố định. Với người từng rời: upsert và đặt lại `joinedAt`, mốc lịch sử, `leftAt`, read pointer; không cấp lại quyền xem lịch sử cũ. Đảm bảo đúng một owner hoạt động bằng transaction và kiểm thử cạnh tranh; cân nhắc constraint/trigger nếu cần bảo vệ cả đường ghi ngoài service.

### 5.2 Tin nhắn, lịch sử, unread, tệp

1. Tạo phép kiểm membership dùng chung cho timeline, gửi, sửa, thu hồi, xóa phía mình, reaction, reply, forward và mark-read. Chỉ gửi khi người dùng đang tham gia. `assertDirectConversationAllowed` hiện chỉ kiểm membership sơ bộ; ở transaction ghi tin, kiểm lại membership bằng khóa tương thích với thao tác xóa để tránh request gửi đã qua kiểm tra trước đó vẫn commit sau khi bị xóa.
2. Timeline và `after/cursor` chỉ trả tin sau ranh giới lịch sử của lần tham gia hiện tại; cursor không nằm trong miền được xem phải bị từ chối. `replyTarget` không được lộ tin trước mốc hoặc tin đã ẩn ở phía người xem; forward từ nguồn không được phép xem phải bị từ chối. `markAsRead`, sửa/thu hồi/reaction/xóa phía mình và truy vấn media archive chỉ chấp nhận tin thuộc miền hiển thị. Tin thu hồi trong miền đó vẫn xuất hiện dưới dạng placeholder.
3. `countUnread`, truy vấn batch, inbox preview và Redis revision phải tính từ mốc gia nhập, không tính tin trước khi vào hoặc tin đã ẩn cho người xem. Khi thêm lại thành viên, read pointer và cache unread phải được reset/invalidated; inbox có thể hiện số 0 khi nhóm chưa có tin mới.
4. Lấy profile sender bằng một truy vấn batch cho một trang timeline; tin realtime có sender sau khi gửi, không tạo N+1. Xác định fallback tên/avatar cho tài khoản bị vô hiệu hóa hoặc xóa.
5. Kiểm tra media authorization: URL tải đã ký có thể sống tới 24 giờ trong mapper hiện tại. Nếu chính sách yêu cầu thu hồi quyền tải ngay khi rời nhóm, cần endpoint tải có kiểm quyền hoặc URL sống rất ngắn; không tuyên bố rằng xóa socket đã thu hồi URL cũ. `FileService` hiện chưa có loại upload `GROUP_AVATAR`; khi triển khai avatar nhóm phải thêm loại/key riêng, xác nhận quyền owner/admin khi gắn object vào nhóm và không chấp nhận URL tùy ý. Quyết định về thu hồi URL là điều kiện an toàn cần chốt trước khi phát hành.

### 5.3 Realtime và đồng bộ cache

1. Sau commit DB, gửi `conversation:created` tới `user:*` của từng thành viên; `conversation:updated` và `member:added/removed/role-changed` tới đúng đối tượng. Event gồm `conversationId`, version/thời điểm cập nhật và payload tối thiểu; tránh dữ liệu cá nhân thừa.
2. Join tất cả tab/socket của thành viên mới vào `conv:*`; khi xóa/rời, buộc tất cả socket của người đó rời phòng trên mọi node **trước khi** phát thêm nội dung nhóm, đồng thời gửi event trực tiếp tới `user:*` của họ. Kiểm tra API/socket vẫn là lớp bảo vệ chính vì event có thể tới trễ; test race giữa gửi tin và xóa thành viên.
3. `conv:join` kiểm membership mỗi lần như hiện tại; `typing:start/stop` hiện chỉ nhìn `client.rooms`, vì vậy phải kiểm lại membership hoặc có cơ chế phiên bản quyền đủ tin cậy sau khi thành viên bị xóa. Xóa trạng thái typing cũ khi rời nhóm; client clear typing/cache và chuyển màn nếu chính mình bị xóa.
4. Sự kiện là best-effort sau commit. Khi socket đứt, UI refetch inbox/detail/membership và sync tin thiếu theo cursor; không dựa vào event như nguồn dữ liệu bền vững. Nếu sau này cần đảm bảo phát event qua nhiều node, thiết kế outbox riêng.

## 6. Thiết kế UI theo thứ tự triển khai

1. Mở rộng types và API trong `features/conversation`; chuyển `createGroup` đang nằm ở `features/friend/api/friend.api.ts` về conversation API. Sửa payload từ `{ name, memberIds }`/`POST /groups` sang `{ title, memberIds, clientRequestId }`/`POST /conversations/groups`. `useCreateGroup` thêm cache/invalidation và điều hướng tới `/messages/:id` sau thành công. Wizard chặn submit trùng, yêu cầu tối thiểu **2 bạn** và tối đa **9 bạn** khi giữ quy tắc tổng 3–10 người, hiển thị lỗi field/API và giữ `clientRequestId` khi retry cùng thao tác.
2. Mapper dùng `title`, `avatarUrl`, `memberCount`, `myRole` từ server; inbox, tìm kiếm, header và cửa sổ thông tin hiển thị đúng metadata nhóm. Preview inbox nhóm gắn tên người gửi cho tin của người khác, vẫn phân biệt tin của mình và marker tệp/thu hồi; không dựa vào `partner` của direct. Avatar fallback khi chưa có ảnh. Mobile giữ cơ chế back hiện tại.
3. Sửa luồng message mapper để dùng `sender`; thêm `senderId` vào `ChatMessage` và để `MessageList` nhóm hiện tên/avatar người gửi tại đầu cụm, direct vẫn dùng đối tác. `buildMessageGroups` hiện gộp theo `sender` hiển thị nên phải đổi sang `senderId`, tránh gộp hai người khác nhau có cùng tên. Reply quote dùng tên sender đúng; lịch sử và tin realtime cùng UI.
4. Panel nhóm: danh sách thành viên, badge owner/admin, thêm bạn bè, sửa tên, xóa thành viên, chuyển owner và rời nhóm; chỉ hiện thao tác hợp vai trò nhưng luôn để BE quyết định quyền. Có dialog xác nhận cho thao tác mất quyền, trạng thái loading/empty/error, bàn phím và nhãn hỗ trợ đọc màn hình, bản dịch Việt–Anh.
5. Realtime client cập nhật query cache/invalidation theo event nhóm. Khi bị xóa: hủy request đang chạy, xóa cache chi tiết/timeline/live messages/media archive/typing của nhóm, điều hướng về inbox, không để request cũ ghi lại dữ liệu nhạy cảm. Tái sử dụng luồng `message:hidden` cho xóa phía mình và giữ placeholder `message:revoked`; khi reconnect, làm mới cả danh sách nhóm lẫn membership.
6. Trước khi phát hành, ẩn hoặc vô hiệu nút gọi thoại/video trong `ConversationHeader` và đường vào call ở `ChatShell` cho `isGroup` nếu BE/luồng call chưa hỗ trợ nhiều người thực sự; kế hoạch cuộc gọi nhóm tách riêng.

## 7. Kiểm thử và tiêu chí nghiệm thu

| Nhóm kiểm thử | Kịch bản bắt buộc |
| --- | --- |
| Unit BE | Tạo nhóm hợp lệ/không hợp lệ, bạn bè/chặn, giới hạn và idempotency; role matrix; đổi owner; owner cuối; thêm lại thành viên; lỗi DB/Redis không bị biến thành lỗi nghiệp vụ |
| DB integration/e2e | Transaction rollback; hai request thêm thành viên đồng thời; gửi tin đồng thời với xóa thành viên; một owner; timeline/unread theo ranh giới lịch sử bền vững; cursor/reply/forward/mark-read/xóa phía mình trước mốc bị chặn; tin trùng timestamp không phá thứ tự |
| Socket e2e | 3+ tài khoản, nhiều tab, nhiều node nếu có hạ tầng test; member mới join, member bị xóa rời phòng; typing/read/message không rò; reconnect tự đồng bộ |
| UI | Wizard chọn người/tạo thành công/thất bại; inbox và header đúng; hai sender liên tiếp không gộp cụm; permission actions; dialog và i18n; tự rời/bị xóa; mobile |
| Hồi quy | Direct 1–1, story reply/reaction, gửi tệp, chỉnh sửa tin `TEXT`, xóa phía mình, thu hồi/placeholder, reaction, unread, media archive, scroll/pagination, reconnect; i18n VI/EN cho thao tác nhóm và tin nhắn |

Lệnh gate: BE `bun run typecheck`, `bun run lint`, `bun run fmt:check`, `bunx vitest run --pool=threads --maxWorkers=2`, `bun run build`, E2E qua Vitest config với PostgreSQL/Redis test và `--maxWorkers=1`; UI `bun run lint`, `bunx oxfmt --check src/`, `bunx vitest run --pool=threads --maxWorkers=2`, `bun run build`. Giới hạn worker vì Vitest mặc định từng timeout khi khởi tạo nhiều worker trong môi trường này. Chạy thử tương thích Node >= 20 và Bun cho Backend theo `AGENTS.md`.

Điều kiện hạ tầng: tách DB/Redis **test** khỏi `.env` đang dùng phát triển; giới hạn worker/pool cho E2E để tránh vượt giới hạn kết nối DB. Môi trường khảo sát hiện không có Redis tại `127.0.0.1:6379`, nên không coi việc chạy E2E với Redis là đã được xác nhận cho đến khi có dịch vụ test. Không chạy migration nhóm trên DB phát triển trước khi rà migration và rollback ở DB test.

## 8. Mốc bàn giao đề xuất

| Mốc | Kết quả kiểm chứng được | Phụ thuộc |
| --- | --- | --- |
| A. Hợp đồng và migration | Chốt ranh giới lịch sử có thứ tự toàn phần, DTO/API/event, ma trận quyền, migration sau `0020` có test; chuẩn bị DB/Redis test | Quyết định ở mục 3 |
| B. BE lõi | Tạo nhóm, inbox/detail/member CRUD, message/unread đúng quyền và mốc lịch sử | A |
| C. Realtime | Join/leave đa tab, event metadata, reconnect, test socket | B |
| D. UI | Wizard đến phòng chat; inbox/header/sender/panel quản lý | B, C |
| E. Nghiệm thu | E2e 3 tài khoản, hồi quy, kiểm thử mobile và bảo mật tệp | A–D |

Có thể chia PR nhỏ theo mốc để review, nhưng mỗi mốc phải giữ API direct đang chạy. Dùng cùng branch `feat/group-chat` hiện tại ở hai repo nếu tiếp tục cách tổ chức branch mà dự án đang dùng.

## 9. Điểm cần chốt trước khi code

1. Giới hạn 10 người và yêu cầu tất cả thành viên mới phải là bạn bè với người thêm có phù hợp sản phẩm không?
2. Người vừa vào có được xem lịch sử trước khi vào không? Kế hoạch mặc định **không**. Nếu giữ mặc định, phải chốt cơ chế mốc lịch sử bền vững ở mốc A; `joinedAt` đơn lẻ không đủ khi giao dịch gửi/thêm thành viên chạy đồng thời. Thay đổi câu trả lời này ảnh hưởng schema/query/unread và quyền riêng tư.
3. Có cần thu hồi **ngay lập tức** link tải tệp đã cấp khi thành viên bị xóa không? Nếu có, cần thay cơ chế tải hiện tại, không chỉ giảm TTL.
4. Bản đầu có cần avatar tùy chỉnh, quyền `ADMIN`, thông báo hệ thống “A thêm B”, hay chỉ tạo nhóm + nhắn tin + rời nhóm? Những phần này có thể triển khai theo mốc sau mà vẫn giữ hợp đồng mở rộng.
5. Quy tắc xử lý owner cuối cùng: bản kế hoạch mặc định `POST /leave` đóng mềm nhóm khi chỉ còn owner, và owner cũ trở thành `ADMIN` khi chuyển quyền; cần xác nhận thời hạn giữ lịch sử nhóm đã đóng.
6. Đã chốt: nếu A chặn B sau khi cả hai đã ở cùng nhóm, cả hai vẫn ở trong nhóm, xem và gửi tin nhóm. Quy tắc này không thay đổi việc kiểm tra chặn ở chat trực tiếp và không cho mời người không phải bạn bè.

### Tiến độ triển khai

- Đã triển khai: migration cho creator/idempotency/history boundary; API nhóm và ma trận quyền; giới hạn 10 người; mốc lịch sử cho timeline/unread/preview/reply/forward; sender profile; typing và cache realtime; wizard, metadata và panel thành viên VI/EN; ẩn gọi thoại/video nhóm. Người bị chặn sau khi cùng vào nhóm vẫn ở lại và có thể xem/gửi tin nhóm.
- Migration: `bun run db:migrate` đã áp 0021–0022 thành công; chạy lại sau đó cũng hoàn tất thành công.
- Kiểm tra UI ngày 16/11/2026: build, lint, kiểm tra format trong `src/` và Vitest đều đạt; 20 file, 132 test passed. Build phát cảnh báo bundle JavaScript lớn hơn 500 KB.
- Kiểm tra BE ngày 16/11/2026: typecheck, lint, format, build đều đạt; toàn bộ unit test đạt 77 file/813 test; sau sửa truy vấn, test repository conversation/message đạt 34 test và E2E group chat đạt 1/1.
- E2E hồi quy: group chat 1/1, direct chat 29/29 khi chạy riêng với timeout 30 giây, socket/readiness/Redis 11/11, story flow 5/5 sau khi nâng timeout của luồng nhiều request lên 90 giây, auth-version 4/4 sau khi mô phỏng lỗi trên kết nối listener và kiểm tra socket ngắt/kết nối lại. Kiểm thử ngắt PostgreSQL bằng PID qua pooler đã được thay bằng mô phỏng lỗi kết nối listener để kiểm tra hành vi phục hồi một cách xác định.
- Lượt E2E tổng hợp gần nhất: 6/7 suite và 21 test đạt; 29 test direct chat bị bỏ qua vì Redis remote không trả lời PING trong giới hạn khởi tạo, và cleanup của suite sau đó đã được bảo vệ khi setup thất bại. Direct chat đã đạt 29/29 trong lượt riêng trước đó. E2E hiện dùng dịch vụ development cấu hình trong `.env`, chưa có PostgreSQL/Redis test tách biệt.
- Chưa nghiệm thu: một lượt E2E tổng hợp xanh ổn định trên hạ tầng test riêng, realtime đa tab/reconnect của Group Chat, và kiểm thử mobile thủ công.
- Tải media hiện dùng URL presign có thời hạn 24 giờ; xóa thành viên không thu hồi được URL đã phát hành. Cần thay đổi cơ chế tải nếu sản phẩm yêu cầu thu hồi quyền tức thì; hiện tại đây là giới hạn cần xử lý trước khi tuyên bố đạt nghiệm thu bảo mật media.

## 10. Nguồn tham khảo

- [Discord — Group Chat and Calls](https://support.discord.com/hc/en-us/articles/223657667-Group-Chat-and-Calls): tạo từ bạn bè, giới hạn thành viên, đổi tên, danh sách thành viên, rời/xóa.
- [Slack — Understand direct messages](https://slack.com/help/articles/212281468-Understand-direct-messages) và [Add people to a direct message](https://slack.com/intl/en-gb/help/articles/1500002969782-Add-people-to-a-direct-message): group DM và quyết định chia sẻ lịch sử.
- [Socket.IO — Rooms](https://socket.io/docs/v4/rooms/) và [Server API](https://socket.io/docs/v4/server-api/): join/leave, broadcast, multi-server adapter.
- [OWASP — Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html): mặc định từ chối, kiểm tra quyền ở mỗi yêu cầu.
- [Drizzle ORM — Migrations](https://orm.drizzle.team/docs/migrations): sinh và áp migration từ schema.
