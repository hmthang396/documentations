# Bộ câu hỏi phỏng vấn hệ thống & fintech

## 1. Thăm dò nền tảng

**Mục tiêu:** xem họ có mental model về những thứ họ dùng hàng ngày không.

- Một request từ browser tới service của bạn đi qua những chặng nào? Chặng nào có thể là điểm nghẽn đầu tiên?
- Bạn dùng DB gì ở dự án gần nhất? Vì sao chọn nó, ai là người quyết định?
- Index giúp gì và làm chậm cái gì? Khi nào thêm index lại là sai lầm?
- Connection pool đặt bao nhiêu là hợp lý và dựa vào cơ sở nào?

## 2. Thiết kế một hệ thống vừa phải

Chọn một bài quen thuộc nhưng bắt họ tự đặt ràng buộc: URL shortener, rate limiter, hệ thống thông báo, hoặc feed đơn giản.

**Điểm quan trọng:** đưa đề mơ hồ và xem họ có hỏi lại không. Senior sẽ hỏi QPS, read/write ratio, số user, yêu cầu latency, có cần realtime không, budget. Junior/mid sẽ vẽ box ngay lập tức.

Sau khi họ vẽ xong, hỏi:

- Con số ước lượng của bạn từ đâu ra? Nếu traffic gấp 10 thì component nào vỡ trước?
- Thành phần nào trong sơ đồ này bạn sẽ bỏ đi nếu team chỉ có 3 người?

> Câu cuối rất hiệu quả — nó tách người thiết kế theo nhu cầu thật với người copy kiến trúc từ blog.

## 3. Trade-offs (đây là phần lõi)

Hỏi dạng "chọn một và giải thích cái giá phải trả":

- Consistency hay availability cho luồng thanh toán? Còn cho luồng đếm lượt xem? Vì sao khác nhau?
- Khi nào bạn chấp nhận eventual consistency, và bạn giải thích điều đó với team product thế nào?
- Sync call giữa 2 service vs message queue — chọn queue thì bạn vừa mua thêm những vấn đề gì? (Muốn nghe: ordering, duplicate, retry, DLQ, khó debug, latency khó đoán)
- Cache: chiến lược invalidate của bạn là gì? Điều gì xảy ra khi cache chết hoàn toàn lúc 9h sáng?
- Monolith đang chạy ổn, khi nào bạn thực sự tách microservices? Tách sai thì hậu quả là gì?
- Normalize hay denormalize? Bạn từng denormalize và sau đó hối hận chưa?

> Tín hiệu senior thật: họ nêu được cái giá của lựa chọn mình ủng hộ, không chỉ ưu điểm.

## 4. Failure & HA

Đây là chỗ lộ rõ nhất ai từng vận hành production.

- Định nghĩa HA của bạn là gì — 99.9% với 99.99% khác nhau bao nhiêu phút/tháng và chi phí chênh ra sao?
- Vẽ sơ đồ vừa rồi, giờ mình giết 1 node DB primary. Chuyện gì xảy ra trong 30 giây tiếp theo? Ai promote replica? Split-brain xử lý thế nào?
- Downstream service trả lời chậm 5 giây thay vì lỗi — hệ thống bạn phản ứng ra sao? (Muốn nghe: timeout, circuit breaker, bulkhead, backpressure — và hiểu vì sao chậm nguy hiểm hơn chết)
- Retry có thể làm sập hệ thống như thế nào? Cách phòng? (thundering herd, retry storm, exponential backoff + jitter)
- Multi-AZ với multi-region: bạn cần cái nào, RPO/RTO mục tiêu bao nhiêu, và ai trả tiền cho nó?
- Idempotency bạn implement ra sao ở tầng API?

## 5. Scale & dữ liệu lớn

