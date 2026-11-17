# Kế hoạch xử lý review Group Chat trước khi merge

> Ngày lập: 17/11/2026. Trạng thái: **đã triển khai và đạt kiểm thử tự động BE/UI; còn nghiệm thu thủ công**.
> Phạm vi: `lemon-backend` tại `72652b1`, `lemon-ui` tại `b7e8c35` và nhánh UI hiện tại sau merge `3165004`.
> Liên quan: [kế hoạch Group Chat tổng thể](./GROUP_CHAT_IMPLEMENTATION_PLAN.md). Tài liệu này tập trung vào lỗi cần sửa trước khi merge, không thay thế đặc tả sản phẩm của kế hoạch tổng thể.

## 1. Mục tiêu, phạm vi và điều kiện hoàn thành

### Mục tiêu

1. Thành viên nhóm chỉ nhận nội dung và metadata nhạy cảm của tin thuộc **lần tham gia hiện tại**; một `replyToId` dạng ID mờ có thể tồn tại để UI hiện trạng thái “tin không thể xem”. Người bị xóa/rời nhóm không nhận tin được tạo sau khi quyền của họ bị thu hồi.
2. Retry tạo nhóm thực sự idempotent: không cấp lại quyền realtime, không phát lại sự kiện tạo nhóm, vẫn trả kết quả hợp lệ sau khi tên nhóm hoặc tình bạn thay đổi.
3. Gửi tin thành công/thất bại không làm mất bản nháp hoặc tệp; retry không tạo tin trùng.
4. Reconnect xác minh lại quyền của nhóm đang mở và làm mới metadata, vai trò, thành viên; cache cũ không xuất hiện khi không còn quyền.
5. Chat trực tiếp, các thao tác tin nhắn khác, UI Việt/Anh và API hiện có không bị hồi quy.

### Quy tắc quyền dùng để nghiệm thu

- Với nhóm, ranh giới lịch sử là `historyBoundaryMessageId` của **membership đang hoạt động**. Giá trị `null` là hợp lệ cho thành viên từ lúc tạo nhóm hoặc tham gia khi chưa có tin; **không tìm thấy membership** là một trạng thái khác và phải bị từ chối.
- Tin được ghi sau khi thao tác xóa/rời/giải tán đã commit không được phát tới tài khoản bị thu hồi quyền. Tin đã ghi trước commit thu hồi có thể đang trên đường truyền; UI phải dọn dữ liệu khi phát hiện mất quyền. Đây là mốc thứ tự dùng trong test realtime.
- Với HTTP timeline, phép đọc được coi là đã được cấp quyền khi transaction đọc lấy được khóa membership trước transaction thu hồi. Nếu thu hồi đã commit trước thời điểm đó, request phải bị từ chối. Một request đọc hợp lệ đã bắt đầu trước thu hồi có thể hoàn tất sau commit thu hồi; test phải phân biệt hai thứ tự này và không được để truy vấn kế tiếp dùng boundary cũ.
- Tất cả phép kiểm quyền ở BE là nguồn quyết định cuối. Việc ẩn dữ liệu/cache ở UI bảo vệ trải nghiệm và tránh lộ từ cache, không thay thế kiểm quyền server.
- Các URL tải tệp đã ký trước lúc thu hồi có thể còn hiệu lực đến hạn của chúng theo thiết kế hiện tại; kế hoạch này ngăn cấp dữ liệu/URL mới sau khi mất quyền. Chính sách vô hiệu ngay URL đã cấp thuộc thay đổi riêng về cơ chế tải tệp.

### Điều kiện hoàn thành

Mọi ca trong ma trận kiểm thử ở mục 8 đạt, bao gồm các thứ tự race có điều khiển; BE và UI cùng sử dụng hợp đồng sự kiện mới; quality gate hai repo xanh; nghiệm thu ba tài khoản, nhiều tab và **hai node dùng Redis adapter**. `src/main.ts` bật Redis adapter khi khởi động, nên test socket chỉ dùng `IoAdapter` trong bộ E2E hiện tại không đủ làm điều kiện merge.

## 2. Bản đồ lỗi và nguyên nhân

