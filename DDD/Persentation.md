**Trang 1**

---

### **Kịch bản Thuyết trình: Trang 1 (Trang bìa & Dẫn nhập)**

**Slide hiển thị:** \* **Tiêu đề chính:** Strategic Domain-Driven Design: Mapping Business Value with Subdomains.

- **Phụ đề:** Khung làm việc cho sự rõ ràng về kiến trúc, phân bổ nguồn lực và lợi thế cạnh tranh.
- **Đối tượng:** Dành cho Software Architects, CTOs và Engineering Strategists.

---

#### **1. Lời chào và Đặt vấn đề (30 - 45 giây)**

"Chào buổi sáng/chiều tất cả mọi người. Hôm nay, chúng ta sẽ không chỉ thảo luận về việc viết code, mà chúng ta sẽ thảo luận về cách **kết nối mã nguồn với giá trị cốt lõi của doanh nghiệp**.

Trong vai trò là những người dẫn dắt kỹ thuật (Architects, CTOs), chắc hẳn các bạn đã từng đối mặt với câu hỏi: _Tại sao hệ thống của chúng ta ngày càng phức tạp? Tại sao việc thay đổi một tính năng nhỏ lại tốn quá nhiều thời gian?_ Câu trả lời thường không nằm ở ngôn ngữ lập trình, mà nằm ở cách chúng ta hiểu và phân chia 'miền tri thức' (Domain) của doanh nghiệp."

#### **2. Giới thiệu chủ đề: DDD Chiến lược (45 - 60 giây)**

"Chủ đề hôm nay là **Strategic Domain-Driven Design** (DDD chiến lược). Khác với các kỹ thuật code chi tiết, DDD chiến lược tập trung vào bức tranh lớn:

- **Mapping Business Value:** Chúng ta sẽ học cách bản đồ hóa các giá trị kinh doanh.
- **Subdomains:** Cách chia nhỏ một bài toán khổng lồ thành những phần có thể quản lý được.

Mục tiêu của buổi hôm nay là giúp các bạn có được sự **rõ ràng về kiến trúc (Architectural Clarity)**, biết cách **phân bổ nguồn lực (Resource Allocation)** một cách thông minh và cuối cùng là tạo ra **lợi thế cạnh tranh (Competitive Advantage)** bền vững cho sản phẩm."

#### **3. Chuyển tiếp (15 giây)**

"Để bắt đầu, chúng ta hãy nhìn vào thực tế của một doanh nghiệp hiện đại. Nó không bao giờ là một khối đơn giản, mà là một hệ sinh thái đầy phức tạp. Chúng ta sẽ cùng tìm hiểu điều này ở trang tiếp theo..."

---

**Trang 2**

---

#### **1. Đặt vấn đề: Sự phức tạp của doanh nghiệp (45 giây)**

"Hãy nhìn vào sơ đồ này. Một doanh nghiệp hiện đại ngày nay không bao giờ là một khối đồng nhất (Monolith). Thực tế, nó là một **hệ sinh thái phức tạp**.

Khi chúng ta nói về việc 'xây dựng một hệ thống thương mại điện tử', chúng ta không chỉ nói về một cái web bán hàng. Chúng ta đang nói về hàng loạt các nghiệp vụ khác nhau: từ Quản lý sản phẩm (Product Management), Vận chuyển (Logistics), Hỗ trợ khách hàng (Support) cho đến Thanh toán (Payments) và Marketing. Mỗi mảng này đều có những quy tắc, quy trình và ưu tiên hoàn toàn khác nhau."

#### **2. Nhận thức sai lầm (30 giây)**

"Sai lầm lớn nhất của các đội ngũ kỹ thuật là đối xử với mọi dòng code đều quan trọng như nhau. Chúng ta thường có xu hướng xây dựng một cơ sở dữ liệu khổng lồ cho tất cả các phòng ban dùng chung, và kết quả là gì? Là một mớ hỗn độn (Big Ball of Mud), nơi mà một thay đổi nhỏ ở phần Thanh toán cũng có thể làm hỏng phần Vận chuyển."

#### **3. Giải pháp: Khái niệm Subdomain (45 giây)**

"Để chinh phục sự phức tạp này, chúng ta cần một chìa khóa: **Subdomains (Phân vùng miền tri thức)**.

Thay vì coi toàn bộ doanh nghiệp là một Problem Space (Không gian vấn đề) khổng lồ, chúng ta chia nhỏ nó thành các năng lực kinh doanh riêng biệt.

- **Subdomain là gì?** Đó là một phần logic của domain lớn, đại diện cho một khả năng cụ thể của doanh nghiệp.
- **Tại sao phải làm vậy?** Để chúng ta có thể tập trung vào từng phần một, hiểu rõ đặc thù của nó và quan trọng nhất là để xác định xem phần nào thực sự mang lại tiền và sự khác biệt cho công ty."

#### **4. Chốt hạ (15 giây)**

"Nói tóm lại: Để xây dựng một hệ thống lớn, đừng cố hiểu tất cả cùng một lúc. Hãy chia để trị. Ở slide tiếp theo, chúng ta sẽ xem cách các Subdomain này giúp chúng ta định hướng chiến lược thực thi như thế nào."

---

**Trang 3**

---

#### **1. Dẫn dắt: "Chia để trị" (30 giây)**

"Sau khi đã xác nhận rằng doanh nghiệp là một hệ sinh thái phức tạp ở trang trước, câu hỏi tiếp theo là: _Chúng ta phải làm gì với sự phức tạp đó?_ Câu trả lời nằm ở khái niệm **Partitioning (Phân vùng)**. Trong DDD, chúng ta coi toàn bộ bài toán kinh doanh là một **Problem Space (Không gian vấn đề)**. Trang này sẽ giải thích tại sao việc chia nhỏ không gian này thành các Subdomain lại là bước đi chiến lược quan trọng nhất."

#### **2. Ba mục tiêu cốt lõi của việc phân chia (60 giây)**

"Việc chia nhỏ này không phải là ngẫu hứng, mà nhằm phục vụ 3 mục tiêu chiến lược:

- **Thứ nhất - Understand Structure (Hiểu cấu trúc):** Nó giúp chúng ta tháo gỡ (deconstruct) mô hình kinh doanh tổng thể. Thay vì nhìn vào một mớ bòng bong, chúng ta nhìn thấy các module nghiệp vụ rõ ràng.
- **Thứ hai - Identify Value (Xác định giá trị):** Đây là điểm mấu chốt. Không phải phần nào trong hệ thống cũng có giá trị như nhau. Việc phân chia giúp chúng ta nhận diện được đâu là phần tạo nên sự khác biệt (Differentiating) và đâu là phần phổ thông (Commodity).
- **Thứ ba - Guide Execution (Định hướng thực thi):** Khi đã có các Subdomain rõ ràng, chúng ta sẽ biết cách tổ chức đội ngũ (Team Organization), thiết lập ranh giới kỹ thuật (Bounded Contexts) và đưa ra chiến lược tích hợp phù hợp."

#### **3. Định nghĩa chuẩn (20 giây)**

"Để tất cả chúng ta cùng chung một ngôn ngữ: **Subdomain là một sự phân chia mang tính logic**. Nó không nhất thiết là sơ đồ tổ chức của công ty, mà là sự phân chia dựa trên năng lực kinh doanh (Business Capability). Một Subdomain đại diện cho một mảng chuyên môn cụ thể mà doanh nghiệp cần có để vận hành thành công."

#### **4. Chuyển tiếp (10 giây)**

"Vậy, sau khi đã chia nhỏ được các Subdomain rồi, chúng ta có nên đầu tư tiền bạc và nhân sự vào tất cả chúng như nhau không? Câu trả lời là KHÔNG, và chúng ta sẽ tìm hiểu tại sao ở trang tiếp theo về phân loại đầu tư."

---

**Trang 4**

---

#### **1. Đặt vấn đề: Nguồn lực là hữu hạn (30 giây)**

"Một trong những sai lầm đắt giá nhất trong quản lý dự án phần mềm là dàn trải nguồn lực. Chúng ta thường có xu hướng muốn mọi thứ đều phải hoàn hảo, mọi module đều phải được viết bằng công nghệ mới nhất.

Nhưng thực tế là: **Nguồn lực của chúng ta (thời gian, tiền bạc, kỹ sư giỏi) luôn hữu hạn.** Vì vậy, câu hỏi chiến lược ở đây là: _Chúng ta nên tập trung công sức vào đâu để tạo ra tác động lớn nhất?_ Theo Eric Evans, cha đẻ của DDD, chúng ta có 3 loại Subdomain."

#### **2. Phân loại 3 loại Subdomain (60 - 90 giây)**

"Hãy cùng nhìn vào ba nhóm này để thấy sự khác biệt về chiến lược:

- **Nhóm 1 - Core Domain (Miền lõi):** Đây là 'viên kim cương' của doanh nghiệp. Nó là lý do khách hàng chọn bạn thay vì đối thủ. Đây là nơi chứa đựng logic phức tạp nhất và mang lại lợi thế cạnh tranh.
- _Chiến lược:_ Phải tự xây dựng (In-house), sử dụng những kỹ sư giỏi nhất và áp dụng DDD triệt để nhất.
- \*Trái tim của lợi thế cạnh tranh của doanh nghiệp. Rất phức tạp, độc đáo và mang tính chiến lược. Yêu cầu đầu tư và phát triển tùy chỉnh nhiều nhất.

- **Nhóm 2 - Supporting Subdomain (Miền hỗ trợ):** Những phần này cần thiết để vận hành nhưng không tạo ra lợi thế cạnh tranh. Ví dụ: một hệ thống quản lý danh mục sản phẩm đơn giản.
- _Chiến lược:_ Giữ cho nó đơn giản nhất có thể. Không cần quá cầu kỳ, không cần những kiến trúc quá phức tạp.
- \*Thiết yếu cho hoạt động cốt lõi, nhưng không phải là yếu tố khác biệt chính. Thường phức tạp, nhưng ít độc đáo hơn. Có thể được xây dựng tùy chỉnh hoặc mua.

- **Nhóm 3 - Generic Subdomain (Miền chung):** Đây là những bài toán mà công ty nào cũng có, như Thanh toán, Gửi Email, hay Xác thực người dùng.
- _Chiến lược:_ Đừng 'tái phát minh bánh xe'. Hãy mua các giải pháp có sẵn (SaaS) hoặc dùng thư viện mã nguồn mở để tiết kiệm thời gian."
- \*Cần thiết cho hoạt động kinh doanh nhưng hoàn toàn được tiêu chuẩn hóa. Độ phức tạp thấp và không có sự khác biệt. Giải quyết tốt nhất bằng các giải pháp có sẵn.

#### **3. Tóm tắt giá trị (20 giây)**

"Việc phân loại này giúp chúng ta thoát khỏi tư duy 'mọi dòng code đều quý giá như nhau'. Nó giúp lãnh đạo kỹ thuật dũng mạnh nói 'Không' với việc tự xây dựng những thứ không cần thiết, để dồn toàn lực vào những thứ thực sự tạo ra giá trị cho doanh nghiệp."

#### **4. Chuyển tiếp (10 giây)**

"Để hiểu rõ hơn sự khác biệt về bản chất kỹ thuật và kinh doanh giữa 3 loại này, chúng ta hãy cùng đi sâu vào bảng so sánh chi tiết ở trang tiếp theo..."

---

**Trang 6**

---

#### **1. Dẫn dắt: Áp dụng vào thực tế (20 giây)**

- \*Mức đầu tư vừa phải. Mục tiêu là chức năng, không phải sự đổi mới. "Đủ tốt" là mục tiêu cần đạt được.
  Đặc điểm:
  • Giá trị: Cần thiết cho hoạt động, nhưng không tạo nên sự khác biệt giữa doanh nghiệp với các đối thủ cạnh tranh.

• Giá trị: Cần thiết cho hoạt động, nhưng không tạo nên sự khác biệt so với các đối thủ cạnh tranh.

• Độ phức tạp: Từ đơn giản đến trung bình.

• Nhân tài: Lập trình viên từ cấp độ sơ cấp đến trung cấp, hoặc các nhóm thuê ngoài.