- DB một bảng 2 tỷ dòng, query đang chậm. Trình tự bạn xử lý từ rẻ đến đắt là gì? (Muốn nghe: đo trước đã — explain plan, index, query rewrite, read replica, partition, rồi mới shard)
- Chọn shard key thế nào và sai shard key thì sửa ra sao khi đã có dữ liệu?
- Migration schema trên bảng lớn mà không downtime: kể quy trình bạn từng làm.
- Hot partition / hot key xử lý ra sao?
- Backfill 500 triệu bản ghi mà không làm chết production?

## 6. Fintech

### 6.1. Câu mở đầu — lấy "vật liệu thật"

> "Kể cho tôi nghe về hệ thống liên quan tới tiền mà bạn gắn bó lâu nhất. Nó làm gì, quy mô thế nào, và phần nào của nó là do bạn thiết kế?"

Vì sao câu này tốt hơn "bạn hiểu double-entry không":

- Nó không có đáp án đúng, nên ứng viên thả lỏng và nói thật.
- Nó buộc họ tự vạch ra biên giới giữa "tôi làm" và "team tôi làm" — chỗ này rất hay lộ.
- Quan trọng nhất: nó cho bạn vật liệu để đào suốt phần còn lại. Mọi câu hỏi sau sẽ bám vào chính hệ thống của họ, nên họ không thể trả lời bằng kiến thức học thuộc.

Trong lúc nghe, bạn ghi lại 3 thứ:

- Một con số họ nhắc tới (TPS, số giao dịch/ngày, số tiền xử lý).
- Một đối tác ngoài (ngân hàng, cổng thanh toán, ví khác).
- Một quyết định thiết kế họ tự nhận là của mình.

> Ba thứ này là ba cái móc cho ba tầng sau.

Hai câu phụ để kết thúc phần mở đầu:

- "Con số đó bạn biết từ đâu — dashboard nào, hay ai nói lại?" (người vận hành thật luôn biết mình xem ở đâu)
- "Nếu được làm lại phần đó từ đầu, bạn giữ gì và bỏ gì?"

### 6.2. Đi từ thứ họ chắc chắn biết → chỗ họ chưa nghĩ tới

**Nguyên tắc sắp xếp:** bắt đầu ở vùng an toàn của họ, rồi đẩy ra rìa. Không bao giờ mở màn bằng câu khó — người giỏi bị stress sẽ trả lời tệ, và bạn mất luôn dữ liệu.

Móc vào chính hệ thống họ vừa kể:

> "Trong hệ thống bạn vừa nói, một giao dịch nạp tiền 100k đi qua những bước nào? Vẽ giúp tôi, và đánh dấu chỗ nào ghi vào DB."

Rồi lần lượt:

- Số tiền lưu kiểu dữ liệu gì?
- Số dư là cột balance hay tính từ ledger?
- Giao dịch có những trạng thái nào, trạng thái nào không quay lui được?

Đây vẫn là tầng dễ. Mục tiêu không phải loại người, mà là xác định độ sâu để biết khoan tiếp ở đâu. Nếu tới đây đã lúng túng, bạn biết buổi phỏng vấn sẽ kết thúc ở tầng 3 và đừng cố ép.

### 6.3. Hai bài tập lõi — chấm điểm chính ở đây

Đây là phần quyết định. Chỉ hai bài, làm kỹ, đừng tham.

#### Bài 1 — Concurrency (đưa ra lúc phút ~20)

> "User có 100k. Hai request rút 80k gửi đồng thời vào hai pod khác nhau. Viết cho tôi logic đảm bảo không âm số dư."

**Cách hỏi rất quan trọng:** đưa đề rồi im lặng. Đừng gợi ý. Đừng nói "bạn có nghĩ tới lock không". Sự im lặng 20 giây là một phần của bài test.

Cách khoan theo tầng, mỗi tầng chỉ đi tiếp khi tầng trước xong:

1. Họ viết ra gì? (SELECT rồi UPDATE = dừng, đã đủ tín hiệu)
2. "Nếu hai request này rơi vào hai DB replica khác nhau thì sao?"
3. "Giả sử tài khoản đó là ví merchant, có 500 giao dịch/giây đổ vào. Cách của bạn còn chạy không?"
4. "Nếu bạn buộc phải dùng Redis lock, lỗi gì có thể xảy ra?"

> Tầng 3 và 4 là chỗ tách senior thật khỏi senior danh nghĩa.

#### Bài 2 — Timeout với đối tác (phút ~28)

Móc vào đối tác họ đã nhắc ở đầu buổi:

> "Bạn gọi API chuyển tiền sang [ngân hàng họ vừa kể], bị timeout sau 30 giây. Giao dịch này bây giờ ở trạng thái gì, và bước tiếp theo của hệ thống là gì?"

Đây là câu hỏi có giá trị phân loại cao nhất trong toàn bộ fintech, vì nó không thể trả lời đúng bằng lý thuyết — chỉ người từng mất tiền thật mới phản xạ đúng.

Khoan tiếp:

- "Ai đi hỏi lại bank? Bao lâu một lần? Hỏi bằng cái gì?"
- "Trong lúc chờ, user mở app thấy gì?"
- "Nếu 3 ngày sau vẫn không có câu trả lời?"

### 6.4. Đẩy tới điểm họ không biết — và quan sát phản ứng

Đây là bước nhiều interviewer bỏ qua, nhưng nó cho tín hiệu mạnh nhất về level.

Bạn cố tình hỏi tới khi ứng viên hết vốn. Mục đích không phải làm khó, mà xem họ ứng xử thế nào khi chạm giới hạn:

- Người mid thường bịa hoặc nói vòng vo bằng thuật ngữ.
- Người senior nói thẳng "chỗ này tôi chưa làm", rồi lập luận từ nguyên lý ra một hướng hợp lý.
- Người rất mạnh còn hỏi ngược lại bạn: "bên anh đang xử lý case này ra sao?" — và câu hỏi đó thường rất sắc.

Câu để đẩy tới giới hạn, chọn theo hướng họ yếu:

- "Sổ sách lệch 1 đồng so với bank. Sáng mai bạn làm gì đầu tiên?"
- "Bank partner down 2 tiếng — cho user tiếp tục đặt lệnh rồi queue lại, hay chặn? Lập luận cả hai hướng."
- "Compensating transaction cho một lệnh đã thành công ở bank là gì?"

> Cách chấm ở đây quan trọng hơn nội dung câu trả lời. Một người nói "tôi chưa gặp, nhưng tôi sẽ suy luận thế này..." rồi đi tới được 70% đáp án — đó là tín hiệu tốt hơn người trả lời trơn tru 100% một câu đã học thuộc.

### 6.5. Câu chuyện sự cố — kiểm chứng lại toàn bộ

> "Kể cho tôi một lần hệ thống làm sai về tiền. Lệch bao nhiêu, ai phát hiện, mất bao lâu để đóng, và sau đó thay đổi gì?"

Đặt câu này ở cuối, không phải đầu. Lý do: lúc này bạn đã biết họ hiểu gì về mặt kỹ thuật, nên bạn kiểm chứng được câu chuyện có khớp không. Người kể chuyện phóng đại thường lộ ở chỗ chi tiết kỹ thuật trong câu chuyện mâu thuẫn với những gì họ vừa trả lời 20 phút trước.

**Điểm cần nghe:** "ai phát hiện". Khách hàng báo không phải điểm trừ. Điểm trừ là sau sự cố không xây được cảnh báo tự động.

---

## 7. Chủ đề: Đăng nhập & quản lý phiên (login, JWT, refresh token)

Đây là chủ đề thay thế tốt nhất cho fintech, vì nó có đúng những tính chất làm nên một bài phỏng vấn phân loại chính xác:

- **Tính phổ quát tuyệt đối.** Không có backend dev nào chưa từng làm login. Kể cả người làm e-commerce, logistics, SaaS, game hay nội bộ doanh nghiệp. Bạn loại bỏ hoàn toàn biến số may rủi về domain.
- **Ngưỡng vào thấp, trần rất cao.** Ai cũng làm được một cái login chạy được. Nhưng làm đúng thì cần hiểu stateless vs stateful, revocation, race condition, clock skew, multi-device, và bài toán đánh đổi giữa bảo mật với trải nghiệm.
- **Ai cũng tin mình làm đúng.** Đa số hệ thống login ngoài đời đều có lỗ hổng, nhưng không ai phát hiện vì ngày thường nó chạy tốt. Ứng viên sẽ trả lời rất tự tin, và độ sâu lộ ra tự nhiên chứ không phải do bị dồn.
- **Có câu hỏi "sát thủ"** mà tỉ lệ trả lời trọn vẹn rất thấp (Vòng 4).

Cũng như trước: **chỉ dùng một chủ đề này suốt buổi**, khoan sâu dần, không đổi đề.

---

## Vòng 0 — Mở đầu (3 phút)

> **"Hệ thống gần nhất bạn làm, người dùng đăng nhập bằng cách nào? Kể cho tôi cơ chế phiên đăng nhập của nó, và phần nào là do bạn xây."**

Ghi lại ba thứ để làm móc cho cả buổi:
1. **Cơ chế** họ dùng (session cookie / JWT / OAuth / SSO).
2. **Loại client** (web, mobile app, hay cả hai) — rất quan trọng, vì mobile làm bài toán khó hơn hẳn.
3. Họ tự xây hay dùng thư viện / dịch vụ có sẵn (Keycloak, Auth0, Firebase, Spring Security).

Nếu họ dùng thư viện có sẵn, đừng coi là điểm trừ. Hỏi tiếp: *"Thư viện đó xử lý refresh token thế nào, bạn có đọc phần đó không?"* Người giỏi luôn biết bên trong công cụ mình dùng.

---

## Vòng 1 — Vẽ luồng (5–8 phút)

> **"Vẽ cho tôi luồng từ lúc user nhập email/mật khẩu cho tới lúc họ gọi được một API cần xác thực. Đánh dấu giúp tôi: cái gì được sinh ra ở đâu, lưu ở đâu, và ai kiểm tra nó."**

Câu này dễ, mục đích là lấy bản đồ để khoan. Nhưng tín hiệu phân loại xuất hiện ngay:

| Quan sát | Ý nghĩa |
|---|---|
| Chỉ có **một** token, thời hạn dài (7 ngày, 30 ngày), không có refresh | Chưa nghĩ tới thu hồi. Đào ở Vòng 2 |
| Có access token ngắn hạn + refresh token dài hạn | Ổn, đi tiếp |
| Tự phân biệt rõ "token này để làm gì, token kia để làm gì, và vì sao phải tách" | Tín hiệu tốt ngay từ phút thứ 8 |

Hai câu chốt vòng này:
- "Access token của bạn sống bao lâu? **Con số đó dựa trên cơ sở gì?**" (Người hiểu sẽ nói được nó là đánh đổi giữa cửa sổ rủi ro khi token bị lộ và tải lên hệ thống auth. Người học thuộc sẽ nói "15 phút vì mọi người làm thế")
- "Trong token có những thông tin gì? Vì sao bạn để những thứ đó vào?"

---

## Vòng 2 — Ba câu tấn công vào lỗ hổng phổ biến (10 phút)

**2.1 — Lưu token ở đâu:**
> "Frontend web lưu access token ở đâu? Vì sao chọn chỗ đó?"

Rồi im lặng, sau đó đào:
> "Nếu trang của bạn dính XSS thì chuyện gì xảy ra với token đó?"

