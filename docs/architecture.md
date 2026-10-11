# Kiến trúc

[Game design](../game-design.md) · [Lộ trình](roadmap.md)

## 1. Stack

**TypeScript + Phaser + Vite → Vercel; thêm Neon khi làm leaderboard.**

| Thành phần | Trách nhiệm |
| --- | --- |
| TypeScript thuần | Bàn, bộ khối, phép xoay/lật, kiểm tra placement và điều kiện thắng; dùng chung client/server |
| Phaser | Vẽ bàn, input, kéo thả, animation |
| HTML/CSS | Nút, hướng dẫn, hộp thoại, kết quả; dùng React nếu thấy thuận tiện |
| Local storage | Trạng thái bắt đầu/nộp/bỏ cuộc, timer/thời gian cá nhân, tiến độ, tùy chọn, username, heatmap và streak |
| Vercel CDN | Phục vụ ứng dụng và asset tĩnh |
| Vercel Functions | Nhận kết quả/bỏ cuộc, kiểm tra lời giải, cập nhật/đọc thống kê Daily và top 10; đặt username cho kết quả top |
| Neon Postgres | Một record thống kê/ngày và bảng hash cookie/trạng thái cuối ngắn hạn |
| Upstash Redis — tùy nhu cầu | Rate limit API ghi; thêm cache khi số liệu thực tế cho thấy cần |
| Solver cục bộ | Chứng minh có lời giải, tạo dữ liệu kiểm chứng; không chạy trong request chơi game |