| ID | Ưu tiên | Đường xử lý hiện tại | Nguyên nhân và ảnh hưởng |
| --- | --- | --- | --- |
| BE-1 | P1 | `lemon-backend/src/domain/message/service.ts` → `realtime/realtime.service.ts` | `replyTarget` được dựng theo quyền người gửi rồi gửi nguyên cho `conv:*`; thành viên mới có thể nhận nội dung tin trước khi vào nhóm. |
| BE-2 | P1 | `conversation/service.ts` → `conversation/pg.ts` | Cả nhánh tạo mới lẫn retry đều gọi `joinConversation(input.memberIds)` và phát `conversation:created`; người đã bị xóa có thể được đưa trở lại room. |
| BE-3 | P1 | `realtime/chat.gateway.ts`, `realtime/realtime.service.ts` | Membership được kiểm trước `client.join`; thao tác xóa có thể `socketsLeave` trước khi join hoàn tất. Còn đường tự join lúc kết nối và `socketsJoin` từ service. |
| BE-4 | P1 | `message/service.ts`, `message/pg.ts` | Sau kiểm quyền, membership có thể biến mất; optional chaining chuyển kết quả thành boundary `undefined`. Truy vấn timeline/reply target bỏ điều kiện lịch sử. |
| BE-5 | P2 | `conversation/pg.ts` | Retry so tên hiện tại với tên gốc dù `createRequestHash` khớp; đổi tên làm retry thất bại. Creator đã rời nhóm còn có đường `findById(...)!` trả `null`. |
| UI-1 | P1 | `lemon-ui/src/features/messaging/components/conversation/MessageComposer.tsx` | Xóa draft trước khi request kết thúc; nếu đã có draft mới thì nội dung gửi lỗi không được khôi phục. Tệp upload lỗi trước khi có `retryPayload` cũng cần giữ lại. |
| UI-2 | P2 | `ConversationView.tsx` | Sau khi composer đã xóa nội dung của lần gửi, `setDraft` lần hai có thể xóa bản nháp mới trùng văn bản vừa gửi. |
| UI-3 | P2 | `hooks/useMessagingRealtime.ts`, `shell/components/ChatShell.tsx` | Reconnect chỉ refresh inbox và tải tin nhỡ; nhóm trong inbox cache làm query chi tiết bị bỏ qua. Bỏ lỡ sự kiện xóa thành viên khiến cache, vai trò và roster cũ tiếp tục hiển thị. |
| BE-6 | P1, phát hiện bổ sung | `message/service.ts` → `realtime/realtime.service.ts` | `message:updated` phát **toàn bộ nội dung tin cũ** cho room sau khi tin được sửa. `message:reaction`/`message:revoked`/`read:updated` cũng cần rà dữ liệu lịch sử trong payload. Sửa riêng BE-1 sẽ chưa chặn hết lộ lịch sử. |
| BE-7 | P1, phát hiện khi rà kế hoạch | `message/{service,pg}.ts`, `conversation/{service,pg}.ts` | Sửa/thu hồi/ẩn tin và mark-read kiểm membership ở service, nhưng truy vấn ghi tương ứng chưa khóa hoặc kiểm lại membership trong transaction. Nếu bị xóa sau bước kiểm tra, thao tác cũ vẫn có thể ghi DB và phát event. `reactMessage` đã khóa participant, có thể dùng làm mẫu. |

**Lưu ý UI:** input hiện thường bị `disabled` khi `isSending`, nên ví dụ gõ tiếp giữa request khó tái hiện bằng thao tác thông thường. Logic draft vẫn phụ thuộc timing bất đồng bộ và sẽ không an toàn khi có cập nhật draft khác hoặc cho phép soạn tin song song. `ConversationView` được `key` theo ID hội thoại; chuyển phòng giữa request còn làm state cục bộ unmount.

## 3. Thiết kế BE: kiểm quyền đọc và sự kiện realtime

### 3.1 Đọc timeline và reply target — BE-4

**File dự kiến:** `src/domain/message/{service,repository,pg}.ts`; có thể mở rộng `src/domain/conversation/pg.ts` để dùng chung điều kiện quyền đọc.