- **Yếu:** "localStorage cho tiện" và không nêu được hệ quả.
- **Đạt:** biết localStorage dễ bị XSS đọc mất, cookie `HttpOnly` an toàn hơn trước XSS.
- **Senior:** nói được **cả hai chiều của đánh đổi**. Cookie `HttpOnly` chống XSS nhưng mở ra CSRF, nên cần `SameSite` hoặc CSRF token. localStorage miễn nhiễm CSRF nhưng chết với XSS. Và nhấn mạnh: không có lựa chọn nào an toàn nếu ứng dụng đã dính XSS, nên XSS mới là thứ phải chặn trước.

Câu này rất hay vì nó đo trực tiếp khả năng **nêu cái giá của lựa chọn mình ủng hộ**, thay vì chỉ nêu ưu điểm.

**2.2 — Xác thực token:**
> "Server nhận một JWT. Bạn kiểm tra những gì trước khi cho request đi tiếp?"

Checklist đầy đủ (senior thường tự nêu 5/7):
1. Chữ ký hợp lệ
2. **Thuật toán có đúng như mình mong đợi không** (chống tấn công `alg: none` hoặc đổi RS256 thành HS256) — rất ít người nhắc
3. Hạn (`exp`) và thời điểm bắt đầu có hiệu lực (`nbf`)
4. `iss` và `aud` có đúng không
5. Token này có bị thu hồi chưa
6. User còn tồn tại / còn active không
7. Quyền trong token có còn đúng với thực tế không

Đào tiếp: *"Ai giữ khóa ký? Nếu khóa lộ thì bạn xoay khóa thế nào mà không đá văng toàn bộ user đang đăng nhập?"* (muốn nghe: `kid` trong header, chấp nhận nhiều khóa trong giai đoạn chuyển tiếp)

**2.3 — Đăng xuất:**
> "User bấm 'Đăng xuất'. Backend bạn làm gì?"

Đây là chỗ lộ rất nhanh:

- **Yếu:** "xóa token ở phía client là xong."
- **Đạt:** nhận ra JWT là stateless nên **token vẫn còn hiệu lực** cho tới khi hết hạn, dù client đã xóa. Đề xuất blacklist hoặc thu hồi refresh token.
- **Senior:** phân biệt rõ hai việc. Thu hồi refresh token thì dễ vì nó vốn có state ở server. Còn access token thì trong cửa sổ vài phút còn lại **về nguyên tắc không chặn được** nếu không kiểm tra state. Từ đó nói được: chính vì vậy access token mới phải ngắn hạn, và độ dài đó chính là cửa sổ rủi ro bạn chấp nhận.

---

## Vòng 3 — Bài tập concurrency (10 phút) — chấm điểm chính

> **"App mobile của bạn mở màn hình chính, gọi song song 5 API. Access token vừa hết hạn đúng lúc đó. Cả 5 request cùng trả về 401, và client gọi refresh 5 lần gần như đồng thời. Refresh token của bạn có cơ chế xoay vòng (dùng một lần, mỗi lần refresh sinh refresh token mới). Chuyện gì xảy ra?"**

Đưa đề rồi **im lặng**. Đây là bài phân loại mạnh nhất của chủ đề này, vì nó biến một thứ ai cũng nghĩ là đơn giản thành bài toán race condition thật, và nó xảy ra **hàng ngày trong mọi app mobile**.

Điều thực sự xảy ra: request đầu tiên đổi được token mới, bốn request còn lại cầm refresh token đã bị vô hiệu. Nếu hệ thống có cơ chế phát hiện tái sử dụng refresh token, nó sẽ coi đây là dấu hiệu token bị đánh cắp và **đăng xuất toàn bộ phiên của user**. Kết quả: user bị văng ra khỏi app một cách ngẫu nhiên, và đây là loại bug cực kỳ khó tái hiện.

Thang chấm:

