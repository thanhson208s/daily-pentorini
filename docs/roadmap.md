# Lộ trình

[Game design](../game-design.md) · [Kiến trúc & ngân sách](architecture.md)

Mỗi mốc phải dùng được trước khi thêm phạm vi mới. Leaderboard làm sau khi trải nghiệm giải đố đã ổn.

## Mốc 1 — Game hoàn chỉnh cục bộ

- Chuẩn hóa bàn, 12 pentomino và module luật độc lập Phaser.
- Solver cục bộ: kiểm tra mọi tổ hợp lịch có thể xuất hiện, giữ lời giải mẫu; xử lý ngày vô nghiệm trước khi public.
- Render bàn/khay, xoay/lật, kéo thả, chọn rồi chạm để đặt; kiểm chứng bố cục mobile sớm.
- Preview placement, Undo, lấy khối về khay, làm lại bàn.
- Hướng dẫn ngắn, nhận diện thắng, timer cá nhân, autosave/restore.
- Tách chuỗi giao diện và bảng màu ngay từ đầu để DOM/Phaser dùng chung, không gắn text/màu rải rác trong logic game.
- Bỏ cuộc có xác nhận, dừng timer và hiển thị đáp án tính trước; lưu trạng thái cục bộ, chơi lại không xóa dấu bỏ cuộc.

**Hoàn thành khi:** có thể tự chơi trọn ván trên desktop và điện thoại, tải lại không mất tiến độ, lời giải hợp lệ được nhận diện đúng.

**Kiểm chứng chính:** phép xoay/lật; chồng/vượt biên; thiếu/trùng khối; nhãn ngày/thứ trùng; đủ 60 ô nhưng sai mục tiêu; lời giải mẫu; ngày nhuận. Solver chỉ chứng minh tồn tại, không cần đếm hết lời giải.

## Mốc 2 — Bản public để giới thiệu

- Daily UTC, đếm ngược, xử lý ván qua ngày.
- Archive 30 ngày và chơi lại luyện tập; lịch sử cá nhân trên thiết bị.
- Heatmap lưu cục bộ: trắng hôm nay chưa chơi/tương lai, xám ngày cũ chưa chơi, vàng đang dở, xanh lá hoàn thành Daily đúng ngày, xanh dương hoàn thành luyện tập ngày cũ, đỏ bỏ cuộc. Chơi bù cập nhật đúng ngày mục tiêu, không ghi BXH; giữ lịch sử ngoài cửa sổ Archive.
- Hiển thị và lưu streak cục bộ: số ngày liên tiếp hoàn thành Daily đúng ngày UTC; chơi bù không nối chuỗi, refresh/chơi lại không cộng trùng.
- Thẻ chia sẻ không lộ lời giải, tải ảnh/copy nội dung dự phòng.
- Hoàn thiện Tiếng Việt/English và Sáng/Tối/Theo hệ thống; lưu tùy chọn cục bộ, áp dụng cả bàn Phaser, heatmap và ảnh chia sẻ.
- Deploy Vercel, kiểm tra production build và usage thực tế.
- Cho một nhóm nhỏ chơi thử để đánh giá độ khó, mức dễ hiểu của hướng dẫn và thao tác mobile; chưa hứa thời lượng giải cố định.

**Hoàn thành khi:** người mới mở link, hiểu luật, chơi, rời trang/quay lại và chia sẻ kết quả mà không cần hướng dẫn trực tiếp. Chưa có leaderboard ở mốc này.

**Kiểm chứng chính:** đổi ngày UTC, restore ván cũ, giới hạn archive, màu heatmap khi bắt đầu/hoàn thành/bỏ cuộc/chơi lại, chơi bù và refresh; màn hình nhỏ, touch/keyboard, chia sẻ dự phòng.

**Heatmap/streak:** trắng → xám khi bỏ lỡ; ngày đang dở giữ vàng; hoàn thành qua nửa đêm hoặc chơi bù → xanh dương, không ghi hạng/nối chuỗi; giữ đỏ sau khi xem đáp án. Kiểm tra giữ streak trong ngày chưa hoàn thành, ngắt chuỗi khi bỏ lỡ/bỏ cuộc, chuỗi qua tháng/năm/năm nhuận và dài hơn 30 ngày, mở lại sau nhiều ngày, API BXH lỗi và xóa storage.

**Ngôn ngữ/theme:** đủ khóa dịch, fallback locale, chữ dài/dấu tiếng Việt không tràn ô trên mobile; đổi giữa ván không reset; khôi phục settings, theo thay đổi theme hệ thống, tương phản hai theme, local storage bị chặn. Đổi locale không làm lệch ngày UTC; ảnh chia sẻ đúng ngôn ngữ/theme đã chọn.

## Mốc 3 — Thi đua trực tuyến