1. Thay hợp đồng đọc từ `viewerId + boundary tùy chọn` sang kết quả phân biệt rõ `authorized membership`, `boundary = null` và `not participant`. Với direct, giữ chính sách quyền direct đang có; chỉ group dùng boundary của lần tham gia hiện tại.
2. Đặt điều kiện participant đang hoạt động và boundary **trong truy vấn lấy tin**, cùng điều kiện ẩn riêng cho viewer. Truy vấn reply target phải dùng cùng quy tắc. Không chấp nhận cursor/`after` ngoài miền nhìn thấy.
3. Đọc membership, trang tin, attachment/reaction và reply target trong một luồng DB nhất quán. Phương án ưu tiên: transaction đọc có khóa `FOR SHARE` trên hàng participant trong lúc thu thập dữ liệu DB; thao tác xóa/rời/giải tán hiện cập nhật hàng này và phải chờ. Nếu cần khóa thêm hàng conversation, dùng cùng thứ tự khóa với transaction ghi (conversation trước, participant sau) để tránh deadlock. Mỗi truy vấn lấy tin vẫn có điều kiện membership/boundary. Không giữ khóa khi ký URL storage hoặc gọi mạng.
4. Khi không còn membership, trả `NOT_PARTICIPANT` (403); không trả trang rỗng để che mất quyền, không ký URL. Nếu membership thay đổi giữa các bước đọc, fail closed thay vì trả một phần dữ liệu.
5. Test hàng `historyBoundaryMessageId = null` hợp lệ, membership bị xóa, vào lại với boundary mới, cursor và reply target trước mốc, tin đã ẩn riêng; test thứ tự xóa giữa kiểm quyền và đọc DB.

### 3.2 Hợp đồng sự kiện nhóm — BE-1 và BE-6

**File dự kiến:** `src/domain/message/service.ts`, `src/domain/realtime/realtime.service.ts`, các phương thức repository đọc người nhận hợp lệ.

1. Phân loại các event nhóm: `message:new`, `message:updated`, `message:revoked`, `message:reaction`, `read:updated`, `typing`. Chỉ gửi trường dữ liệu mà **người nhận cụ thể** được phép xem. Không phát nội dung trích dẫn hoặc nội dung tin cũ nguyên dạng vào `conv:*`.
2. Với tối đa 10 thành viên/nhóm, lấy **danh sách người nhận đang hoạt động bằng truy vấn batch** sau thao tác DB; phát event nhạy cảm qua `user:*` của từng người hợp lệ. Lọc thêm theo boundary và tin ẩn riêng cho các event tham chiếu tin cũ. Tránh một truy vấn SQL cho mỗi người nhận; chuẩn bị kết quả quyền theo lô.
3. `message:new`: tin mới được gửi cho thành viên đang hoạt động. Phần `replyTarget` được cá nhân hóa theo quyền xem target của từng người; nếu không được xem thì `replyTarget = null` và chỉ giữ `replyToId` theo hợp đồng UI. Không gửi `body`, `storyPreview`, sender profile của target cũ tới người không có quyền. Loại `clientMsgId` khỏi event chung cho các tab/người nhận; `message:correlated` vẫn chỉ gửi cho người gửi. Response HTTP cho người gửi vẫn theo quyền người gửi.
4. `message:updated`: người nhận không có quyền xem tin gốc không nhận body đã sửa. `message:reaction`, `message:revoked` và `read:updated` không gửi metadata của tin trước mốc. Event typing nhóm xác thực người gửi và fanout theo danh sách thành viên hiện tại qua `user:*`, không yêu cầu socket đang nằm trong `conv:*`.
5. Giữ contract direct chat hiện tại nếu phù hợp. Kiểm tra tương thích handler UI trước khi đổi payload; nếu event cá nhân hóa có cùng shape thì UI chỉ cần xử lý `replyTarget = null` đúng cách. `ChatShell` chỉ dựng trích dẫn từ các tin viewer đã được BE cho phép và cache thuộc membership hiện tại.
6. Rà inbox preview, media archive và sự kiện khác dùng `conv:*` để bảo đảm cùng quy tắc lịch sử/thu hồi; ghi lại event nào chỉ chứa ID và event nào chứa dữ liệu nhạy cảm.
7. Với `editMessage`, `revokeMessage`, `hideMessageForMe` và `markAsRead`, kiểm quyền thành viên **trong transaction ghi** hoặc thêm điều kiện participant đang hoạt động vào lệnh ghi để đóng race sau bước kiểm ở service. Kiểm cùng boundary của tin được thao tác; giữ thứ tự khóa nhất quán với xóa thành viên. `reactMessage` đã kiểm/khóa participant trong transaction. Chỉ phát event nếu thao tác ghi hợp lệ thực sự commit.