| Mức | Biểu hiện |
|---|---|
| **Junior** | Không thấy vấn đề. Nghĩ 5 lần refresh thì ra 5 token, đều dùng được |
| **Mid** | Thấy race condition, đề xuất khóa ở client: chỉ cho một request refresh, bốn request kia xếp hàng chờ rồi dùng chung kết quả |
| **Senior** | Giải quyết cả hai phía. Client gom request (single-flight). Server nới một khoảng ân hạn: refresh token vừa bị dùng vẫn chấp nhận trong vài giây và trả về cùng token mới, thay vì báo tái sử dụng. Nói được vì sao chỉ sửa ở client là chưa đủ (app cũ vẫn ngoài kia, mạng chập chờn vẫn gây retry) |
| **Rất mạnh** | Hỏi ngược: *"vậy làm sao phân biệt được đây là race condition bình thường với trường hợp token bị đánh cắp thật?"* — và tự nêu hướng: gắn refresh token theo family/chuỗi, so sánh device fingerprint, xét khoảng thời gian giữa hai lần dùng |

Câu đào cuối vòng này:
> "User của bạn đăng nhập trên 3 thiết bị. Đăng xuất ở điện thoại **không** được làm văng laptop. Mô hình dữ liệu của bạn trông thế nào?"

(Muốn nghe: phiên gắn theo thiết bị, không phải theo user. Nhiều người thiết kế một cột `refresh_token` duy nhất trên bảng `users`, và đó là lỗi lộ ra ngay ở câu này.)

---

## Vòng 4 — Câu hỏi sát thủ (7 phút)

> **"9 giờ sáng, HR báo vừa sa thải một nhân viên. Yêu cầu: chặn quyền truy cập của người này **ngay lập tức**, không được chờ. Hệ thống của bạn đang dùng JWT stateless. Bạn làm gì?"**

Tỉ lệ trả lời trọn vẹn rất thấp, vì câu này đánh thẳng vào **mâu thuẫn cốt lõi của JWT** mà đa số người dùng nó chưa từng đối diện: stateless nghĩa là không thu hồi được, và đó không phải bug mà là bản chất.

Các mức:

- **Yếu:** "xóa user trong DB là xong." Không nhận ra token vẫn hợp lệ vì server không tra DB mỗi request.
- **Mid:** nhận ra vấn đề, đề xuất blacklist token trong Redis và kiểm tra mỗi request.
- **Senior:** nêu được đánh đổi một cách rõ ràng. Kiểm tra blacklist mỗi request nghĩa là **bạn đã từ bỏ tính stateless** và quay lại mô hình session, chỉ là phức tạp hơn. Vậy nên câu hỏi thật sự là: hệ thống này có thực sự cần JWT không, hay session truyền thống với Redis đơn giản hơn và đủ dùng? Người mạnh sẽ hỏi ngược lại bạn: *"'ngay lập tức' ở đây nghĩa là bao nhiêu giây? Nếu chấp nhận trễ 5 phút thì chỉ cần access token 5 phút là xong, không cần làm gì thêm."*
- **Staff:** thêm hai lớp nữa. Một là kiểm tra chọn lọc thay vì kiểm tra tất cả: chỉ các endpoint nhạy cảm mới tra state, endpoint đọc thông thường thì không, để cân bằng chi phí. Hai là nói về **con người và quy trình**: ai được phép bấm nút thu hồi, hành động đó có audit log không, và đội vận hành có công cụ để làm việc đó lúc 9 giờ sáng hay phải nhờ engineer deploy.

Câu phụ rất hay để đào tiếp:
> "Vậy theo bạn, **khi nào JWT thực sự là lựa chọn đúng**, và khi nào session cookie tốt hơn?"

Đáp án tốt: JWT phát huy khi có nhiều service độc lập cần xác thực mà không muốn gọi về một auth server, hoặc khi bên thứ ba cần xác minh. Còn với một ứng dụng web đơn khối có một backend, session cookie đơn giản hơn, thu hồi được ngay, và hầu như luôn là lựa chọn đúng. Ai trả lời được câu này tức là họ chọn công nghệ theo nhu cầu chứ không theo trào lưu.