"Để những khái niệm về Core, Supporting và Generic không còn là lý thuyết suông, chúng ta hãy cùng phân tích 'giải phẫu' một hệ thống Thương mại điện tử điển hình. Hãy tưởng tượng bạn là CTO và bạn phải quyết định sẽ đổ tiền đầu tư vào đâu."

#### **2. Phân tích cụ thể từng phần (90 giây)**

- **Tại sao Recommendation Engine là Core Domain?**
- "Trong thế giới Thương mại điện tử, thứ giữ chân khách hàng và tăng doanh thu chính là khả năng hiểu khách hàng muốn gì trước cả khi họ biết điều đó. Một thuật toán gợi ý sản phẩm độc quyền chính là **Lợi thế cạnh tranh**. Đây là nơi chúng ta cần những Data Scientist và Software Engineer giỏi nhất để tùy biến liên tục."

- **Tại sao Inventory/Catalog là Supporting?**
- "Bạn cần quản lý kho và danh mục sản phẩm để vận hành? Chắc chắn rồi. Nhưng liệu khách hàng có chọn mua hàng của bạn chỉ vì bạn có một cái trang quản lý kho đẹp không? Thường là không. Nó quan trọng nhưng không tạo sự khác biệt. Chúng ta sẽ xây dựng nó một cách đơn giản, đủ dùng, hoặc thuê một bên thứ ba tùy chỉnh nhẹ."

- **Tại sao Payments/Identity là Generic?**
- "Việc xác thực người dùng hay tích hợp thẻ tín dụng là những bài toán đã được giải quyết xong bởi Auth0, Stripe hay PayPal. Nếu bạn dành 6 tháng để tự xây dựng một cổng thanh toán từ đầu, bạn đang **đốt tiền vô ích** vào những thứ mà đối thủ chỉ cần mất 15 phút để tích hợp. Hãy dùng giải pháp có sẵn!"

#### **3. Bài học rút ra (30 giây)**

"Nhìn vào ví dụ này, các bạn sẽ thấy ranh giới rõ ràng:

- **Core:** Để chiến thắng đối thủ.
- **Supporting:** Để vận hành trơn tru.
- **Generic:** Để tiết kiệm thời gian và đảm bảo tiêu chuẩn.
  Nếu chúng ta đảo lộn thứ tự ưu tiên này, hệ thống của chúng ta sẽ trở nên cồng kềnh và mất đi sức mạnh cạnh tranh trên thị trường."

---

**Trang 7**

---

Đặc điểm:
• Giá trị: Không thể thiếu đối với doanh nghiệp (phải có), nhưng không mang lại lợi thế cạnh tranh.
• Tính độc đáo: Không có. Hầu hết các công ty đều thực hiện điều này theo cùng một cách.

Chiến lược đầu tư:
Nỗ lực kỹ thuật tối thiểu. Mua, không tự xây dựng. Sử dụng các giải pháp SaaS của bên thứ ba hoặc các giải pháp có sẵn.

Cảnh báo: Việc tự xây dựng một tên miền chung thường là lãng phí nguồn lực chiến lược.

#### **4. Chuyển tiếp (10 giây)**

"Vậy, sau khi đã xác định được các Subdomain, làm sao để chúng ta bắt đầu thiết kế hệ thống mà không làm mọi thứ rối tung lên? Chúng ta sẽ bước sang một khái niệm quan trọng tiếp theo: **Ubiquitous Language - Ngôn ngữ chung** ở trang 7."

---

**Trang 12 + 13**

Ngôn ngữ phổ quát: Thu hẹp khoảng cách giữa Kinh doanh và Lập trình

---

Dựa trên tài liệu của bạn, **Trang 7** là bước ngoặt quan trọng: Chuyển từ việc phân loại "Chiến lược" sang cách giao tiếp và tư duy mô hình thông qua **Ubiquitous Language (Ngôn ngữ chung)**. Đây là linh hồn của DDD.

---

### **Kịch bản Thuyết trình: Trang 7 (Ngôn ngữ chung - Ubiquitous Language)**

**Slide hiển thị:**

- **Tiêu đề:** Ubiquitous Language: Bridging the Gap Between Business and Code.
- **Hình ảnh:** Một bên là "Business Experts" (Chuyên gia kinh doanh), một bên là "Developers" (Lập trình viên), ở giữa là một vòng tròn giao thoa ghi chữ **"Ubiquitous Language"**.
- **Thông điệp chính:** Ngôn ngữ chung phải được sử dụng xuyên suốt từ cuộc họp đến mã nguồn.

---

#### **1. Dẫn dắt: Rào cản ngôn ngữ (30 giây)**

"Sau khi đã xác định được đâu là Core Domain để tập trung sức mạnh, câu hỏi tiếp theo là: _Làm sao để chúng ta đảm bảo những gì lập trình viên hiểu cũng chính là những gì chuyên gia kinh doanh mong muốn?_ Trong hầu hết các dự án thất bại, vấn đề không nằm ở thuật toán, mà nằm ở sự sai lệch về thuật ngữ. Chuyên gia kinh doanh nói một kiểu, Business Analyst (BA) viết tài liệu một kiểu, và Lập trình viên lại đặt tên biến trong code theo một kiểu khác hoàn toàn. Eric Evans gọi đây là thảm họa về giao tiếp."

#### **2. Giải pháp: Ubiquitous Language là gì? (60 giây)**

"Giải pháp của DDD chính là **Ubiquitous Language (Ngôn ngữ chung)**.

- Đây không phải là ngôn ngữ của riêng dân kỹ thuật, cũng không phải ngôn ngữ thuần kinh doanh.
- Đây là một ngôn ngữ được **thống nhất và sử dụng bởi tất cả mọi người** trong một đội nhóm.

Hãy tưởng tượng, nếu chuyên gia kinh doanh gọi quá trình thanh toán là 'Settlement', thì trong Code của bạn phải có một Class hoặc Method tên là `Settlement`. Tuyệt đối không được đặt là `ProcessPayment` hay `ExecuteTransaction`. Tại sao? Vì khi nhìn vào code, chuyên gia kinh doanh cũng có thể hiểu được logic đang vận hành."