- Neon: một record thống kê/ngày gồm số hoàn thành, histogram 496 bucket và tối đa 10 kết quả; bảng `daily_submissions` ngắn hạn chứa ngày, hash cookie 128 bit và boolean `is_submitted` (`true` đã nộp, `false` bỏ cuộc; chưa có dòng là chưa kết thúc).
- Bắt đầu, tiến độ và thời gian cá nhân vẫn lưu ở client; server giữ trạng thái cuối đã nộp/bỏ cuộc, thời gian chính xác chỉ giữ trong top 10.
- API đọc thống kê, nộp kết quả và bỏ cuộc Daily. Submit INSERT trạng thái và cập nhật count/bucket/top cùng transaction; bỏ cuộc chỉ INSERT trạng thái, không tạo/sửa thống kê. Không có API bắt đầu hoặc server timer.
- Cookie ổn định qua refresh/ngày mới, mỗi cookie chỉ có một trạng thái cuối/ngày. Retry có giới hạn không ghi đè kết quả cũ; bỏ cuộc được server xác nhận trước khi mở đáp án. Submit/bỏ cuộc đồng thời chỉ một trạng thái thắng; đã bỏ cuộc không được gửi điểm, đã nộp không bị đổi thành bỏ cuộc.
- Tác vụ dọn dấu của ngày đã đóng, có khoảng đệm; ngày mới nhận kết quả ngay theo khóa ngày mà không phụ thuộc cleanup. Đo dung lượng bảng/index và chi phí insert/delete; chưa cần Redis/Bloom filter.
- Khi vào web hiện số hoàn thành Daily và best time; có trạng thái chưa ai hoàn thành/đang tải/lỗi, cache theo ngày.
- Nhóm top 1/2/3/4/5/10/15/20/25/50% khi đủ mẫu, chọn mốc nhỏ nhất được bảo đảm từ histogram; ưu tiên hạng cụ thể cho top 10 người. Bằng thời gian thì mục được ghi nhận trước đứng trước. Archive so sánh histogram lịch sử nhưng không gửi điểm.
- Prompt username khi lần đầu lọt top 10; có thể bỏ qua, lưu tên cục bộ để tự dùng lần sau và sửa trong cài đặt. API đặt tên tách khỏi nộp điểm, kiểm tra quyền bằng `player_hash` tính từ cookie và lưu trong mục top; không cần mã sửa tên riêng.
- Cache bảng công khai, giới hạn tần suất API ghi; thêm Upstash nếu cần cho cơ chế đã chọn.
- Fallback khi backend lỗi; đo quota và tối ưu trước khi mở rộng quảng bá.

**Hoàn thành khi:** chỉ payload/lời giải hợp lệ của ngày Daily hiện tại được cộng thống kê; mỗi cookie/ngày tối đa một lần và không được ghi điểm sau bỏ cuộc, kể cả retry/gửi đồng thời. Record thống kê luôn giữ 496 bucket và tối đa 10 mục; trạng thái cuối được dọn, không tích lũy lịch sử cá nhân. Bỏ cuộc không cộng thống kê, luyện tập ngày cũ không ghi server; mọi dịch vụ vẫn dùng free tier.

**Kiểm chứng chính:** cookie thiếu/sai, payload/lời giải sai, thời gian không hợp lệ về định dạng/phạm vi, submit quá hạn; biên bucket, tổng bucket bằng số hoàn thành; nhóm percentile khi ít mẫu/đúng ngưỡng; top 10 hòa thời gian và đẩy mục cũ. Kiểm tra gửi đồng thời cùng cookie, đổi `resultId`, retry sau mất phản hồi và nhiều tab chỉ cộng một lần; lỗi cập nhật thống kê phải rollback cả dấu chống trùng. Kiểm tra cookie cũ nộp ngày mới, cleanup chậm/chạy lại/qua nửa đêm không mở lại ngày cũ, cache, DB lỗi và giới hạn API. Đổi cookie hoặc giả thời gian hợp lệ vẫn có thể vượt quy tắc: giới hạn thiết kế.

**Bỏ cuộc server:** retry bỏ cuộc không thêm dòng/count; submit sau bỏ cuộc bị chặn, bỏ cuộc sau submit không xóa điểm; submit/bỏ cuộc đồng thời và rollback không để lại hai trạng thái. Mất phản hồi/refresh giữ hành động chờ và chưa mở đáp án; trả `given_up` cập nhật đỏ/streak. Xóa local storage nhưng giữ cookie vẫn bị khóa; qua ngày mới được chơi lại, Archive bỏ cuộc chỉ lưu cục bộ.

**Username:** chỉ hỏi khi lọt top và chưa lưu tên; bỏ qua, tên Unicode/trùng tên/rỗng/quá dài, render text, tự dùng lại sau refresh/ngày mới. Lưu tên/retry không tăng count; thiếu/khác cookie hoặc giả `player_hash` trong payload không sửa được tên người khác; mọi response không lộ hash. Kết quả rời top hoặc hết ngày trong lúc nhập không được tạo lại; tên vẫn lưu cục bộ, sửa settings không đổi lịch sử.

## Điểm dừng

Sau mốc 3, project đã bao gồm hình học ô lưới, input đa thiết bị, trạng thái game, serverless, SQL, caching và kiểm soát chi phí. Chỉ mở rộng khi phản hồi người chơi cho thấy nhu cầu; chưa làm multiplayer, tài khoản, đồng bộ, achievement hoặc hint nâng cao.

## Backlog — khả năng cao sẽ làm sau MVP

- **Ads nếu traffic tốt:** đánh giá nhu cầu/doanh thu dự kiến, lựa chọn banner/interstitial, thử vị trí không ảnh hưởng thao tác game; kiểm tra plan Vercel cho sử dụng thương mại và ngân sách mới trước khi bật kiếm tiền.
- **Timeline chống cheat bổ sung:** ghi/lưu cục bộ hành động và giờ local có offset, gửi cùng kết quả; server replay/đối chiếu thứ tự, bàn cuối và thời lượng. Giới hạn payload/CPU, xử lý tạm không tăng dữ liệu cá nhân dài hạn. Kiểm chứng Undo/reset, refresh/restore, rời tab, thay đổi đồng hồ/múi giờ, timeline thiếu/sai và payload quá lớn.
