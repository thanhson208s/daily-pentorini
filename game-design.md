# Daily Pentorini — Game design

Pet project giải đố solo, public miễn phí; học Phaser, TypeScript, Vercel và backend serverless. Ưu tiên game hoàn chỉnh, dễ chia sẻ, chạy trong free tier.

- [Kiến trúc](docs/architecture.md)
- [Lộ trình](docs/roadmap.md)

## 1. Luật chơi

Xếp đủ 12 pentomino khác nhau (tham khảo https://en.wikipedia.org/wiki/Pentomino), mỗi khối gồm 5 ô và dùng đúng một lần, lên bàn 7 × 9:

```text
Mon | Tue | Wed | Thu | Fri | Sat | Sun
Jan | Feb | Mar | Apr | May | Jun | Jul
Aug | Sep | Oct | Nov | Dec | 25  | 26
27  | 28  | 29  | 30  | 1   | 2   | 3
4   | 5   | 6   | 7   | 8   | 9   | 10
11  | 12  | 13  | 14  | 15  | 16  | 17
18  | 19  | 20  | 21  | 22  | 23  | 24
25  | 26  | 27  | 28  | 29  | 30  | 31
月  | 火  | 水  | 木  | 金  | 土  | 日
```

- Cho phép xoay theo bước 90° và lật; không chồng khối hoặc vượt biên.
- 12 khối phủ 60/63 ô. Thắng khi 3 ô còn lại gồm đúng một tháng, một ngày và một thứ của ngày mục tiêu.
- Hai hàng thứ có cùng ý nghĩa: `Mon = 月`, …, `Sun = 日`, có thể thay hàng cuối thành hình ảnh thiên thể tương ứng (mặt trăng, mặt trời, sao Kim/Thùy/Mộc/Hỏa/Thổ).
- Các ngày 25–30 và các thứ có hai vị trí. Được để hở **một trong các vị trí tương ứng**; vị trí còn lại phải được phủ.
- Không cố định trước ba vị trí để hở: mọi lời giải thỏa điều kiện trên đều hợp lệ.
- Cùng bộ tháng–ngày–thứ sẽ có cùng câu đố, kể cả ở năm khác. Không thêm random chỉ để tránh lặp.
- Chỉ nhận ngày lịch hợp lệ, có xử lý năm nhuận.

**Cần kiểm chứng:** diện tích khớp không đảm bảo có lời giải. Viết solver chạy cục bộ, kiểm tra mọi tổ hợp tháng–ngày–thứ có thể xuất hiện trước khi public; lưu ít nhất một lời giải cho mỗi tổ hợp làm dữ liệu kiểm chứng. Nếu có ngày vô nghiệm, phải giải quyết bàn/luật trước khi phát hành.

## 2. Chế độ chơi & thời gian

### Daily

- Không yêu cầu đăng nhập. Người chơi bấm bắt đầu để mở lượt và tính giờ.
- Ngay khi vào web, thấy số lượt hoàn thành Daily hôm nay và **best time**, không cần bắt đầu chơi. Mỗi cookie chỉ được cộng một kết quả/ngày; số hoàn thành không bảo đảm là số người thật duy nhất. Client không gửi kết quả bỏ cuộc/luyện tập.
- Chưa có ai hoàn thành: hiện `0 lượt hoàn thành` và best time `∞`. Đang tải hoặc lỗi thống kê hiển thị `-`, cache 1-5 phút.
- Dùng ngày **UTC**, đổi thử thách lúc 00:00 UTC (07:00 Việt Nam); hiển thị ngày thử thách và đếm ngược.
- Đây là lựa chọn thay cho reset theo múi giờ từng người, để có một cửa sổ xếp hạng chung. Ngày Daily có thể khác ngày địa phương.
- Ván đang chơi qua mốc đổi ngày vẫn được giữ để hoàn thành, nhưng không còn được ghi hạng Daily. Thời gian đếm ngược ở client giữ ở mốc `00:00:00`, tải lại trang sẽ mở puzzle của ngày mới.
- Timer tính thời gian thực đã trôi qua kể từ khi ấn bắt đầu, kể cả khi rời tab hoặc tải lại trang.
- Thời điểm bắt đầu, tiến độ và thời gian hoàn thành do client tính/lưu cục bộ. Bắt đầu/tiếp tục không ghi server; nộp kết quả hoặc bỏ cuộc Daily ghi trạng thái cuối vào cùng bảng ngắn hạn theo cookie/ngày.

### Luyện tập / Archive

- Chơi lại Daily hoặc chọn ngày trong 10 tuần gần nhất; không ghi thêm vào leaderboard.
- Ngày đã kết thúc chỉ được chơi luyện tập. Tự hoàn thành lần đầu sau ngày mục tiêu được đánh dấu xanh dương cục bộ, không tính vào streak Daily.
- Vẫn tính giờ, lưu thành tích cá nhân và cho chia sẻ.
- Khi có backend: so sánh thời gian với histogram lịch sử cùng ngày để hiển thị **nhóm percentile tham khảo**, không ghi hạng luyện tập kể cả nhanh hơn top 10.
- Có chức năng chọn ngày tùy ý ngoài phạm vi 10 tuần nhưng sẽ không có dữ liệu xếp hạng, chỉ hiển thị thành tích cá nhân.

### Bỏ cuộc & xem đáp án

- Ván chưa hoàn thành có nút **Bỏ cuộc / Xem đáp án**; xác nhận rõ việc mất quyền ghi BXH cho ngày mục tiêu trước khi thực hiện.
- Khi xác nhận bỏ cuộc Daily: client khóa thao tác và gửi yêu cầu; chỉ sau khi server xác nhận `given_up` mới kết thúc lượt, dừng timer, cập nhật heatmap/streak và mở đáp án hoàn chỉnh. Không cần khớp với các khối đang đặt.
- Server lưu bỏ cuộc trong cùng bảng với dấu đã nộp, chặn cookie đó gửi điểm ngày hôm nay kể cả refresh, xóa local storage hoặc chơi lại. Xem đáp án Daily qua luyện tập cũng phải đi qua bước này nếu chưa hoàn thành.
- Request lỗi/mất phản hồi: lưu trạng thái chờ cục bộ, chưa mở đáp án; cho retry cùng cookie/ngày. Nếu nộp điểm và bỏ cuộc đồng thời, trạng thái ghi thành công trước được giữ: đã nộp thì không chuyển bỏ cuộc, đã bỏ cuộc thì không được nộp.
- Bỏ cuộc Archive hoặc bản chưa có backend chỉ lưu cục bộ. Nếu ngày mục tiêu đã đóng trong lúc chờ, không ghi trạng thái server cho ngày cũ nữa; có thể tiếp tục bỏ cuộc/xem đáp án dưới dạng luyện tập.
- Ngày đã tự hoàn thành trước đó chỉ có **Xem đáp án / Chơi lại**, không chuyển thành bỏ cuộc hoặc xóa thành tích đã ghi nhận.
- Đáp án lấy từ dữ liệu tĩnh đã tính và kiểm chứng trước; không chạy solver theo từng request. Đây là tính năng trong MVP, tách biệt với hint từng bước để sau.

## 3. Điều khiển & UX

| Thành phần | Hành vi |
| --- | --- |
| Desktop | Kéo thả; phím xoay/lật, di chuyển khối và xác nhận đặt; có chú thích phím |
| Mobile | Kéo thả hoặc chọn khối rồi chạm để đặt; nút xoay/lật trên màn hình |
| Đặt khối | Bóng xem trước, phân biệt hợp lệ/không hợp lệ; đặt sai không làm mất trạng thái cũ |
| Chỉnh sửa | Undo, lấy khối khỏi bàn, trả về khay, làm lại bàn |
| Tiến độ | Tự lưu sau thay đổi, khôi phục khi tải lại; làm lại bàn trong lượt Daily không reset timer |
| Hướng dẫn | Ngắn, có hình minh họa ba ô phải để hở và cách xoay/lật |
| Kết thúc | Tự phát hiện thắng, dừng timer, mở kết quả và hành động chia sẻ/chơi lại |

Prototype mobile sớm: bàn 7 × 9, khay 12 khối và nút điều khiển phải dùng được trên màn hình nhỏ. Có thể phóng lớn riêng khối đang chọn; tránh ngón tay che vị trí đặt.

## 4. Thành tích & leaderboard

- Thi đua vui, tin thời gian do client cung cấp. Server giữ tạm hash cookie và boolean `is_submitted`: `true` đã nộp, `false` đã bỏ cuộc; chưa có bản ghi nghĩa là chưa kết thúc lượt ở server. Không lưu cookie thô, tài khoản hoặc lịch sử bắt đầu; tên và thời gian chính xác chỉ lưu trong top 10.
- Client chỉ gửi lần tự hoàn thành Daily đầu tiên trong ngày, chưa bỏ cuộc/xem đáp án. Refresh tiếp tục trạng thái đã lưu; chơi lại là luyện tập. Server bảo đảm cùng cookie không được cộng thêm lần nữa trong ngày, kể cả gửi đồng thời từ nhiều tab.
- Server nhận thời gian và lời giải để kiểm tra payload, ngày Daily hiện tại và cách xếp. Xác thực lời giải không chứng minh thời gian là thật hoặc chưa xem đáp án.
- Mỗi ngày server giữ **một bản tổng hợp**: số hoàn thành, histogram thời gian có số khoảng cố định và tối đa 10 kết quả nhanh nhất. Ngoài top 10 không lưu thời gian cá nhân chính xác hoặc lời giải; không có danh sách cookie ẩn trong record.
- Bảng Neon ngắn hạn chứa ngày, hash cookie 128 bit và trạng thái đã nộp/bỏ cuộc; tối đa một dòng cho mỗi cookie/ngày. Bỏ cuộc không tăng count/histogram hoặc thêm vào top. Sang ngày UTC mới, cookie được chơi/nộp lại; trạng thái ngày cũ được dọn sau đó. Phần tạm tăng theo số cookie đã nộp hoặc bỏ cuộc, không tích lũy lịch sử cá nhân lâu dài.
- Ngoài top 10 người, hiện nhóm **top 1%, 2%, 3%, 4%, 5%, 10%, 15%, 20%, 25%, 50%** khi có ít nhất 50 kết quả. Chọn mốc nhỏ nhất có thể bảo đảm từ histogram; không hứa thứ hạng chính xác. Nếu chưa đủ mẫu hoặc không thuộc top 50%, chỉ hiện thời gian cá nhân và lời chúc hoàn thành, thêm lời nhắc user quay lại để kiểm tra xếp hạng sau.
- Top 10 giữ thời gian chính xác, username và mã kết quả để client nhận diện kết quả của mình. Xếp tăng dần theo thời gian; bằng thời gian thì kết quả được ghi nhận trước đứng trước, luôn tối đa 10 mục. Best time lấy từ mục đầu.
- Hiển thị `Hạng 3/100 lượt ghi nhận` chỉ khi kết quả còn trong top 10; rời top thì chuyển sang nhóm percentile. Luôn ghi rõ thành tích trong ngày là tạm thời.
- Hết cửa sổ Daily, server không nhận thêm kết quả ngày đó. Archive chỉ đọc thống kê đã chốt.
- Mất phản hồi sau khi gửi: giữ nguyên cookie và payload, cho thử lại khi Daily còn mở. Server trả đã ghi nhận nếu cookie đã nộp, hoặc từ chối nếu đã bỏ cuộc; không tăng count/histogram hay ghi đè top. Qua ngày chỉ giữ kết quả cục bộ, không nộp bù cho ngày đã đóng.
- Kiểm tra cookie không chống được sửa thời gian, xóa/đổi cookie hoặc dùng trình duyệt khác; không hứa mỗi người thật chỉ đóng góp một kết quả.
- Không cần hồ sơ công khai, cập nhật realtime hoặc đăng nhập trong MVP.

### Username cho top 10

- Sau khi server xác nhận kết quả Daily lọt top 10, nếu chưa có username đã lưu thì mở hộp thoại nhập tên để hiển thị trên BXH. Ngoài top 10 và luyện tập không hỏi tên.
- Tên là tên hiển thị công khai, không phải tài khoản; được trùng tên. Cho phép 1–24 ký tự sau khi bỏ khoảng trắng đầu/cuối, gồm chữ Unicode, số, và dấu cách.
- Có thể bỏ qua; kết quả vẫn được giữ và hiện tên mặc định **Ẩn danh**. Nhập tên không nộp lại điểm, không tăng số hoàn thành hoặc thay đổi thời gian.
- Quyền đặt tên được kiểm tra bằng cookie: server tính `player_hash` và đối chiếu chủ kết quả top 10. Hash chỉ lưu nội bộ, không công khai trên BXH; không cần mã sửa tên riêng.
- Lưu username hợp lệ vào local storage; những lần hoàn thành sau tự dùng tên đó nếu lọt top 10, không hỏi lại. Có thể sửa tên đã lưu trong cài đặt để dùng cho các kết quả tiếp theo; không đổi tên lịch sử.
- Nếu kết quả đã rời top 10 hoặc Daily đã kết thúc khi lưu tên, không ghi tên lên BXH nữa; vẫn giữ tên cục bộ cho lần sau. Bỏ qua/lỗi lưu tên không làm mất kết quả.
- Username chỉ được lưu ở server trong tối đa 10 mục hiện có; khi kết quả rời top thì tên đi kèm cũng bị loại. Xóa dữ liệu trình duyệt sẽ mất tên tự điền.

## 5. Chia sẻ & dữ liệu cá nhân

- Thẻ kết quả: ngày thử thách, thời gian, Daily/luyện tập, nhóm percentile hoặc hạng top 10 nếu có, đường dẫn game.
- Không hiển thị cách xếp để tránh lộ đáp án; hạng trong ngày ghi rõ là tại thời điểm chia sẻ.
- Tạo ảnh ngay trên trình duyệt; hỗ trợ chia sẻ khi thiết bị cho phép, có tải ảnh/copy nội dung dự phòng.
- Tiến độ, tùy chọn và lịch sử cá nhân lưu trên thiết bị; không đồng bộ nhiều thiết bị. Xóa dữ liệu trình duyệt có thể làm mất chúng.
- Backend lỗi: vẫn chơi và lưu kết quả cục bộ; không hứa ghi hạng hồi tố cho lượt không được server xác thực đúng hạn.

## 6. Lịch heatmap — lưu cục bộ

Lịch 10 tuần gần nhất để biết đã chơi thử thách của ngày nào; khóa theo **ngày mục tiêu UTC**, không phải ngày thực tế mở web/chơi bù.

| Màu | Ý nghĩa |
| --- | --- |
| Nền (empty) | Hôm nay chưa chơi hoặc ngày tương lai; ngày tương lai chưa được mở |
| Xám | Ngày đã kết thúc mà chưa có bản ghi chơi (`played/completed/given_up/practice_completed`) |
| Vàng | Đã bấm Bắt đầu nhưng chưa hoàn thành, chưa bỏ cuộc |
| Xanh lá | Đã tự hoàn thành Daily trong đúng ngày mục tiêu UTC |
| Xanh dương | Đã tự hoàn thành ngày cũ khi luyện tập, chưa từng bỏ cuộc/xem đáp án ngày đó |
| Đỏ | Đã bỏ cuộc/xem đáp án trước khi tự hoàn thành |

- Ngày hiện tại có thêm viền để dễ nhận biết
- Nhãn/tooltip phân biệt chưa chơi, bỏ lỡ, đang chơi, hoàn thành Daily, hoàn thành luyện tập và bỏ cuộc; không chỉ truyền trạng thái bằng màu.
- Chỉ mở web/xem thống kê không tạo bản ghi chơi. Qua ngày UTC, ô trống chưa chơi chuyển xám; ngày đang dở vẫn vàng, không tự chuyển thành bỏ cuộc.
- Bắt đầu từ trống/xám chuyển vàng. Tự hoàn thành trong đúng ngày Daily chuyển xanh lá; hoàn thành sau khi ngày mục tiêu kết thúc chuyển xanh dương, kể cả ván bắt đầu từ hôm trước. Bỏ cuộc chuyển đỏ.
- Ngày đã xanh lá/xanh dương/đỏ giữ nguyên màu khi chơi lại hoặc xem đáp án; luyện tập không nâng xanh dương thành xanh lá.
- Lưu trong local storage, giữ khi refresh/qua ngày; không tự xóa lịch sử chỉ vì vượt cửa sổ Archive. Không cần API hay đồng bộ tài khoản.
- Heatmap là lịch sử trên thiết bị, độc lập việc kết quả có được BXH chấp nhận hay không. Xóa local storage có thể mất lịch; nếu cookie còn nguyên, server vẫn chặn nộp lại hoặc nộp sau bỏ cuộc trong ngày. Khi API trả `given_up`, đồng bộ ô ngày đó thành đỏ và tính lại streak; bảng tạm không khôi phục tiến độ hoặc lịch sử cá nhân đầy đủ.

## 7. Streak — lưu cục bộ

- Hiển thị **Streak: N ngày** trên màn hình chính, gần ngày thử thách; cập nhật khi hoàn thành, bỏ cuộc, đổi ngày UTC hoặc mở lại web.
- Streak là số ngày lịch UTC liên tiếp đã tự hoàn thành Daily đúng ngày (ô xanh lá). Mỗi ngày chỉ tính một lần; hoàn thành luyện tập (xanh dương), bỏ cuộc hoặc xem đáp án không cộng và không nối lại chuỗi.
- Nếu hôm nay đã hoàn thành, đếm lùi từ hôm nay. Nếu hôm nay chưa hoàn thành và chưa bỏ cuộc, giữ chuỗi kết thúc hôm qua để còn cơ hội nối tiếp trong ngày.
- Khi hết một ngày mà chưa hoàn thành, chuỗi bị ngắt; bỏ cuộc Daily hôm nay làm streak hiện tại về 0 ngay. Hoàn thành Daily tiếp theo bắt đầu chuỗi mới từ 1.
- Ví dụ: hoàn thành ngày 10 và 11 → streak 2; ngày 12 đang chơi vẫn 2, hoàn thành → 3. Nếu bỏ lỡ ngày 12, sang ngày 13 → 0; chơi bù ngày 12 chỉ đổi màu xanh dương, không khôi phục streak.
- Lưu lịch sử hoàn thành và streak trên thiết bị, không cần backend; tự hoàn thành đúng ngày vẫn được tính khi API BXH lỗi. Refresh/chơi lại không cộng trùng; xóa dữ liệu trình duyệt có thể mất streak.

## 8. Đa ngôn ngữ & theme

### Ngôn ngữ

- MVP hỗ trợ **Tiếng Việt (`vi`), English (`en`) và 中文(`zh`)*; cấu trúc cho phép bổ sung ngôn ngữ sau.
- Có bộ chọn ngôn ngữ trong cài đặt. Ưu tiên lựa chọn đã lưu → ngôn ngữ trình duyệt được hỗ trợ → English.
- Dịch giao diện, hướng dẫn, thông báo/lỗi, xác nhận bỏ cuộc, lịch/tooltip, kết quả và nội dung chia sẻ. Ngày/số hiển thị theo locale; ngày thử thách và reset vẫn theo UTC.
- Nhãn tháng/thứ trên bàn dùng bản dịch viết tắt vừa ô. Bàn ở mục 1 là sơ đồ gốc; bản dịch giữ nguyên vị trí, giá trị và cả hai hàng thứ, không gộp các ô trùng.
- Đổi ngôn ngữ ngay trong ván không tải lại game, reset timer hoặc mất tiến độ; tên game Daily Pentorini giữ nguyên.

### Theme

- Có ba lựa chọn **Sáng / Tối / Theo hệ thống**; mặc định theo hệ thống, lựa chọn thủ công được ưu tiên.
- Áp dụng đồng bộ cho giao diện HTML, bàn/khay Phaser, hộp thoại, lịch heatmap và ảnh kết quả tạo mới.
- Đổi theme ngay trong ván không ảnh hưởng gameplay. Khi chọn theo hệ thống, giao diện cập nhật nếu thiết bị đổi chế độ sáng/tối.
- Bảo đảm chữ, đường biên, khối và bóng placement dễ phân biệt ở cả hai theme. Heatmap giữ ý nghĩa trắng/xám/vàng/xanh lá/xanh dương/đỏ, có viền/nhãn phù hợp và không dựa riêng vào màu.
- Ngôn ngữ và lựa chọn theme lưu cục bộ, khôi phục khi mở lại web; không cần tài khoản/API. Nếu không lưu được, vẫn cho đổi trong phiên hiện tại.

## 9. Backlog — khả năng cao sẽ làm sau MVP

- **Ads khi traffic tốt:** đánh giá traffic, doanh thu dự kiến và ảnh hưởng trải nghiệm trước khi tích hợp banner/interstitial; tránh che bàn hoặc ngắt lúc đang giải. Khi bật ads kiếm tiền, đánh giá lại ngân sách và chuyển khỏi Vercel Hobby theo điều kiện sử dụng thương mại.
- **Timeline hành động trong payload submit:** ghi sự kiện bắt đầu, đặt/di chuyển/gỡ khối, xoay/lật, Undo và làm lại bàn, kèm giờ local có offset múi giờ và thời gian đã trôi qua từ lúc bắt đầu. Lưu cục bộ cùng tiến độ, gửi một lần khi nộp kết quả; server replay để kiểm tra chuỗi thao tác, bàn cuối và tính hợp lý của thời gian. Đây là lớp kiểm tra bổ sung vì timeline/client clock vẫn có thể bị giả; giới hạn số sự kiện/kích thước payload và xử lý tạm thời, không lưu lịch sử hành động dài hạn trong DB.

## 10. Ngoài phạm vi MVP

Multiplayer, tài khoản, đồng bộ thiết bị, achievement, hint nâng cao, tạo màn ngẫu nhiên, bảng hạng realtime, lưu ảnh chia sẻ trên server. Hint để sau vì cần solver lúc chơi và quy định riêng cho thành tích có trợ giúp; cân nhắc theo phản hồi người chơi.