Mã nguồn đúng cú pháp, nhưng lại phá hủy ngữ nghĩa. Hệ thống thất bại vì nó không mô phỏng được thực tế kinh doanh.

#### **3. Nguyên tắc "Code là tài liệu" (30 giây)**

"Mục tiêu cao nhất của Ubiquitous Language là biến mã nguồn thành một dạng tài liệu sống. Khi bạn thay đổi một thuật ngữ trong cuộc họp, bạn phải cập nhật nó trong code. Nếu một từ ngữ không tồn tại trong ngôn ngữ chung, nó không được phép tồn tại trong code. Điều này giúp loại bỏ hoàn toàn bước 'dịch' (translation) lãng phí và đầy rủi ro giữa các bên."

#### **4. Chốt hạ và Chuyển tiếp (15 giây)**

"Tuy nhiên, có một thách thức: Một từ có thể mang nghĩa này ở phòng ban này, nhưng lại mang nghĩa khác ở phòng ban khác. Làm sao để giải quyết sự nhập nhằng đó? Chúng ta sẽ bước sang khái niệm quan trọng nhất để giữ cho Ngôn ngữ chung được trong sạch: **Bounded Context (Ngữ cảnh giới hạn)** ở trang tiếp theo."

---

### **Ghi chú cho người thuyết trình:**

- **Từ khóa:** "No Translation" (Không thông dịch), "Living Documentation" (Tài liệu sống), "Shared Understanding" (Thấu hiểu chung).
- **Ví dụ nhanh:** "Đừng gọi là 'User' một cách chung chung nếu chuyên gia kinh doanh gọi họ là 'Subscriber' hoặc 'Lead'. Sự chính xác trong ngôn ngữ sẽ dẫn đến sự chính xác trong logic."
- **Mẹo diễn thuyết:** Hãy nhấn mạnh rằng đây là một nỗ lực của **cả đội ngũ**, không phải chỉ của riêng lập trình viên. Lập trình viên phải chủ động 'ép' chuyên gia kinh doanh định nghĩa thuật ngữ rõ ràng.

---

**Trang 15**

---

Defining Ubiquitous Language
A shared, precise, and consistent language used by Domain Experts, Engineers, Documentation, and Code

---

**Trang 27**

---

## Phần 1: Mở đầu - Người bảo vệ ý nghĩa (Trang 1)

**Lời dẫn:**
"Chào mọi người, hôm nay chúng ta sẽ cùng đi sâu vào một trong những mô hình quan trọng nhất, nhưng cũng thường bị hiểu lầm nhất trong thiết kế hướng tên miền (Domain-Driven Design) — đó chính là **Bounded Context** (Ngữ cảnh giới hạn).

Chúng ta thường gọi Bounded Context là **'Người bảo vệ ý nghĩa'** (The Guardian of Meaning). Tại sao lại như vậy? Trong một hệ thống phần mềm phức tạp, điều khó nhất không phải là viết code, mà là giữ cho ý nghĩa của các khái niệm nghiệp vụ luôn đồng nhất và không bị sai lệch khi hệ thống mở rộng.

---

**Trang 28**

---

## Phần 2: Tại sao chúng ta cần Bounded Context? (Trang 2)

**Lời dẫn:**
"Hãy nhìn vào chiếc dao đa năng này. Đây là hình ảnh phản chiếu của một hệ thống thiếu ranh giới. Khi chúng ta không xác định rõ giới hạn, hệ thống sẽ rơi vào tình trạng:

- **Mô hình bị thối rữa (Models rot):** Các đội nhóm bắt đầu tranh cãi về định nghĩa.

- **Địa ngục 'if-else':** Gốc rễ của vấn đề không phải do kỹ năng code kém, mà là do **ngôn ngữ mơ hồ**.

Khi một mô hình duy nhất cố gắng gánh vác quá nhiều thứ, chúng ta sẽ thấy 3 hệ lụy cực kỳ nguy hiểm:

1.  **Sự bất nhất cồng kềnh (Bloated Inconsistency):** Một thay đổi nhỏ ở phần này lại làm hỏng những phần hoàn toàn không liên quan.

2.  **Sự mong manh (Fragility):** Ví dụ điển hình là khi bạn thay đổi logic giao hàng nhưng lại vô tình làm hỏng các quy tắc kế toán.

3.  **Xung đột đội nhóm (Team Friction):** Những cuộc tranh luận bất tận về việc 'cái này thuộc về ai' và 'định nghĩa này nghĩa là gì'."

---

**Trang 29**

---

## Phần 3: Định nghĩa cốt lõi (Trang 3)

**Lời dẫn:**
"Vậy chính xác Bounded Context là gì? Hãy nhớ kỹ điều này: **Nó là ranh giới của ngôn ngữ, không chỉ là ranh giới của code**.

- **Định nghĩa:** Đó là một ranh giới rõ ràng mà trong đó một **Ngôn ngữ chung (Ubiquitous Language)** cụ thể được áp dụng. Bên trong ranh giới này, các mô hình domain có ý nghĩa rõ ràng, không gây nhầm lẫn.

- **Nguyên tắc:** Cùng một từ (như từ 'User'), nhưng trong các ngữ cảnh khác nhau sẽ có ý nghĩa hoàn toàn khác nhau.

**Lưu ý quan trọng cho các Architect:** Bounded Context trước hết **không phải** là một ranh giới kỹ thuật như database hay microservice. Nó là một **ranh giới về mặt ý nghĩa** (Meaning boundary)".

---

**Trang 30**

---

## Phần 4: Nhất quán bên trong, Phi tập trung bên ngoài (Trang 4)

**Lời dẫn:**
"Khi thiết kế Bounded Context, chúng ta phải tuân thủ một nguyên tắc 'sắt đá': **Sự nhất quán là bắt buộc ở bên trong, nhưng sự tách biệt (decoupling) là yêu cầu tiên quyết ở bên ngoài**.

- **Bên trong (INSIDE):** Ranh giới này bảo vệ mô hình nội bộ của chúng ta. Mọi quy tắc phải căn chỉnh theo Ngôn ngữ chung (Ubiquitous Language), đảm bảo không có sự mâu thuẫn.