---

## Vòng 5 — Mở rộng nếu ứng viên mạnh (5–8 phút)

Chỉ dùng khi họ qua Vòng 4 tốt. Chọn 1–2 câu:

- "Mật khẩu lưu thế nào? Vì sao không dùng SHA-256? Tham số của thuật toán bạn chọn đặt bao nhiêu và dựa vào đâu?"
- "Chống brute-force login: bạn giới hạn theo IP hay theo tài khoản? Mỗi cách bị đánh bại bằng cách nào?" (muốn nghe: theo IP thì thua botnet, theo tài khoản thì mở ra tấn công khóa tài khoản người khác, nên cần kết hợp)
- "Server A và server B lệch đồng hồ 30 giây. Token vừa sinh ở A bị B từ chối vì `nbf`. Bạn xử lý thế nào?"
- "Luồng quên mật khẩu: link reset gửi qua email, bạn thiết kế token đó ra sao và có những rủi ro gì?" (câu nhỏ nhưng lộ nhiều: token dùng một lần, thời hạn ngắn, vô hiệu hóa mọi phiên sau khi đổi mật khẩu, không tiết lộ email có tồn tại hay không)
- "Bạn test luồng hết hạn token thế nào, khi không thể ngồi chờ 15 phút?"

---

## Bảng tổng hợp chấm điểm

| Ứng viên dừng ở đâu | Kết luận |
|---|---|
| Một token dài hạn, đăng xuất bằng cách xóa ở client | Junior |
| Có access + refresh, verify chữ ký, biết blacklist | Mid, làm được feature nhưng cần review |
| Vượt Vòng 3, xử lý cả client lẫn server, hiểu phiên gắn theo thiết bị | **Senior** |
| Vượt Vòng 4: tự nêu rằng blacklist nghĩa là từ bỏ stateless, và hỏi ngược "ngay lập tức là bao nhiêu giây" | **Senior mạnh**, đây là ngưỡng tuyển |
| Nói được về kiểm tra chọn lọc, audit, và công cụ cho đội vận hành | Staff |

---

## Vì sao chủ đề này đo chính xác

Ba lý do:

**Không ai có cớ né.** Khác với fintech hay hệ thống realtime, login là thứ 100% ứng viên backend từng làm. Khi họ trả lời kém, đó là tín hiệu về năng lực chứ không phải về việc họ chưa gặp domain.

**Từ Vòng 3 trở đi không thể trả lời bằng tài liệu.** Docs của thư viện JWT chỉ dạy happy path. Race condition khi refresh và mâu thuẫn stateless/revocation chỉ học được bằng cách từng bị đau, hoặc bằng cách thực sự suy nghĩ kỹ.

**Nó đo đúng thứ bạn cần đo.** Vòng 2 đo khả năng nêu đánh đổi hai chiều. Vòng 3 đo hiểu biết về hệ phân tán. Vòng 4 đo khả năng nhận ra giới hạn bản chất của một công cụ mình đang dùng — thứ phân biệt rõ nhất người chọn công nghệ có chủ đích với người chọn theo thói quen.

---

Nếu chủ đề này vẫn hơi nghiêng về bảo mật so với vị trí bạn đang tuyển, hai lựa chọn thay thế cũng phổ quát không kém mà mình có thể soạn tương tự:

- **Xử lý tác vụ nền / hàng đợi** (gửi email, xuất báo cáo, đồng bộ dữ liệu). Câu sát thủ: "job chạy 10 phút, pod bị kill lúc phút thứ 7, chuyện gì xảy ra khi nó chạy lại?"
- **Upload và xử lý file** (ảnh, video, import Excel). Câu sát thủ: "user upload file 2GB, mạng đứt ở 80%, và ngày mai có 10.000 file mồ côi trong storage."

Bạn muốn mình soạn thêm cái nào không, hay chuyển bộ auth này thành scorecard một trang để cả team chấm thống nhất?