Không cần Next.js chỉ để deploy Vercel. Phaser có [template chính thức](https://docs.phaser.io/phaser/getting-started/project-templates) cho Vite, React và Next.js.

Neon và Upstash tích hợp được qua [Vercel Marketplace](https://vercel.com/docs/marketplace-storage); vẫn là dịch vụ riêng với quota riêng. Ban đầu chỉ cần Neon, không lưu leaderboard song song trong Redis.

## 2. Phân chia logic

- Mỗi ô có ID vị trí, loại (`month/day/weekday`) và giá trị chuẩn hóa; nhãn tiếng Anh/Nhật chỉ để hiển thị.
- Mỗi pentomino có ID và tọa độ tương đối; kiểm tra hình dạng theo các phép xoay/lật được phép.
- Phaser dùng module luật để phản hồi tức thì; server dùng cùng module để xác thực lại.
- Ngày mục tiêu và thứ được tính từ ngày UTC hợp lệ, không lấy thứ do client tự khai.
- Không dùng physics engine để quyết định placement; đây là bài toán ô lưới.
- Lời giải mẫu được tính trước theo tháng–ngày–thứ, phục vụ như asset tĩnh. Với Daily chưa hoàn thành, UI chỉ mở đáp án sau khi API bỏ cuộc xác nhận; không có endpoint bắt đầu hoặc phiên chơi phía server. Asset tĩnh không ngăn người dùng tự tìm đáp án ngoài luồng UI.
- Heatmap lưu theo `puzzleDate`, trạng thái `played/completed/given_up/practice_completed`: vàng/xanh lá/đỏ/xanh dương tương ứng. Không có bản ghi: ngày quá khứ là xám, hôm nay/tương lai là trống; suy màu từ ngày UTC hiện tại, không cần ghi các ngày bỏ lỡ vào storage.
- Client lưu thời điểm tự hoàn thành: đúng ngày mục tiêu → `completed`, sau ngày mục tiêu → `practice_completed`. Chơi lại không ghi đè trạng thái hoàn thành/bỏ cuộc thành `played`; đã `given_up` thì giữ đỏ dù luyện tập thành công. Heatmap lưu cục bộ, đồng bộ `given_up` khi nhận trạng thái server; luyện tập ngày cũ không gửi lên thống kê BXH.
- Streak tính từ các ngày `completed` liên tiếp, neo ở hôm nay nếu đã hoàn thành, ngược lại ở hôm qua; nếu hôm nay `given_up` thì bằng 0. Ngày neo không `completed` thì bằng 0; `practice_completed` không nối chuỗi.
- Lưu `currentStreak` cục bộ và tính lại từ lịch sử khi mở web, trở lại tab, đổi ngày UTC, hoàn thành hoặc bỏ cuộc để tránh cộng trùng/streak cũ. Giữ lịch sử ngoài cửa sổ Archive 30 ngày để chuỗi dài không bị cắt. Heatmap/streak không cấp quyền ghi hạng và không cần API bổ sung.

### Ngôn ngữ & theme

- Tách chuỗi UI khỏi logic thành dictionary `vi/en/zh` có cùng bộ khóa, fallback English khi thiếu bản dịch. DOM và Phaser dùng chung nguồn bản dịch; API trả mã lỗi để client dịch.
- Locale chỉ thay nhãn/định dạng, không đổi ID ô, `puzzleDate` hay dữ liệu xếp hạng. Định dạng ngày Daily với múi giờ UTC tường minh; cập nhật thuộc tính `lang` của trang.
- Lưu `locale` khi người dùng chọn và `themePreference: light/dark/system` trong settings cục bộ. Tách lựa chọn theme khỏi theme thực tế đang hiển thị để giữ chế độ theo hệ thống.
- Dùng bảng màu chung cho CSS và Phaser, gồm nền, chữ, lưới, khối, placement và trạng thái heatmap; tránh gắn màu cố định riêng từng nơi.
- Khởi tạo theme trước lần vẽ đầu để hạn chế nháy sai màu. Thay locale/theme chỉ cập nhật text/style, không tạo lại ván; ảnh chia sẻ dùng locale/theme tại lúc tạo.
- Bản dịch và bảng màu là asset/code tĩnh; không cần dịch vụ dịch, database hay request backend bổ sung. Nếu local storage không khả dụng, giữ settings trong bộ nhớ phiên.

## 3. Trạng thái client & thống kê server

### Client

- Local storage theo `puzzleDate`: trạng thái chơi, `startedAt`, `completedAt`, `elapsedMs`, placement, `resultId` và trạng thái gửi (`not_sent/sending/sent/unknown`). Cookie riêng chứa ID ẩn danh ngẫu nhiên 128 bit ổn định, được khởi tạo trước khi gửi điểm hoặc bỏ cuộc và giữ qua refresh/retry/ngày mới; không phải tài khoản.
- Bắt đầu, timer và khôi phục ván ở client. Bỏ cuộc Daily gọi server; lưu hành động chờ để refresh không mở lại luồng nộp điểm trong lúc chưa rõ kết quả. Chỉ mở đáp án/cập nhật đỏ sau xác nhận `given_up`; Archive bỏ cuộc chỉ lưu cục bộ.
- Chỉ gửi lần tự hoàn thành Daily đầu tiên đúng ngày và chưa bỏ cuộc; chặn double-click/phối hợp nhiều tab ở client, khóa duy nhất trong Neon bảo vệ khi request vẫn gửi trùng.
- Đánh dấu `sending` trước khi gửi, `sent` khi nhận xác nhận mới hoặc đã nộp. Nếu server trả `given_up`, khóa gửi và đồng bộ heatmap/streak. Mất phản hồi hoặc mở lại khi còn `sending` → `unknown`, cho retry có giới hạn/backoff khi Daily còn mở; giữ nguyên cookie, `resultId` và payload. Qua ngày không retry ghi ngày cũ.
- Tất cả thời gian cá nhân vẫn lưu ở client. Gửi `elapsedMs` để server xử lý tạm thời; chỉ lưu chính xác trên server nếu kết quả lọt top 10.
- Username lưu riêng trong settings, dùng lại giữa các ngày. Chỉ mở prompt khi phản hồi xác nhận vào top 10 và chưa có tên; bỏ qua thì hiển thị tên mặc định. Sửa settings chỉ ảnh hưởng kết quả tương lai.
- Dùng cùng cookie đã nộp kết quả để đặt username; không tạo/lưu mã sửa tên riêng. Mất hoặc đổi cookie thì không sửa tên kết quả cũ được.

### Luồng API

1. **Đọc:** trả thống kê ngày mục tiêu: `completedCount`, histogram, top 10, `bestTimeMs`, `updatedAt`. Ngày chưa có record trả số hoàn thành `0`, best time `null`; không tạo record chỉ vì có lượt xem.
2. **Nộp:** client gửi cookie ổn định cùng `puzzleDate`, mã kết quả ngẫu nhiên `resultId`, `elapsedMs`, placement và username đã lưu nếu có. Bảng ngắn hạn chỉ giữ hash cookie và trạng thái cuối; khi lọt top, lưu cùng `player_hash` do server tính để xác định chủ kết quả.
3. **Kiểm tra:** server chỉ nhận ngày Daily hiện tại theo UTC; kiểm tra cookie đúng định dạng, kích thước payload, thời gian là số nguyên dương dưới 24 giờ, ID và lời giải đầy đủ/hợp lệ. Thiếu/sai cookie thì từ chối trước khi ghi, không cấp ID mới âm thầm trên mỗi retry. Trạng thái bỏ cuộc được kiểm tra trong transaction; thời gian thực vẫn do client khai.
4. **Cập nhật:** trong cùng một transaction, kiểm tra lại cửa sổ ngày và INSERT `(puzzle_date, player_hash, is_submitted = true)` với `ON CONFLICT DO NOTHING RETURNING`. Chỉ khi INSERT thành công mới khóa/tạo thống kê, tăng count/bucket và cập nhật top 10. Nếu xung đột, đọc boolean đã có: `true` trả đã nộp, `false` từ chối do bỏ cuộc; không sửa giá trị cũ. Lỗi bất kỳ bước nào rollback cả dấu lẫn thống kê.
5. **Phản hồi:** sau commit, trả `accepted`, `already_submitted` hoặc `given_up`; chỉ trả hạng của kết quả khớp mã trong top. Kết quả bằng thời gian xếp sau mục đã có. Phản hồi trùng không chứng nhận thời gian trong payload retry là kết quả ban đầu. Không lưu payload/lời giải/thời gian cá nhân ngoài top hoặc cookie thô vào DB/log.
6. **Đặt tên:** nếu lọt top nhưng chưa có username, client mở prompt, lưu tên hợp lệ cục bộ rồi gọi API đặt tên riêng với cookie, ngày, `resultId` và username. Server tính `player_hash` từ cookie, đối chiếu mục top có cùng ngày/`resultId`/`player_hash` và kiểm tra ngày vẫn mở; chỉ cập nhật tên, không cộng histogram/count hoặc sửa thời gian/thứ tự. Gửi lại cùng tên an toàn.
7. **Bỏ cuộc:** nhận cookie/ngày Daily, kiểm tra ngày còn mở và INSERT `is_submitted = false` theo cùng khóa duy nhất. Nếu bản ghi đã là `false`, trả `given_up`; nếu là `true`, giữ kết quả và trả `already_submitted`. Không tạo/sửa `daily_stats`, không tăng count/bucket. Client xử lý trạng thái trả về trước khi mở đáp án; request này không cache công khai.

### Thống kê lâu dài, cố định theo ngày

Một record `daily_stats`, khóa chính `puzzleDate`:

| Trường | Dữ liệu |
| --- | --- |
| `completedCount` | Tổng số cookie đã có kết quả được chấp nhận trong ngày, không phải số người thật duy nhất |
| `timeBuckets` | Mảng bộ đếm có độ dài cố định |
| `top10` | Tối đa 10 mục `{ resultId, elapsedMs, username, player_hash }`, sắp tăng dần; `player_hash` chỉ dùng nội bộ |
| `updatedAt` | Thời điểm cập nhật thống kê |

- Histogram: dưới 10 phút chia mỗi 5 giây (120 bucket); 10–60 phút chia mỗi 30 giây (100); 1–24 giờ chia mỗi 5 phút (276). Tổng **496 bucket**, khoảng 4 KB số đếm/ngày nếu dùng số nguyên 8 byte, chưa tính overhead DB. Khoảng có biên trái đóng, phải mở; số đếm tăng nhưng mảng không dài thêm.
- Ngoài top 10, thời gian chính xác chỉ dùng để chọn bucket rồi bỏ; kết quả bị đẩy khỏi top cũng không giữ riêng. Bảng bên dưới chỉ giữ ngày, hash và trạng thái cuối ngắn hạn; không có bảng lịch sử lượt chơi, Redis session hoặc Bloom filter.
- `username` có thể null để client hiển thị Ẩn danh theo locale. Chuẩn hóa tên, kiểm tra giới hạn 1–24 ký tự/chữ Unicode, số, dấu cách, `._-` ở cả client/server; render như text, không HTML. Username sai không làm mất lượt hoàn thành: nhận kết quả với tên null để nhập lại.
- `player_hash` trong top dùng cùng cách tính với bảng `daily_submissions`; luôn tính từ cookie ở server, không tin hash client gửi. Loại trường này khỏi mọi response; API đọc công khai chỉ trả `resultId`, `elapsedMs`, `username`. `resultId` công khai không đủ quyền sửa tên. Khi cập nhật, khóa record ngày và kiểm tra lại chủ kết quả/còn trong top; không tạo lại mục đã rời top.
- Username và `player_hash` có độ dài giới hạn, được loại cùng kết quả khi rời top; thống kê vẫn chỉ một record/ngày, không có bảng người dùng. Không lưu username của các kết quả ngoài top.
- Thống kê ngày cũ giữ nguyên để so sánh Archive. Dữ liệu thống kê lâu dài tăng theo số ngày; riêng bảng trạng thái tạm tăng theo số cookie đã nộp hoặc bỏ cuộc trong cửa sổ giữ dữ liệu. Ghi/update, compute và overhead vận hành vẫn tăng theo traffic.
- Các mốc percentile: `[1, 2, 3, 4, 5, 10, 15, 20, 25, 50]`, chỉ hiện khi ít nhất 50 kết quả; ưu tiên hạng cụ thể nếu thuộc top 10 người. Với mẫu gồm kết quả đang xét, gọi `F` là số ở bucket nhanh hơn, `B` là số trong bucket hiện tại, `N` là tổng: hạng nằm trong `[F + 1, F + B]`. Chọn mốc nhỏ nhất `p` thỏa `100 × (F + B) / N <= p`, không làm tròn trước khi so sánh. Nếu không có mốc phù hợp thì không gắn nhãn top; không nội suy thành hạng chính xác.
- Với luyện tập hoặc kết quả chưa xác nhận đã được đếm, so sánh tham khảo bằng cách thêm một mẫu tạm vào histogram ở client; không ghi server. Nếu không chắc mẫu đã được đếm, chỉ ghi percentile ước lượng.
- Client nhận diện top bằng `resultId` đã lưu cục bộ. Khi mã không còn trong top, chỉ hiển thị nhóm percentile; không suy ra hạng chính xác từ thời gian bằng nhau.
- Cùng cookie/ngày chỉ được cộng một lần và không được nộp sau `given_up`, kể cả đổi `resultId` hoặc gửi đồng thời. Đổi/xóa cookie vẫn có thể tạo lượt khác; server vẫn tin thời gian client. Không bảo đảm chống gian lận theo người thật.
- Giới hạn payload/tần suất API để bảo vệ quota, không dùng rate limit như bằng chứng mỗi người chỉ nộp một lần. Không lưu trạng thái theo cookie trong Upstash; nếu cần limiter, ưu tiên ngân sách request toàn cục theo cửa sổ thời gian.
- Credentials database chỉ có ở server, không đưa vào bundle trình duyệt.

### Trạng thái cuối ngắn hạn trong Neon

```text
daily_submissions
  puzzle_date DATE
  player_hash BYTEA  // đúng 16 byte
  is_submitted BOOLEAN NOT NULL  // true = đã nộp; false = đã bỏ cuộc
  PRIMARY KEY (puzzle_date, player_hash)
```

- Dùng chung `daily_submissions` cho đã nộp và bỏ cuộc, tối đa một dòng/cookie/ngày. Không có bản ghi nghĩa là chưa có hành động cuối được server ghi nhận, không chứng minh chưa bắt đầu chơi. Đây là bảng thường dọn định kỳ, không phải TEMP TABLE theo connection; không lưu thời gian, lời giải, username hay trạng thái bắt đầu.
- Boolean chỉ mô tả lượt đã kết thúc: `true` đã nộp, `false` đã bỏ cuộc; chưa có dòng là chưa kết thúc, không dùng `false` để biểu diễn chưa chơi. Không có giá trị mặc định, mỗi INSERT phải truyền tường minh. Hai giá trị không được chuyển qua lại. Submit/bỏ cuộc đồng thời tranh cùng unique key; transaction ghi thành công trước quyết định kết quả. Không dùng UPSERT ghi đè `is_submitted`; nếu transaction submit rollback thì yêu cầu bỏ cuộc đang chờ có thể INSERT thành công.
- Mã phản hồi API (`accepted/already_submitted/given_up`) và trạng thái heatmap cục bộ giữ ý nghĩa riêng; chỉ cột trạng thái trong bảng server dùng boolean.
- Hash ở server: 16 byte đầu của SHA-256 trên ID ổn định trong cookie, không hash toàn bộ chuỗi cookie chứa trường thay đổi. Lưu nhị phân 128 bit, không chuỗi hex 32 ký tự; không rút xuống 32 bit vì dễ va chạm. Hash chỉ tiết kiệm/chuẩn hóa định danh, không xác thực người chơi.
- Unique constraint và cập nhật thống kê cùng transaction là cơ chế chống trùng; không đọc kiểm tra trước rồi INSERT riêng. PostgreSQL hỗ trợ [ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html) và [transaction](https://www.postgresql.org/docs/current/tutorial-transactions.html).
- Sang ngày UTC mới, khóa ngày mới có hiệu lực ngay mà không chờ cleanup. Dọn dấu ngày đã đóng bằng tác vụ hằng ngày, chừa khoảng đệm cho request đang xử lý; dự trù có lúc giữ dữ liệu hôm nay và hôm qua. Không xóa dấu ngày còn mở hoặc nhận submit cho ngày đã đóng.
- Reset Daily vẫn suy ra từ thời gian, không cần cron tạo puzzle. Tác vụ dọn chỉ thu hồi dữ liệu cũ; chạy chậm không làm sai quyền nộp ngày mới, nhưng cần theo dõi để tránh tích lũy lâu dài.
- Hash + boolean: 17 byte dữ liệu thô/cookie, tương ứng 17 KB/170 KB/1,7 MB cho 1.000/10.000/100.000 cookie đã nộp hoặc bỏ cuộc; chưa gồm ngày, dòng, index, alignment và dữ liệu vận hành. Đổi sang boolean không bảo đảm giảm tổng kích thước dòng do alignment. Đo dung lượng bảng/index thực tế; DELETE không làm file DB nhỏ ngay, [vacuum](https://www.postgresql.org/docs/current/routine-vacuuming.html) giúp tái sử dụng vùng trống.
- Chưa cần Upstash cho chống trùng: Neon giữ dấu và thống kê trong cùng transaction, tránh lỗi chỉ ghi thành công một bên khi tách Redis/Neon. Bloom filter tiết kiệm hơn nhưng có thể chặn nhầm người chưa nộp, chưa phù hợp MVP.

## 4. Tiết kiệm tài nguyên

- Asset tĩnh, gameplay cục bộ; request nghiệp vụ gồm đọc thống kê, nộp kết quả hoặc bỏ cuộc Daily, đặt tên riêng nếu lần đầu lọt top 10. Tên đã lưu gửi cùng kết quả lần sau; bắt đầu/timer không gọi backend, lời giải mẫu tải như asset tĩnh.
- Không polling leaderboard/thống kê; thống kê tải khi vào web/đổi ngày, bảng tải khi mở, cập nhật sau hoàn thành. Cache dữ liệu công khai theo ngày mục tiêu khoảng 1–5 phút, không lẫn ngày cũ khi qua 00:00 UTC.
- Cache có thể làm hạng trễ cập nhật; kết quả nộp thành công vẫn phải rõ ràng.
- Không cron để tạo ngày mới: Daily suy ra từ thời gian UTC.
- Không SSR mỗi lượt xem, không chạy solver trong Function, không lưu ảnh chia sẻ lên server.
- Không ghi analytics mỗi lần kéo thả. Theo dõi bundle/asset, CDN requests, transfer, invocations, CPU, DB compute và Redis commands nếu dùng.
- Kiểm tra index/query trước khi thêm Redis; đặt Function và database gần nhau nếu có region phù hợp.

## 5. Free tier

Mốc tham khảo đã đối chiếu **11/10/2026**; xác nhận lại plan trong dashboard khi triển khai. Đây là một số quota chính, không phải toàn bộ giới hạn hay cam kết số người dùng.

| Dịch vụ | Quota đáng chú ý | Nguồn |
| --- | --- | --- |
| Vercel Hobby | 100 GB Fast Data Transfer; 1 triệu CDN requests; 1 triệu Function invocations; 4 CPU-hours | [Hobby](https://vercel.com/docs/plans/hobby) |
| Neon Free | 1 GB storage/project; 100 CU-hours/project/tháng | [Cập nhật Free plan 02/10/2026](https://neon.com/blog/neon-free-plan-1-gb-per-project) |
| Upstash Redis Free | 256 MB dữ liệu; 500.000 commands/tháng | [Pricing](https://upstash.com/pricing/redis) |

- Vercel Hobby dành cho cá nhân, phi thương mại; phù hợp mục tiêu hiện tại. Nếu kiếm tiền cần đánh giá lại plan.
- Neon có thể hết **compute** trước storage: `0,25 CU × 24 giờ × 30 ngày = 180 CU-hours` nếu luôn hoạt động, vượt 100 CU-hours. Hạn chế query/polling giữ DB thức, tận dụng scale-to-zero và đo usage.
- Redis tính commands, không đồng nghĩa số HTTP requests; rate limit/cache cũng tiêu quota.
- Không mặc định nâng lên trả phí khi chạm hạn mức; theo dõi usage và điều chỉnh trong ngân sách miễn phí.
- Backend lỗi/hết quota: tiếp tục gameplay và lưu cục bộ, thông báo không thể ghi/xem hạng; không tự biến kết quả offline thành thành tích hợp lệ.
- Hosting Vercel hết quota có thể ảnh hưởng cả website; fallback cục bộ không bảo đảm truy cập mới khi hosting bị dừng.

### Ước tính quy mô vận hành lâu dài

**Mục tiêu ban đầu: 100–500 người chơi/ngày (DAU); 1.000 DAU là mốc thử nghiệm có điều kiện**, chưa phải cam kết của free tier. DAU là người truy cập/chơi trong ngày, không phải số người online đồng thời. Điểm nghẽn dự kiến là thời gian Neon hoạt động, không phải số dòng lưu kết quả.

Giả định MVP, tháng 30 ngày: mỗi DAU tải tổng 2 MB dữ liệu/ngày, tạo 15 CDN requests và tối đa trung bình 3 Function invocations; mỗi invocation dùng 50 ms CPU. Đây là ngân sách giả định, cần đo sau deploy; chưa tính ads/timeline, bot, retry và traffic tăng đột biến.

| DAU trung bình | Transfer/tháng | CDN requests/tháng | Invocations/tháng | CPU-hours/tháng |
| --- | --- | --- | --- | --- |
| 100 | 6 GB | 45.000 | 9.000 | 0,125 |
| 500 | 30 GB | 225.000 | 45.000 | 0,625 |
| 1.000 | 60 GB | 450.000 | 90.000 | 1,25 |

- **Điều kiện Neon:** ở 0,25 CU, đặt ngân sách tối đa khoảng 10 giờ hoạt động/ngày → 75 CU-hours/tháng, chừa 25% dự phòng. Thời gian hoạt động gồm cả chờ idle trước khi ngủ; compute tăng trên 0,25 CU sẽ giảm số giờ khả dụng. Traffic rải suốt ngày có thể vượt quota ngay cả với vài trăm DAU; cache đọc không loại được các lượt ghi submit/bỏ cuộc. Vì vậy không có một mốc DAU bảo đảm miễn phí chỉ từ số user.
- **Storage dài hạn:** histogram khoảng 4 KB/ngày → 1,5 MB/năm dữ liệu số thô, cộng top 10 và overhead. Ở 1.000 DAU, bảng trạng thái tạm giữ tối đa khoảng 2.000 dòng nếu mỗi người dùng một cookie và giữ hai ngày; cleanup/vacuum đều đặn giúp tránh tích lũy theo năm. Đo cả bảng/index thực tế, không lấy 17 byte/hash + boolean làm kích thước cả dòng.
- **Upstash chưa bắt buộc.** Nếu thêm và trung bình dùng 5 commands/DAU/ngày, 1.000 DAU tương ứng 150.000 commands/tháng; phải cộng riêng cache miss, limiter và tác vụ nền.
- **Chốt mức ổn định bằng số đo:** theo dõi ít nhất một chu kỳ tháng; giữ các quota chính dưới khoảng 70–75%, kiểm tra thêm origin transfer, Function memory và DB transfer. Với giả định trên, Vercel còn dư ở 1.000 DAU, nhưng chỉ nâng mục tiêu khi Neon cũng đạt ngân sách. Asset tăng từ 2 lên 5 MB/DAU/ngày thì riêng transfer ở 1.000 DAU đã thành 150 GB/tháng. Tải dồn lúc reset ngày cần kiểm thử riêng vì các submit cùng cập nhật một record thống kê.

## 6. Backlog kỹ thuật — khả năng cao sẽ làm

- **Ads:** chọn nhà cung cấp/vị trí sau khi có số liệu traffic; tải script khi cần, đo ảnh hưởng tới tốc độ/input mobile. Bật kiếm tiền phải đánh giá lại hosting và ngân sách vì [Vercel Hobby giới hạn sử dụng phi thương mại](https://vercel.com/docs/limits/fair-use-guidelines).
- **Timeline submit:** sự kiện có số thứ tự, loại hành động, dữ liệu khối/vị trí/hướng, timestamp local dạng ISO 8601 có UTC offset và `elapsedMs` kể từ bắt đầu. Giờ local để đối chiếu; thứ tự dựa vào số thứ tự, thời lượng dựa vào đồng hồ monotonic của client, nối tiếp khi restore thay vì chỉ trừ giờ hệ thống.
- Persist timeline cùng ván và gửi trong một payload submit; ghi các thao tác làm thay đổi bàn, không ghi từng frame/pointer move. Server giới hạn số sự kiện/bytes/chi phí replay, kiểm tra thứ tự, thao tác hợp lệ, bàn cuối khớp placement và thời lượng khớp kết quả. Thời gian bỏ tab vẫn phải phù hợp quy tắc timer hiện tại; ngày Daily luôn là UTC.
- Replay trước transaction ghi thống kê, không lưu timeline dài hạn/log đầy đủ. Client có thể giả cả timeline và thời gian nên đây là tín hiệu bổ sung, không chứng minh người chơi tự giải; kiểm thử restore, đổi giờ/múi giờ và ván dài trước khi dùng để từ chối kết quả.