- **Bên ngoài (OUTSIDE):** Ngôn ngữ có thể khác biệt và các thành phần phải được ngắt kết nối với nhau.

**Hệ quả:** Nếu chúng ta vi phạm quy tắc này, hệ thống sẽ rơi vào tình trạng **'Phụ thuộc về mặt ngữ nghĩa' (Semantic Coupling)**. Đây là dạng phụ thuộc nguy hiểm nhất trong kiến trúc phần mềm, nơi mà một thay đổi nhỏ về ý nghĩa ở module này sẽ tạo ra 'sóng thần' lỗi lan tỏa không kiểm soát trên toàn bộ hệ thống."

---

**Trang 31**

---

## Phần 5: Nghịch lý của thực thể 'Order' (Trang 5)

**Lời dẫn:**
"Để hiểu rõ hơn, hãy nhìn vào thực thể **'Order' (Đơn hàng)**. Trong một doanh nghiệp phức tạp, đây là một thuật ngữ bị 'quá tải' (overloaded term). Nó xuất hiện ở mọi nơi, nhưng ý nghĩa của nó lại không hề giống nhau.

Nếu chúng ta cố gắng nhồi nhét tất cả thực tế kinh doanh của một Đơn hàng vào duy nhất một Class trong code, chúng ta chắc chắn sẽ thất bại. Tại sao?

1.  **Tại bộ phận Bán hàng (Sales):** Đơn hàng là một ý định mua hàng (Purchase intent).

2.  **Tại bộ phận Giao hàng (Shipping):** Đơn hàng là một chiếc hộp vật lý (Physical box) cần vận chuyển.

3.  **Tại bộ phận Kế toán (Accounting):** Đơn hàng lại là một sự kiện tính thuế (Taxable event)."

---

**Trang 32**

---

## Phần 6: Đi sâu vào Context A - Ngôn ngữ của Bán hàng (Trang 6)

**Lời dẫn:**
"Bây giờ, hãy xem cách Bounded Context giải quyết vấn đề này. Trong **Ngữ cảnh Bán hàng (Sales Context)**, chúng ta nhìn nhận Đơn hàng dưới góc độ là 'Ý định mua hàng'.

- **Trọng tâm:** Context này chỉ quan tâm đến giá cả, các chương trình khuyến mãi và quy trình thanh toán (checkout).

- **Ngôn ngữ chung (Ubiquitous Language):** Ở đây, khi nói đến 'Order', mọi người hiểu đó là sự cam kết mua của khách hàng. Chúng ta có các thuật ngữ đi kèm như:

- **Order:** Ý định mua hàng.

- **OrderItem:** Các món hàng được chọn.

- **Cart & Checkout:** Giỏ hàng và quy trình thanh toán.

- **Discount:** Các mã giảm giá áp dụng.

Những khái niệm về trọng lượng vận chuyển hay mã định khoản kế toán hoàn toàn không có chỗ đứng trong mô hình này để đảm bảo sự tinh gọn và chính xác."

---

**Trang 33**

---

## Phần 7: Context B - Shipping: Đơn hàng là một "Kiện hàng" (Trang 7)

**Lời dẫn:**
"Tiếp theo, hãy bước vào thế giới của **Ngữ cảnh Giao hàng (Shipping Context)**. Tại đây, góc nhìn về 'Đơn hàng' thay đổi hoàn toàn.

- **Trọng tâm:** Chúng ta tập trung vào Logistics, trạng thái giao hàng và sự di chuyển vật lý của hàng hóa.

- **Sự loại bỏ:** Những thông tin như giá cả hay chiết khấu là hoàn toàn không liên quan ở đây.

- **Ngôn ngữ chung (Shipping):** Chúng ta không gọi là Order nữa, mà dùng thuật ngữ nội bộ là **Shipment** (Lô hàng). Các thực thể chính bao gồm:

- **Package:** Kiện hàng vật lý.

- **DeliveryRoute:** Tuyến đường giao hàng.

- **Carrier:** Đơn vị vận chuyển.

- **TrackingNumber:** Mã vận đơn để theo dõi hành trình.

Các bạn có thể thấy trên slide, các khái niệm về Credit Card hay Discount đã bị gạch chéo. Điều này giúp mô hình Shipping cực kỳ tinh gọn và tập trung".

---

**Trang 34**

---

## Phần 8: Context C - Accounting: Đơn hàng là một "Nghĩa vụ tài chính" (Trang 8)

**Lời dẫn:**
"Cuối cùng là **Ngữ cảnh Kế toán (Accounting Context)**. Ở đây, Đơn hàng được nhìn nhận như một 'Nghĩa vụ tài chính' (Fiscal Obligation).

- **Trọng tâm:** Mọi thứ xoay quanh hóa đơn, tuân thủ thuế và quyết toán thanh toán.

- **Ngôn ngữ chung (Accounting):** Những người làm kế toán sẽ nói bằng các thuật ngữ:
- **Invoice:** Hóa đơn tài chính.

- **Payment:** Khoản thanh toán.

- **LedgerEntry:** Bút toán ghi sổ.

- **TaxCalculation:** Cách tính thuế cho đơn hàng đó.

Một lần nữa, các chi tiết như 'Mô tả sản phẩm' hay 'Tuyến đường giao hàng' đều bị loại bỏ. Việc giữ cho mô hình này độc lập giúp chúng ta thay đổi luật thuế mà không ảnh hưởng gì đến hệ thống vận chuyển hay bán hàng".

---

**Trang 35**

---

## Phần 9: Hệ quả kiến trúc - Tách biệt Model và Database (Trang 9)

**Lời dẫn:**
"Đây là phần quan trọng nhất đối với các Architects: **Ý nghĩa khác nhau đòi hỏi các Model và Database khác nhau**.

Chúng ta không dùng chung một bảng `Orders` khổng lồ cho cả 3 phòng ban. Thay vào đó:

1.  **SALES:** Sử dụng `SalesDB` với model là 'Purchase Intent'.

2.  **SHIPPING:** Sử dụng `ShippingDB` với model là 'Package'.

3.  **ACCOUNTING:** Sử dụng `FinanceDB` với model là 'Fiscal Obligation'.