### 3.3 Join/leave và thứ tự thu hồi quyền — BE-2, BE-3

**File dự kiến:** `src/domain/realtime/{chat.gateway,realtime.service}.ts`, `src/domain/conversation/service.ts`, UI realtime handler.

1. `conv:join`: sau khi `client.join(room)` hoàn tất, kiểm lại membership; nếu đã mất quyền thì `client.leave(room)`, trả thất bại và phát mã lỗi ổn định. Kiểm socket còn kết nối trước khi xác nhận thành công.
2. `initializeConnection`: sau khi join các phòng ban đầu, kiểm lại từng conversation đang tham gia và rời phòng bị thu hồi. Không để danh sách lấy trước thao tác xóa quyết định quyền cuối cùng.
3. Đối với **nhóm**, bỏ đường service tự `socketsJoin` theo danh sách của request tạo/thêm người. Sau commit, phát `conversation:created` hoặc `member:added` vào `user:*` để UI làm mới inbox/roster. Vì tất cả event nhóm, kể cả typing, đã đi qua `user:*` theo mục 3.2, client **không cần phát thêm `conv:join`** khi được mời; socket offline nhận đồng bộ khi reconnect. `conv:*` có thể còn được duy trì để tương thích trong giai đoạn này nhưng không được là kênh phát dữ liệu nhóm nhạy cảm. Giữ luồng direct riêng và rà tác động với multi-tab.
4. Xóa/rời/giải tán tiếp tục gọi `socketsLeave` để dọn room nhanh và gửi sự kiện riêng tới tài khoản bị tác động. Việc phát nội dung nhóm theo người nhận đang hoạt động ở mục 3.2 là lớp bảo vệ khi lệnh leave tới trễ hoặc room bị join lại. Test phải khẳng định không còn bất kỳ group event nhạy cảm nào phát qua `conv:*`.
5. Định nghĩa thứ tự nghiệm thu theo commit DB: tin ghi **sau** commit thu hồi không được gửi tới người bị thu hồi. Dùng promise/barrier ép `check → remove → leave → join`, cả đường kết nối ban đầu và thêm thành viên. Kiểm thử thực tế với Redis adapter ở hai node; nếu không đạt, bổ sung cơ chế đồng bộ phát/thu hồi theo conversation trước khi merge, không hạ tiêu chí để test xanh.

## 4. Thiết kế BE: idempotency tạo nhóm — BE-2, BE-5

**File dự kiến:** `src/domain/conversation/{repository,pg,service}.ts` và test tương ứng.

1. Repository `createGroup` trả một kết quả có trạng thái phân biệt `created` và `replayed` thay vì chỉ trả conversation. Cả đường tìm thấy từ đầu và đường bắt unique violation dùng cùng logic kiểm hash.
2. Cùng `creatorId + clientRequestId`: so `createRequestHash` được tính từ tên đã trim và danh sách `memberIds` đã sort; không so `existing.title`. Payload khác vẫn `CONFLICT`. Retry hợp lệ trả **tên/avatar/vai trò/số thành viên hiện tại**.
3. Kiểm tính hợp lệ cấu trúc request và hash trước. Với yêu cầu tạo **mới**, kiểm user hoạt động và tình bạn như hiện tại. Với **retry**, không áp lại tình bạn hiện tại của danh sách gốc. Nếu hai request đồng thời đều chưa nhìn thấy bản ghi, nhưng một request tạo xong trong lúc request kia đang kiểm tình bạn, nhánh thứ hai phải tra lại idempotency key trước khi trả lỗi kiểm tình bạn. Người tạo đã rời nhóm nhận `NOT_PARTICIPANT` (403); nhóm đã giải tán nhận `CONFLICT` (409), tuyệt đối không đưa `null` vào mapper.
4. Chỉ trạng thái `created` mới phát `conversation:created`; retry không join hay phát lại event. Phản hồi HTTP giữ hợp đồng đang dùng để tránh thay đổi UI không cần thiết.
5. Test retry sau đổi tên/avatar, xóa thành viên gốc, đổi quan hệ bạn bè, người tạo rời nhóm, giải tán, cùng key khác payload và hai request đồng thời. Khẳng định socket bị xóa không nhận tin mới sau retry.