**Kết luận:** Việc áp dụng Bounded Context cho phép cùng một thuật ngữ tồn tại song song với các ý nghĩa khác nhau một cách an toàn. Đây chính là cách chúng ta triệt tiêu **Semantic Coupling** (Phụ thuộc về mặt ngữ nghĩa) và cho phép mỗi module tiến hóa độc lập".

---

**Trang 36**

---

## Phần 10: Bounded Context vs. Microservice (Trang 10)

**Lời dẫn:**
"Có một sự nhầm lẫn cực kỳ phổ biến: Coi Bounded Context và Microservice là một. Hãy cùng làm rõ điều này để định hình chiến lược kiến trúc đúng đắn:

- **Bounded Context:** Là ranh giới về **ngôn ngữ và mô hình**, mang tính trừu tượng và logic.

- **Microservice:** Là ranh giới về **triển khai và thời gian chạy (runtime)**, mang tính vật lý và cụ thể.

**Chiến lược cho các bạn:** Hãy áp dụng tư duy 'DDD trước, kiến trúc sau'. Bạn hoàn toàn có thể có nhiều Bounded Context (A, B, C) nằm bên trong một khối Monolith duy nhất khi mới bắt đầu. Bounded Context định nghĩa ranh giới logic, còn Microservice chỉ là một trong nhiều lựa chọn để triển khai các ranh giới đó."

---

**Trang 37**

---

## Phần 11: Heuristic 1 - Lắng nghe sự đứt gãy trong ngôn ngữ (Trang 11)

**Lời dẫn:**
"Làm sao để biết khi nào cần tách một Bounded Context? Hãy chú ý đến cách các chuyên gia nghiệp vụ (Domain Experts) nói chuyện. Những tín hiệu sau đây cho thấy một ranh giới mới đang cố gắng hình thành:

- Khi ai đó nói: _'Trong trường hợp này, đơn hàng có nghĩa là...'_.

- Hoặc: _'Đó không phải là cùng một loại đơn hàng đâu'_.

- Và câu nói kinh điển nhất: _'Nó tùy thuộc vào việc bạn đang hỏi ai'_.

Đây là những dấu hiệu có xác suất cực cao cho thấy bạn cần phải tách ranh giới (Boundary Split) để bảo vệ ý nghĩa của mô hình."

---

**Trang 38**

---

## Phần 12: Heuristic 2 - Quy tắc mâu thuẫn (Trang 12)

**Lời dẫn:**
"Dấu hiệu thứ hai là khi cùng một thực thể nhưng lại chịu sự chi phối của các quy tắc nghiệp vụ trái ngược nhau. Hãy nhìn vào ví dụ về việc hủy đơn hàng:

- **Trong Sales Context:** Quy tắc là khách hàng có thể hủy đơn bất cứ lúc nào để tối ưu trải nghiệm.

- **Trong Shipping Context:** Việc hủy đơn bị **CẤM** một khi hàng đã được bốc lên xe vận chuyển.

Nếu bạn cố nhồi nhét cả hai quy tắc này vào cùng một Class 'Order', bạn sẽ tạo ra một mớ hỗn độn các câu lệnh `if-else`. Công thức rất đơn giản: **Cùng một thực thể + Các quy tắc mâu thuẫn = Tách Context ngay lập tức**."

---

**Trang 39**

---

## Phần 13: Heuristic 3 - Định luật Conway (Trang 13)

**Lời dẫn:**
"Dấu hiệu cuối cùng để xác định ranh giới là nhìn vào chính sơ đồ tổ chức của công ty. Đây chính là **Định luật Conway**: Các tổ chức thiết kế hệ thống thường sẽ tạo ra những bản sao của cấu trúc giao tiếp trong chính tổ chức đó.

Hãy nhìn vào ví dụ này:

- **Sales Team (KPI về doanh thu):** Sẽ dẫn dắt việc hình thành **Sales Context**.

- **Logistics Team (KPI về tốc độ):** Sẽ định hình **Shipping Context**.

- **Finance Team (KPI về kiểm toán):** Sẽ tạo ra **Accounting Context**.

Đừng cố gắng chống lại điều này. Nếu các đội nhóm có mục tiêu và cách đo lường thành công khác nhau, họ nên sở hữu những Bounded Context riêng biệt để có thể ra quyết định và thay đổi nhanh chóng mà không cần đợi phe kia đồng ý."

---

**Trang 40**

---

## Phần 14: Quy trình khám phá và Tiến hóa (Trang 14)

**Lời dẫn:**
"Một sai lầm lớn là cố gắng thiết kế hoàn hảo các Bounded Context ngay từ đầu. Thực tế, chúng được **khám phá ra**, không phải được thiết kế sẵn.

Quy trình thực tế thường diễn ra như sau:

1.  **Big Ball of Mud:** Bạn bắt đầu với một mớ hỗn độn nơi mọi thứ dính chặt vào nhau.

2.  **Xác định xung đột:** Tìm ra những điểm mà ngôn ngữ và quy tắc nghiệp vụ bắt đầu 'đá' nhau.

3.  **Trích xuất ngữ cảnh:** Tách dần các ngữ cảnh nhỏ hơn như Sales hay Shipping.

4.  **Tích hợp rõ ràng:** Thiết lập các kết nối minh bạch giữa chúng.

Hãy nhớ rằng: Việc tái cấu trúc (refactoring) các ngữ cảnh là bình thường và được kỳ vọng. Đó là quá trình học hỏi không ngừng về tên miền nghiệp vụ của chúng ta."

---

**Trang 41**

---

## Phần 15: Kết luận - Bảo vệ sự minh bạch (Trang 15)

**Lời dẫn:**
"Để kết thúc, tôi muốn các bạn ghi nhớ thông điệp quan trọng nhất: **Sự nhất quán về ngôn ngữ quan trọng hơn việc tái sử dụng code**.

Tóm lại:

- Bounded Context sinh ra để **bảo vệ ý nghĩa** của hệ thống.

- Mọi sự tích hợp giữa các ngữ cảnh phải là **tường minh**, không được ngầm định.

- Các ranh giới này chính là chìa khóa để mỗi bộ phận có thể **tiến hóa độc lập**.

Nếu Ngôn ngữ chung (Ubiquitous Language) định nghĩa ý nghĩa, thì Bounded Context chính là 'người vệ sĩ' bảo vệ nó khỏi sự thối rữa. Cảm ơn mọi người đã lắng nghe!"

---

**Trang 42**

---

### PHẦN 1: Mở đầu - Ý nghĩa của Context Map (Trang 1)

Thông thường, khi nói đến thiết kế phần mềm, chúng ta hay tập trung vào code. Nhưng thực tế, phần mềm không vận hành trong chân không. Bản đồ ngữ cảnh không chỉ là một sơ đồ kỹ thuật thuần túy; nó là một **công cụ chiến lược**.

Tại sao chúng ta lại cần nó? Bởi vì nó tiết lộ ba điều 'ẩn giấu' mà code không nói cho bạn biết:

1.  **Sự phụ thuộc tiềm ẩn** giữa các hệ thống.

2.  **Ma sát trong tổ chức** (Tại sao hai team lại hay cãi nhau?).

3.  **Động lực quyền lực** (Ai là người quyết định cấu trúc dữ liệu?).

Mục tiêu cao nhất của chúng ta khi sử dụng công cụ này là để phân loại các mối quan hệ—từ hợp tác chặt chẽ đến cách ly hoàn toàn—nhằm tránh rơi vào thảm họa **'Big Ball of Mud' (Đống bùn khổng lồ)**, nơi mà mọi thứ rối tung và không thể kiểm soát."

---

**Trang 43**

---

### PHẦN 2: Đối mặt với rủi ro & Định nghĩa (Trang 2)

Người thuyết trình: "Hãy nhìn vào hình ảnh phía trên trang 2. Đó là 'Big Ball of Mud'. Đây là hậu quả của việc hệ thống bị thoái hóa thành sự hỗn loạn (entropy) do thiếu định hướng.

**Vậy chính xác Context Map là gì?**
Đó là một biểu diễn trực quan về:

- Các **Bounded Contexts** (Ngữ cảnh bị giới hạn) hiện có.
- Các **mô hình tích hợp** giữa chúng.
- Và quan trọng nhất: **Dòng chảy của tầm ảnh hưởng**.

Nó trả lời cho chúng ta những câu hỏi chiến lược như: Hệ thống đang bị chia cắt như thế nào? Ai đang ảnh hưởng đến ai (Upstream vs Downstream)? Và đâu là nơi có rủi ro bị 'ô nhiễm' mô hình dữ liệu?.

Hãy nhớ: Bản đồ này phản ánh **mối quan hệ giữa các con người và tổ chức**, chứ không chỉ là các dòng lệnh."

---

**Trang 44**

---

### PHẦN 3: Ngôn ngữ của sự ảnh hưởng (Trang 3)

Người thuyết trình: "Để đọc được bản đồ này, chúng ta cần hiểu 'Ngôn ngữ của sự ảnh hưởng' với hai khái niệm then chốt:

1.  **Upstream (U) - Phía thượng nguồn:** Đây là bên cung cấp dữ liệu hoặc dịch vụ. Họ là người có tầm ảnh hưởng. Trong các mối quan hệ bất đối xứng, họ là người áp đặt các ràng buộc.

2.  **Downstream (D) - Phía hạ nguồn:** Đây là bên tiêu thụ dữ liệu. Họ chịu tác động trực tiếp từ những thay đổi của phía Upstream. Khi Upstream thay đổi, Downstream phải thích nghi, thương lượng hoặc tự xây dựng hệ thống phòng thủ cho mình.

Sự phân chia này giúp chúng ta thấy rõ 'Dòng chảy quyền lực' trong hệ thống."

---

**Trang 45**

---

### PHẦN 4: Mô hình "Hợp tác chặt chẽ" - Shared Kernel (Trang 4)

**Người thuyết trình:**
"Mô hình đầu tiên chúng ta xem xét là **Shared Kernel**, hay tôi gọi vui là 'Cuộc hôn nhân' giữa hai team.

- **Định nghĩa:** Hai team chia sẻ một tập hợp con nhỏ nhưng quan trọng của mô hình miền, ví dụ như code, database hoặc các Value Objects.

- **Ví dụ:** Trong thương mại điện tử, team **Sales** và team **Catalog** có thể dùng chung class `Product` (ID, Name, SKU) và object `Money`.

- **Quy tắc sắt:** Phần dùng chung này không được phép thay đổi độc lập. Bất kỳ sự chỉnh sửa nào cũng cần sự đồng thuận và thảo luận chặt chẽ từ cả hai phía.

**Đánh giá:** Nó giúp giảm trùng lặp nhưng lại làm tăng sự ràng buộc (coupling). Chỉ nên dùng khi thực sự có sự chồng chéo về nghiệp vụ."

---

**Trang 46**

---

## 1. Shared Kernel

**Shared Kernel** is a Context Mapping pattern where two or more Bounded Contexts share a **small, carefully defined subset of the domain model** (the “kernel”), which may include:

- Entities
- Value Objects
- Ubiquitous Language
- Even code implementations

This shared part **cannot be changed independently**.
Any modification requires **discussion and agreement** between the owning teams.

### Goals

- Avoid duplication when there is genuine domain overlap
- Maintain consistency in the shared core
- Still allow independence outside the kernel

This is a **symmetric pattern**: there is no clear upstream or downstream; the teams are equal partners.

### Real-World Example

In a large e-commerce system:

- **Sales Context**: manages sales, orders, promotions
- **Catalog Context**: manages products, categories, attributes

Both need a basic concept of **Product**:

- ProductId
- Name
- SKU
- Price

→ Create a Shared Kernel containing:

- `Product` class (Id, Name, SKU)
- `Money` value object
- Shared repository interfaces

---

## 2. Customer / Supplier

**Customer/Supplier** describes a clear dependency relationship between two Bounded Contexts (and their teams):

- **Upstream (U) – Supplier**: provides models, services, or data
- **Downstream (D) – Customer**: depends on the upstream to fulfill its business responsibilities

Unlike Shared Kernel or Partnership, this relationship is **asymmetric**.