## 5. Thiết kế UI: lần gửi, bản nháp và retry — UI-1, UI-2

**File dự kiến:** `src/features/messaging/components/conversation/{MessageComposer,ConversationView}.tsx`, `hooks/useMessages.ts`, `utils/messageSend.ts`; test component/hook.

1. Gắn mỗi lần gửi với một snapshot bất biến: `conversationId`, `text`, `pendingFiles`, `replyToId`, `clientMsgId` và ID lần gửi. Bản nháp hiện tại là state khác, không dùng nội dung bằng nhau để đoán nó có phải bản nháp cũ không.
2. Đặt dữ liệu lần gửi và trạng thái lỗi ở một outbox theo conversation **ngoài state cục bộ bị unmount của `ConversationView`**. Outbox là nguồn trạng thái duy nhất cho lần gửi; bubble optimistic được đối chiếu với nó, tránh hiện đồng thời hai bản lỗi. Tin gửi lỗi có trạng thái và thao tác retry/bỏ rõ ràng. Khi request lỗi, giữ cả lần gửi lỗi lẫn draft mới; không ghi đè hoặc tự nối hai nội dung. Tệp lỗi trước bước tạo `retryPayload` vẫn có `File` gốc để retry trong phiên hiện tại.
3. Retry **cùng nội dung** giữ `clientMsgId`; nếu người dùng sửa nội dung/tệp rồi gửi thành lần mới, tạo ID mới. Phân biệt lỗi chắc chắn thất bại với lỗi mạng/5xx không rõ server đã nhận hay chưa; giữ trạng thái `uncertain` và đối chiếu tin từ server trước khi khẳng định thất bại.
4. Composer xóa nội dung của **đúng lần gửi vừa bắt đầu**. Bỏ `setDraft(current === text ? '' : current)` sau khi gửi xong trong `ConversationView`. Chỉ xóa reply context nếu nó vẫn thuộc lần gửi vừa thành công; reply context của bản nháp mới phải được giữ.
5. Giữ object URL preview tới khi gửi thành công hoặc người dùng bỏ lần gửi; giải phóng khi hoàn tất/unmount theo chủ sở hữu state mới. Không tạo upload lặp nếu tệp đã có `uploadedAttachment` hợp lệ.
6. UI lỗi/retry, nhãn trạng thái và thông báo phải có khóa i18n VI/EN. Xác định rõ giữ draft trong phiên ứng dụng; lưu `File` qua reload trình duyệt nằm ngoài phạm vi nếu chưa có cơ chế lưu tệp bền vững.

## 6. Thiết kế UI: xác minh quyền khi reconnect — UI-3

**File dự kiến:** `src/features/messaging/hooks/useMessagingRealtime.ts`, `src/features/shell/components/ChatShell.tsx`, `src/features/conversation/hooks/useConversations.ts`, `src/features/messaging/hooks/useMessages.ts`, các API helper.

1. Dùng trạng thái xác minh theo `conversationId`/phiên kết nối: `verifying`, `allowed`, `revoked`, `retryable-error`. Đánh dấu nhóm cần xác minh ngay khi socket mất kết nối/đổi thế hệ kết nối, trước khi render dữ liệu từ cache. Khi reconnect và có nhóm đang mở, gọi trực tiếp `GET /conversations/:id` dù inbox cache vẫn chứa nhóm. Che nội dung nhóm cũ trong lúc xác minh. Cổng quyền phải chặn cả việc khởi chạy `useConversationMessagesQuery`, `useGroupMembersQuery`, prefetch và tải tin nhỡ cho nhóm đang chờ xác minh; `ChatShell` hiện khởi chạy query tin ngay từ `selectedId`.
2. Khi xác minh thành công, cập nhật detail, refetch inbox và roster; sau đó mới đồng bộ tin nhỡ. Làm mới inbox bằng kết quả server trước khi cho hiện lại preview nhóm: `invalidateQueries` đơn thuần giữ preview cũ trong lúc refetch. Nếu quyền của nhóm khác cũng thay đổi lúc offline, mục nhóm đó phải biến mất khi inbox mới về. Nếu refresh inbox thất bại, tiếp tục che preview nhóm cũ và cho thử lại, không bật lại cache chưa xác minh. Quyền và tên/avatar/thành viên trong panel phải phản ánh server. Tránh gọi lặp khi effect đăng ký lại vì đổi `activeConversationId`; dùng generation ID để bỏ kết quả của lần xác minh cũ.
3. Với 403/404, hủy query đang chạy, dọn `conversation`, `groupMembers`, `messages`, `liveMessages`, `mediaArchive`, preview trong inbox và typing; điều hướng khỏi nhóm. Dữ liệu tin/quote/preview lấy từ server bị xóa; **văn bản và tệp chưa gửi do chính người dùng tạo** được giữ trong vùng phục hồi cục bộ, không còn nút retry vào nhóm mất quyền, có thao tác sao chép hoặc bỏ. Handler `message:new` và response đến trễ không được ghi lại cache vừa dọn. Không coi lỗi mạng hoặc 5xx là mất quyền: giữ màn hình chờ an toàn và nút thử lại.
4. `cancelQueries` cần đi kèm `AbortSignal` từ TanStack Query xuống axios/request nơi phù hợp; query hiện tại chưa tiêu thụ signal. Dù đã hủy, vẫn cần generation guard quanh cập nhật thủ công và các event realtime. Tách thông báo lỗi quyền với lỗi tải tin để không hiện toast sai hoặc lặp vô hạn.
5. Khi thành viên bị xóa rồi thêm lại trong lúc offline, dọn timeline và quote cũ trước khi tải lại theo boundary của membership mới. Không dùng cache cũ để dựng reply target. Bản gửi chưa hoàn tất chỉ được tái sử dụng sau khi xác minh quyền mới và kiểm tra lại reply target/tệp; nếu không hợp lệ, chuyển sang vùng phục hồi cục bộ thay vì tự retry.
6. Giữ đường direct chat và hành vi mở URL trực tiếp. Xử lý đồng thời desktop/mobile, nhiều tab và đổi nhóm trong lúc xác minh.

## 7. Thứ tự thực hiện và điểm bàn giao

| Giai đoạn | Công việc | Kết quả review được |
| --- | --- | --- |
| A — hợp đồng quyền | Chốt mốc commit thu hồi, bảng payload event và shape `replyTarget = null`; bổ sung test tái hiện 8 lỗi + BE-6/BE-7 trước khi sửa | Test đỏ đúng nguyên nhân, không phụ thuộc `sleep` |
| B — BE đọc dữ liệu | Membership/boundary nhất quán cho timeline, reply target; lỗi fail closed | E2E query/race xanh, không trả URL ngoài quyền |
| C — BE realtime | Fanout theo người có quyền, join/revoke, retry tạo nhóm | Unit + socket E2E một node và **hai node Redis** xanh |
| D — UI gửi tin | Snapshot lần gửi, outbox, retry, xóa draft đúng thời điểm | Test async text/file/reply/chuyển phòng xanh |
| E — UI reconnect | Xác minh quyền, cache gate, refresh metadata/roster, request cancellation | Test missed event, revoke, re-add, network failure xanh |
| F — nghiệm thu | Hồi quy, VI/EN, quality gates, kiểm thử thủ công đa tài khoản | Báo cáo kết quả và giới hạn còn lại trước merge |

Các thay đổi trong tài liệu này **dự kiến không cần migration DB**: hash tạo nhóm, membership và boundary đã có. Nếu kiểm thử hai node chứng minh cần version quyền lưu bền vững, thiết kế migration và cập nhật kế hoạch trước khi triển khai phần đó.