The downstream has real needs, and the upstream **should evolve to support those needs**, like a good supplier serving its customer.

On a Context Map, the arrow usually goes **U → D** (upstream influences downstream).

### When to Use Customer/Supplier

- Downstream strongly depends on upstream to deliver its business capability
- Upstream is more powerful or represents a core domain
- Teams can collaborate well: downstream expresses needs, upstream prioritizes support
- You want to avoid complex translation layers (ACL) or forced conformity (Conformist)

### When NOT to Use

- Upstream ignores downstream needs → leads to bottlenecks
- Team relationships are tense → Conformist or ACL may be safer

### Example

In an e-commerce system:

- **Supplier (Upstream)**: Catalog Context (product management team)
- **Customer (Downstream)**: Sales Context (orders and checkout team)

Sales team:

> “We need a `DiscountPrice` field on Product to calculate promotions.”

Catalog team prioritizes and implements it because Sales is an important customer.

---

## 3. Conformist

**Conformist** is a pattern where the downstream context **fully accepts the upstream model** without any translation (no ACL).

- **Upstream (U)**: dominant, unwilling or unable to change
- **Downstream (D)**: must conform, even if the model does not fit its own ubiquitous language
- Arrow: **U → D**

This is a **pragmatic pattern** used when the downstream is weaker (organizationally, politically, or technically).

### Difference from Customer/Supplier

- Customer/Supplier: upstream actively supports downstream
- Conformist: upstream does not care; downstream adapts

### When to Use Conformist

- Upstream is a third-party system (e.g., Stripe, SAP)
- Upstream is a powerful core team that won’t adapt
- Downstream lacks time/resources to build an ACL
- Upstream model is “good enough”

Avoid it if the upstream model is truly harmful → use ACL instead.

### Example

- **Upstream**: Pricing Context (legacy pricing engine)
- **Downstream**: Sales Context

Pricing team refuses to change APIs.
Sales team conforms and uses Pricing’s model directly—even if field names don’t match Sales’ ubiquitous language.

Classic example: integrating with PayPal—you conform to their API and terminology.

---

## 4. Anti-Corruption Layer (ACL)

An **Anti-Corruption Layer (ACL)** is a **translation layer** built by the downstream to protect its domain model from being “corrupted” by upstream models.

“Corruption” means:

- Poor-quality legacy models
- Different ubiquitous language
- Anemic or unstable schemas
- Frequent upstream changes

ACL acts as a **protective wall**:

- Translates inbound models into clean downstream models
- Optionally translates outbound calls as well

Context Map flow: **U → ACL → D**

### Difference from Conformist

- Conformist accepts corruption
- ACL prevents it and keeps the downstream model expressive and clean

### When to Use ACL

- Upstream is legacy or third-party
- Models do not fit downstream language
- Upstream changes frequently
- Downstream is a core domain and must remain clean

### Example

A modern e-commerce system:

- **Downstream**: Sales Context (modern, core domain)
- **Upstream**: Legacy ERP Inventory Context using `ItemMasterRecord`

Without ACL → Sales code becomes polluted
With ACL:

- ACL receives legacy data
- Translates `ItemMasterRecord` → `Product`
- Applies transformation rules (e.g., derive Price from multiple legacy fields)
- Sales works only with its own clean `Product`

### Pros & Cons

**Pros**

- Protects domain purity
- Strong decoupling
- Ideal for legacy modernization

**Cons**

- Extra complexity
- Mapping maintenance required
- Overkill if upstream model is already good

### Typical ACL Components

- **Facade**: Simplified interface over complex external systems
- **Adapter**: Converts external interfaces into expected ones
- **Translator**: Core mapping logic (e.g., legacy status → domain enum)

---

## 5. Separate Ways

**Separate Ways** describes a situation where Bounded Contexts are **completely independent**:

- No dependencies
- No shared models
- No APIs or domain events
- Each context evolves independently

On a Context Map, these contexts appear unconnected or explicitly labeled “Separate Ways”.

This pattern provides **maximum autonomy** and is recommended when integration adds more risk than value.

### When to Use Separate Ways

- Contexts belong to entirely different subdomains
- Integration would create unnecessary coupling
- Supporting or generic domains unrelated to the core
- Large organizations aiming for team autonomy

### Example

In a large e-commerce company:

- **Sales Context**: orders, payments, shipping
- **HR Context**: recruitment, payroll
- **Monitoring Context**: internal logs and metrics

HR and Monitoring do not integrate with Sales → Separate Ways.

I’ve applied this successfully in fintech projects by isolating **Compliance Reporting** from core transaction systems.

---

## 6. Open Host Service (OHS)

### What is Open Host Service?

**Open Host Service (OHS)** is a pattern where an upstream context exposes a **clear, stable, well-defined service/API** for multiple downstream contexts.

- Upstream becomes an “open host”
- Interfaces may be REST, GraphQL, gRPC, messaging, etc.
- Often combined with **Published Language**
- Context Map: **U → many D**

### Goals

- Enable effective integration with loose coupling
- Allow downstreams to consume services without knowing internal models
- Support scale and evolution

### Published Language – the Companion of OHS

OHS works best with **Published Language (PL)**:

- A standardized, documented schema or language
- Versioned and publicly shared
- Examples: OpenAPI, JSON Schema, Protobuf, OpenID Connect

Without PL, OHS degrades into unstable, ad-hoc APIs.

### When to Use OHS

- Upstream is a core or platform domain
- Many downstream consumers
- Microservices or SOA environments
- Need long-term evolvability

### Example

In e-commerce:

- **Upstream**: Catalog Context
- **Downstreams**: Sales, Recommendation, Search

Catalog exposes:

- REST API `/products/{id}`
- Published Language with standardized JSON schema

Downstreams integrate via API or domain events without knowing internal Catalog models.

I’ve used OHS in fintech systems where an **Identity Context** exposed a GraphQL OHS based on OpenID Connect standards, enabling 10+ services to integrate safely.

### Pros & Cons

**Pros**

- Easy integration for many consumers
- Reduced coupling
- Supports versioning and evolution

**Cons**

- Requires strong API design discipline
- Governance and documentation overhead
- Overkill for small systems

---