## 8. Ma trận test chi tiết

### 8.1 Backend unit/integration

| Case | Thiết lập và thứ tự | Kỳ vọng |
| --- | --- | --- |
| Reply trước mốc | A gửi T0, thêm B, A reply T0 | B nhận tin mới nhưng không nhận `body/storyPreview/sender` của T0; A vẫn thấy quote |
| Sửa tin trước mốc | A gửi T0, thêm B, A sửa T0 | Không có body/metadata T0 trong event tới B |
| Reaction/revoke/read trước mốc | A thao tác trên T0 sau khi thêm B | B không nhận metadata T0 vượt quyền |
| Xóa thành viên giữa kiểm quyền và thao tác ghi | Chặn `updateBody`/`softDelete`/`hideForUser`/`updateLastRead`, xóa người thao tác rồi nhả barrier | Thao tác ghi bị từ chối, không phát event, không sửa dữ liệu |
| Timeline bị xóa giữa bước đọc | Dùng barrier tại membership/query, xóa B rồi cho query tiếp | 403 hoặc kết quả fail closed; không có tin/quote/URL cũ |
| Boundary `null` hợp lệ | Thành viên ban đầu hoặc nhóm chưa có tin | Vẫn đọc được đúng tin hợp lệ; không nhầm với mất membership |
| Thêm lại thành viên | Xóa B, gửi T1, thêm lại B, gửi T2 | B chỉ xem T2, kể cả reply, cursor, media |
| Join/xóa race | Chặn `client.join` hoặc autojoin, xóa B rồi nhả promise | B không còn trong room; không nhận tin ghi sau xóa |
| Add/remove race | Chặn thông báo/join sau add, xóa trước khi join kết thúc | Join bị từ chối; group event không cấp lại quyền |
| Retry sau xóa/đổi tên | Tạo nhóm, đổi tên/xóa B, retry key cũ | Trả trạng thái hiện tại cho creator còn quyền; B không được join; không phát `conversation:created` lần hai |
| Retry sau đổi tình bạn | Tạo nhóm rồi thay đổi quan hệ bạn bè | Retry cùng hash hợp lệ theo membership hiện tại; tạo mới vẫn phải kiểm tình bạn |
| Retry sai payload/creator đã rời | Dùng cùng key với payload khác; hoặc creator mất quyền | Conflict/lỗi quyền ổn định, không crash hoặc lộ metadata |

Unit test dùng mock I/O và barrier; DB integration/E2E dùng PostgreSQL thật cho snapshot, row lock, unique violation và boundary. Test socket dùng `socket.io-client` thật. **Bắt buộc** có bài test hai app với Redis adapter như `src/main.ts`, ít nhất hai socket trên hai node và thao tác xóa từ node khác; suite socket hiện dùng `IoAdapter` trong bộ nhớ nên không bao phủ điều này. Test phải khẳng định **payload mà socket nhận**, không chỉ kiểm đã gọi hàm emit/leave. Dùng barrier để ép race; phép khẳng định không nhận event có thể dùng cửa sổ quan sát hữu hạn, tránh `sleep` tùy ý để tạo thứ tự race.

### 8.2 UI component/hook/integration

| Case | Kỳ vọng |
| --- | --- |
| Gửi “ok”, tạo draft “ok” mới trước khi request thành công | Draft thứ hai còn nguyên |
| Gửi thất bại sau khi có draft mới | Lần gửi lỗi và draft mới cùng tồn tại, mỗi cái có thao tác rõ |
| Upload lỗi trước/sau khi có `retryPayload` | Tệp gốc và preview còn để retry; không mất hoặc gửi trùng |
| Lỗi không rõ đã gửi, retry | Cùng `clientMsgId`; tin đã xác nhận từ server chỉ hiện một lần |
| Sender nhận cả response HTTP, `message:correlated` và `message:new` qua user room | Chỉ một bubble/tin trong timeline và preview |
| Chuyển phòng giữa request | Kết quả thuộc đúng conversation, không khôi phục draft nhầm phòng |
| Reconnect sau khi bị xóa/giải tán lúc offline | Che cache, xác minh 403/404, xóa cache và điều hướng; không có response cũ ghi lại |
| Reconnect trong khi inbox còn preview nhóm đã mất quyền | Preview nhóm được che tới khi inbox mới về; mục nhóm bị xóa biến mất sau refresh |
| Refresh inbox lỗi khi reconnect | Preview nhóm cũ tiếp tục bị che; có thao tác thử lại, không hiện dữ liệu cũ |
| Reconnect vẫn còn quyền nhưng role/roster đổi | Hiển thị metadata và danh sách mới, tin nhỡ được đồng bộ sau xác minh |
| Reconnect khi query timeline/roster đang chạy | Query bị hủy hoặc kết quả cũ bị bỏ; không ghi cache sau khi quyền bị thu hồi |
| Reconnect lỗi mạng/5xx | Không xóa quyền giả, không hiện cache nhạy cảm, có cách thử lại |
| Bị xóa rồi thêm lại khi offline | Chỉ nạp lịch sử thuộc lần tham gia mới, quote không lấy từ cache cũ |
| VI/EN và direct regression | Nhãn/lỗi/retry đúng hai ngôn ngữ; direct chat không đổi hành vi |

### 8.3 Quality gate và nghiệm thu

- BE: `bun run typecheck`, `bun run lint`, `bun run fmt:check`, `bun run test`, `bun run build`, `bun run test:e2e` với PostgreSQL/Redis test riêng. Xác nhận chạy trên Node >= 20 và Bun theo `lemon-backend/AGENTS.md`.
- UI: `bun run lint`, `bun run fmt:check`, `bun run test`, `bun run build`.
- Kiểm thủ công ba tài khoản: thành viên gốc, người vào sau, người bị xóa; hai tab/tài khoản, mất mạng/reconnect, sửa/reply tin cũ, gửi text và tệp, đổi tên, retry tạo nhóm. Ghi rõ phiên bản/commit và kết quả từng ca.
- Không chạy migration trên database phát triển cho kế hoạch này khi schema không đổi. Nếu phát sinh migration, chạy trên DB test trước và kiểm rollback/forward migration theo quy trình dự án.

## 9. Rủi ro triển khai và kiểm soát

| Rủi ro | Cách kiểm soát |
| --- | --- |
| Fanout theo user room làm tăng số event/truy vấn | Nhóm tối đa 10 người; truy vấn danh sách người nhận và quyền theo lô, đo số query trong test. Không tạo N+1. |
| Gói tin gửi trước khi thu hồi đến sau thông báo xóa | Dùng mốc commit DB đã định nghĩa; UI xử lý sự kiện `member:removed`/`conversation:disbanded` trước khi hiển thị hoặc ghi cache tiếp, dọn khi reconnect. |
| `socketsLeave` chưa hoàn tất trên node khác | Nội dung nhóm phát theo danh sách người nhận đã xác thực; test hai node và thêm cơ chế thứ tự nếu chưa đạt tiêu chí. |
| Query bị hủy nhưng request mạng vẫn trả kết quả | Truyền `AbortSignal` và dùng generation guard cho các cập nhật cache thủ công. |
| Tệp `File`/object URL bị mất khi chuyển phòng | Outbox theo conversation đặt ngoài component keyed; dọn tài nguyên đúng lúc gửi thành công hoặc bỏ. |
| Refetch timeline ký lại nhiều URL media | Ưu tiên event cá nhân hóa và cập nhật đúng tin; chỉ refetch khi cần đối chiếu. |

## 10. Tài liệu kỹ thuật tham chiếu

- [Socket.IO Rooms](https://socket.io/docs/v4/rooms/) và [Redis adapter](https://socket.io/docs/v4/redis-adapter): room server-side, phân phối sự kiện nhiều node và thao tác room qua adapter.
- [PostgreSQL 16 — Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html): snapshot của từng lệnh dưới `READ COMMITTED`; khóa hàng khi cần đồng bộ quyền đọc với cập nhật membership.
- [TanStack Query — Query Cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation) và [Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates): hủy query có `AbortSignal`, snapshot/rollback và retry state.
