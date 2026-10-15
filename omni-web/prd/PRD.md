# OmniGem Gallery — BRD/PRD (Giai đoạn P1: Khám phá & Yêu cầu)

Trạng thái: **ĐÃ CHỐT (APPROVED)** — chốt ngày 2026-10-15. Tác giả: requirements-analyst. Người phê duyệt: Product Owner + Risk/Compliance.
Nguồn tham khảo: `documents/omnigem-gallery-research.md` (toàn bộ các mục), `documents/create-facebook-fanpage.md` (§1–3).

> **Bản sửa R1 (2026-10-15, PO duyệt) — reconcile theo `design/adr/ADR-002`:** (a) **Giá thực tế: 500k–5tr VND** (không phải 20–200tr như bản gốc). (b) **Lý do giữ model "chốt qua người"** đổi từ "giá quá cao" sang **"giai đoạn đầu: validate bằng bán qua người trước, bán online tính sau"**. (c) **Kênh handoff chính = Facebook Messenger** (Page miễn phí), **Zalo OA bật sau** khi có doanh thu — các mục nhắc "Zalo" ở dưới hiểu theo nghĩa "kênh chat handoff (Messenger-first, Zalo OA sau)". Mô hình phễu và **NG1 (không checkout) giữ nguyên** ở giai đoạn này.

> Ghi chú bản dịch: Đây là bản dịch tiếng Việt của tài liệu `.chapter-forge/artifacts/discover/PRD.md`. Nội dung được giữ nguyên ý nghĩa gốc; các thuật ngữ kỹ thuật phổ biến (checkout, funnel, lead, chatbot, handoff, schema...) được giữ nguyên tiếng Anh kèm giải thích khi cần.

---

## 1. Tóm tắt điều hành & Luận chứng kinh doanh (Executive Summary & Business Case)

OmniGem Gallery là một website quảng bá cho một thương hiệu trang sức ngọc bích & đá quý tự nhiên của Việt Nam. Website này **không phải** là một hệ thống thương mại điện tử có checkout. Các sản phẩm (mức giá **500k–5 triệu VND**) hiện không được mua trực tiếp trên web; nhiệm vụ của website là xây dựng niềm tin và kết nối cảm xúc, sau đó dẫn khách truy cập vào một cuộc hội thoại chốt đơn có người hỗ trợ qua kênh chat (Messenger-first, Zalo OA sau).

**Luận chứng kinh doanh (reconcile R1):**
- **Giai đoạn đầu ưu tiên validate bằng bán qua người**: thu lead → chăm sóc + chốt qua người → ship / kéo về cửa hàng; bán online (checkout) để sau nếu mô hình chứng minh hiệu quả. Đây là lý do (không phải "giá quá cao") để chưa làm giỏ hàng/checkout — tránh đầu tư sớm vào luồng giao dịch chưa được kiểm chứng.
- Thị trường (Việt Nam) chốt giao dịch dựa nhiều vào niềm tin qua chat cá nhân, đáng tin cậy hơn so với checkout tự phục vụ ở giai đoạn thương hiệu còn mới.
- Nhiệm vụ của web = (a) chứng minh tính xác thực ("ngọc thật"), (b) chứng minh uy tín người bán, (c) cung cấp tư vấn khớp phong thủy/ngũ hành, rồi (d) chuyển giao cho người thật.
- Nội dung SEO về tính xác thực của ngọc bích / phong thủy là một kênh traffic bền vững, chi phí biên gần như bằng không (research doc §6b).

**Luận điểm cốt lõi (Thesis):** OmniGem Gallery = showroom số (digital showroom) + cỗ máy niềm tin (trust engine) + phễu dẫn về Zalo/Chatbot, **không phải** một sàn giao dịch (marketplace).

## 2. Mục tiêu & Phi mục tiêu (Goals & Non-Goals)

### Mục tiêu (Goals)
- **G1:** Trình bày catalog sản phẩm (Collections + Product Detail) với hình ảnh/video chất lượng cao, chứng thực được các tuyên bố về tính xác thực.
- **G2:** Cung cấp điều hướng/lọc dựa trên phong thủy/ngũ hành (năm hành) để khớp khách truy cập với sản phẩm theo mệnh (hành bản mệnh).
- **G3:** Chuyển đổi sự quan tâm của khách thành một lead đủ điều kiện thông qua CTA deep-link Zalo, một form liên hệ/lead, và một AI chatbot có human handoff (chuyển giao cho người thật) bắt buộc.
- **G4:** Xếp hạng tự nhiên (organic) cho các truy vấn tìm kiếm về tính xác thực của ngọc bích và trang sức phong thủy (SEO).
- **G5:** Đo lường (instrument) toàn bộ phễu (page view → tương tác chatbot → click Zalo / submit form lead) trong analytics.
- **G6:** Cho phép nhân viên cửa hàng không chuyên kỹ thuật quản lý nội dung catalog/blog qua một built-in CMS (tầng quản trị tự xây bằng React-Admin — xem §4.1, Câu hỏi mở 3 đã chốt).

### Phi mục tiêu (Non-Goals — nêu rõ ràng)
- **NG1:** **Không có checkout, giỏ hàng, hoặc xử lý thanh toán trên website dưới bất kỳ hình thức nào.** Không dữ liệu thẻ, không luồng card-present/card-not-present, không PCI scope trên website trong giai đoạn này.
- **NG2:** Không có hệ thống quản lý tồn kho/đơn hàng; việc hoàn tất (fulfillment) và thanh toán diễn ra hoàn toàn ngoài nền tảng qua Zalo (thương lượng với người thật, COD/đặt cọc theo research doc §6c).
- **NG3:** Không có hệ thống tài khoản/đăng nhập cho người mua (research không nêu là cần; được đánh dấu là câu hỏi mở, xem §8).
- **NG4:** Không đa tiền tệ/giao hàng quốc tế trong giai đoạn này (tập trung thị trường Việt Nam, theo phễu lấy Zalo làm trung tâm).

## 3. Người dùng mục tiêu / Chân dung (Target Users / Personas)

1. **Khách mua khớp phong thủy** — muốn một chiếc vòng/mặt dây chuyền khớp với ngũ hành (hành bản mệnh) của họ; đường chuyển đổi chính là tư vấn dựa trên ngày sinh (DOB) của chatbot (research doc §4, FAQ Q2 của fanpage guide).
2. **Khách mua quà** — mua cho người thân/bạn đời; quan tâm đến cách trình bày, chứng nhận, và chính sách bảo hành/đổi trả (FAQ Q3 của fanpage guide) hơn là sự khớp phong thủy cá nhân.
3. **Nhà sưu tầm / khách giá trị cao mua lặp lại** — đánh giá các tác phẩm điêu khắc/độc bản; cần chi tiết sản phẩm sâu (nguồn gốc, chứng nhận, zoom ảnh độ phân giải cao) và thoải mái khi được dẫn thẳng tới Zalo để thương lượng (research doc §2 "Chi tiết tác phẩm").

*(Các chân dung được suy luận từ phần IA và marketing của research doc, không phải từ nghiên cứu người dùng sơ cấp. Được đánh dấu là một giả định — xem §8.)*

## 4. Phạm vi theo giai đoạn (Scope by Phase — bám theo roadmap §7 của research doc)

| Giai đoạn | Phạm vi | Kết quả chính |
|---|---|---|
| P0 — Nền tảng (Foundation) | Khung sườn **Next.js** (public showroom) + **React-Admin** (trang quản trị nội bộ) + CMS + design system, sitemap, deploy Vercel | Website khung sườn chạy được (public + admin) |
| P1 — Gallery | Home, Collections, Product Detail, hình/video tối ưu, SEO cơ bản | Showroom hoàn chỉnh |
| P2 — Điều hướng/Phễu (Navigation/Funnel) | Widget Zalo + deep-link theo từng sản phẩm, form lead, GA4/Meta Pixel | Tạo lead đi vào hoạt động |
| P3 — AI Chatbot | Widget chat + Gemini API + RAG trên catalog/FAQ + human handoff | Tư vấn tự động 24/7 |
| P4 — Tăng trưởng (Growth) | Nội dung blog SEO, remarketing, A/B test CTA, tin nhắn chăm sóc Zalo ZNS | Tăng trưởng traffic & chuyển đổi |

Các yêu cầu chức năng bên dưới của PRD này bao phủ P1–P3 (các giai đoạn có bề mặt chức năng mới hoàn toàn); P0 là thiết lập hạ tầng và P4 là công việc tối ưu hóa sẽ được xác định phạm vi lại sau.

### 4.1 Quyết định tech stack cho P0 (ghi lại lý do)

Bối cảnh: đội dev là **React thuần**, một thành viên vừa quay lại sau kỳ nghỉ sinh, nên ưu tiên chọn tech stack quen thuộc để dễ bắt nhịp. Phân tích & quyết định:

- **Public showroom → giữ Next.js.** Next.js *chính là* React — dev viết React components y hệt, chỉ cần làm quen thêm một số khái niệm riêng của Next (App Router, file-based routing, ISR) trong vài ngày. Quan trọng hơn: toàn bộ luận chứng kinh doanh đặt cược vào **SEO** (G4, NFR-3, giai đoạn P4 Growth), đòi hỏi render SSG/ISR để crawler nhận HTML đầy đủ. Chuyển public site sang React thuần (Vite SPA) render phía client sẽ làm yếu SEO và phá vỡ chính kênh traffic cốt lõi → **không chọn**. Giữ Next.js là lựa chọn rủi ro thấp nhất mà vẫn 100% là React.
- **Trang quản trị nội bộ → dùng React-Admin.** Đây là chỗ React-Admin thực sự phù hợp: dựng admin/CRUD dashboard phía client (không cần SEO). Dùng cho **Epic F** (quản lý catalog cho nhân viên không chuyên kỹ thuật) và **CRM built-in / dashboard quản lý lead** — thay hoàn toàn cho Google Sheet, xử lý rủi ro **R-03** (kiểm soát truy cập/audit yếu của Google Sheet) và chốt **Câu hỏi mở 7** (kho lead). Dev làm việc bằng React thuần ở phía admin, không phải học concept mới.
- **Kết luận:** dev làm việc trong hệ sinh thái React xuyên suốt cả hai phía (Next.js cho public + React-Admin cho admin). Lựa chọn thay thế (Vite SPA, Remix/React Router v7) đã được cân nhắc và loại; nếu cần xem lại, đưa vào quyết định ở giai đoạn Thiết kế.

## 5. Yêu cầu chức năng — User Stories & Tiêu chí chấp nhận (Acceptance Criteria)

### Epic A — Các trang Gallery

**US-A1: Trang chủ (Home page)**
Là một khách truy cập, tôi muốn một trang chủ ấn tượng điện ảnh (cinematic) làm nổi bật các sản phẩm tiêu biểu, để tôi tin tưởng thương hiệu và muốn khám phá thêm.
- AC1: Khi một khách truy cập vào `/`, lúc trang tải xong, thì một section hero và một tập sản phẩm tiêu biểu được tuyển chọn render trong ngân sách Core Web Vitals (xem NFR-1).
- AC2: Khi khách cuộn trang, lúc đến section tiêu biểu, thì mỗi product card liên kết tới trang Product Detail của nó.

**US-A2: Duyệt Collections với bộ lọc phong thủy**
Là một khách truy cập, tôi muốn duyệt theo danh mục (vòng tay, mặt dây, tượng điêu khắc) và lọc theo ngũ hành/mệnh, để tôi tìm được những sản phẩm khớp với hành của mình.
- AC1: Tại trang Collections, khi khách chọn một danh mục, thì chỉ các sản phẩm thuộc danh mục đó hiển thị.
- AC2: Tại trang Collections, khi khách áp dụng bộ lọc "Mệnh" (hành) — ví dụ Kim/Mộc/Thủy/Hỏa/Thổ — thì chỉ các sản phẩm được gắn tag hành khớp hiển thị.
- AC3: Khi không có sản phẩm nào khớp với bộ lọc kết hợp, lúc bộ lọc được áp dụng, thì một trạng thái rỗng (empty state) với CTA "Hỏi chatbot" hoặc "Nhắn Zalo" hiển thị (fallback phễu, không phải ngõ cụt).

**US-A3: Trang chi tiết sản phẩm (Product Detail)**
Là một khách truy cập, tôi muốn xem đầy đủ chi tiết về một sản phẩm (hình ảnh, thông số, xuất xứ, chứng nhận, độ khớp phong thủy, các CTA), để tôi có thể quyết định theo đuổi nó.
- AC1: Tại trang Product Detail, khi trang tải, thì nó hiển thị: gallery ảnh/video độ phân giải cao (zoom được), tên/mã sản phẩm, loại ngọc, kích thước, xuất xứ, mã tham chiếu chứng nhận, giá HOẶC "Liên hệ để được tư vấn" (theo mô hình giá ở §6a), và một khối độ khớp phong thủy.
- AC2: Tại trang, khi render, thì hai CTA chính hiển thị phía trên nếp gấp (above the fold) trên mobile: "Nhắn Zalo về sản phẩm này" (deep-link, xem US-B1) và "Hỏi AI về sản phẩm này" (mở chatbot đã được nạp sẵn ngữ cảnh sản phẩm).
- AC3: Khi sản phẩm có phản hồi khách hàng/media chứng thực, lúc có sẵn, thì nó render dưới dạng một section bằng chứng xã hội (social proof).

**US-A4: Trang câu chuyện thương hiệu / niềm tin (Brand story / trust)**
Là một khách truy cập, tôi muốn thấy xuất xứ và các tuyên bố chứng nhận, để tôi tin các sản phẩm là thật.
- AC1: Tại `/brand-story` (hoặc tương đương), khi tải, thì nó trình bày câu chuyện nguồn gốc (sourcing), giải thích chứng nhận phòng lab, và liên kết tới trang trưng bày Chứng nhận — tái sử dụng ngôn ngữ niềm tin từ `documents/create-facebook-fanpage.md` §2 FAQ Q1 ("ngọc tự nhiên 100%... chứng nhận từ một phòng lab đá quý uy tín").

### Epic B — Phễu Zalo

**US-B1: CTA deep-link Zalo theo từng sản phẩm**
Là một khách quan tâm một sản phẩm, tôi muốn một cú chạm để mở Zalo với ngữ cảnh sản phẩm được điền sẵn, để nhân viên biết chính xác tôi đang hỏi về gì.
- AC1: Tại trang Product Detail, khi khách chạm "Nhắn Zalo về sản phẩm này", thì Zalo mở ra (app trên mobile, Zalo Web trên desktop) với một tin nhắn điền sẵn chứa mã sản phẩm — cơ chế theo tài liệu deep-link/prefill chính thức của Zalo (research doc §5.3 đánh dấu điều này cần được kiểm chứng lại với tài liệu Zalo Developer hiện hành trước khi build).
- AC2: Khi sự kiện chạm xảy ra, lúc nó kích hoạt, thì một sự kiện analytics `click_zalo` được gửi kèm product ID (research doc §5.5).
- AC3: **Câu hỏi mở:** cú pháp tham số prefill chính xác của Zalo chưa được kiểm chứng trong research doc — phải được xác nhận với tài liệu Zalo Developer trực tiếp trước khi triển khai (xem §8).

**US-B2: Widget Zalo chung**
Là bất kỳ khách truy cập nào, tôi muốn một điểm vào chat Zalo thường trực, để tôi có thể liên hệ người thật bất cứ lúc nào.
- AC1: Tại bất kỳ trang nào, khi nó render, thì một widget chat Zalo nổi (script Zalo chính thức) hiện diện, theo research doc §5.2.

### Epic C — Thu thập Lead (Lead Capture)

**US-C1: Form liên hệ/tư vấn**
Là một khách chưa sẵn sàng dùng Zalo/chat, tôi muốn để lại thông tin liên hệ và mối quan tâm của mình, để nhân viên có thể theo dõi (follow up).
- AC1: Tại trang Liên hệ/Tư vấn, khi khách gửi họ tên, số điện thoại hoặc Zalo ID, **ngày sinh**, và sản phẩm quan tâm, thì nội dung gửi được xác thực (các trường bắt buộc, định dạng số điện thoại) trước khi được chấp nhận.
- AC2: Khi một lượt gửi hợp lệ, lúc lưu, thì nó được ghi vào kho lead (**CRM built-in** — xem Câu hỏi mở 7 đã chốt) và một thông báo xác nhận hiển thị.
- AC3: Khi form thu thập ngày sinh (DOB), lúc form được trình bày, thì một thông báo/tuyên bố đồng ý về quyền riêng tư (privacy notice/consent) được hiển thị giải thích DOB được dùng cho tư vấn phong thủy, và việc thu thập đồng ý thỏa mãn Điều 11 (xem Epic H, US-H2) — **phụ thuộc vào phê duyệt compliance**, xem `compliance-scope.md`.
- AC4: Không có PII thật nào được xuất hiện trong test fixtures hoặc tài liệu; chỉ dùng dữ liệu tổng hợp (synthetic) (theo quy ước của test-engineer/data-classification).

### Epic H — Chính sách quyền riêng tư & Quyền của chủ thể dữ liệu (Nghị định 13/2023)

Compliance-checker đã xác nhận (xem `compliance-scope.md`) rằng đây là một hạng mục bàn giao chức năng chắc chắn cho Gate G0/G1, không phải một chú thích bên lề — được nâng lên thành một epic riêng.

**US-H1: Trang chính sách quyền riêng tư đã công bố**
Là một khách truy cập, tôi muốn đọc về cách dữ liệu cá nhân của tôi được thu thập, sử dụng và bảo vệ, để tôi có thể ra quyết định có hiểu biết trước khi gửi bất kỳ dữ liệu nào.
- AC1: Tại bất kỳ trang nào thu thập dữ liệu cá nhân (form lead, chatbot), khi khách tìm nó, thì một trang Chính sách Quyền riêng tư đã liên kết, đã công bố có thể truy cập được từ trang đó và từ footer của website.
- AC2: Tại trang Chính sách Quyền riêng tư, khi tải, thì nó công khai: dữ liệu nào được thu thập (tên, số điện thoại/Zalo ID, DOB, sản phẩm quan tâm, bản ghi chat, định danh analytics), vì sao (tư vấn phong thủy, theo dõi lead, phân tích phễu), ai xử lý nó (OmniGem là bên kiểm soát; Vercel, Cloudinary, Google/Gemini API, GA4, Meta Pixel, Zalo là các bên xử lý/bên nhận — xem `compliance-scope.md` Nghĩa vụ B), và rằng một số xử lý/lưu trữ diễn ra bên ngoài Việt Nam.

**US-H2: Thu thập đồng ý minh thị, tái lập được (Điều 11)**
Là doanh nghiệp, tôi cần một bản ghi đồng ý đủ tính pháp lý, để các nghĩa vụ Điều 11 PDPD được đáp ứng — không chỉ là một checkbox trên UI mà không có bằng chứng truy xuất được.
- AC1: Tại form lead hoặc yêu cầu DOB của chatbot, khi một khách gửi dữ liệu cá nhân, thì hệ thống thu thập và lưu trữ bền vững một **bản ghi đồng ý tái lập được** (đã đồng ý với điều gì, phiên bản của thông báo quyền riêng tư đã hiển thị, timestamp) — không chỉ đơn thuần là trạng thái checkbox không được ghi log.
- AC2: Khi một bản ghi đồng ý tồn tại, lúc được yêu cầu (ví dụ bởi khách hoặc một kiểm toán viên), thì nó có thể được truy xuất và trình bày như bằng chứng của sự kiện đồng ý cụ thể đó.
- AC3: Vì Điều 11 áp dụng cho MỌI dữ liệu cá nhân mà không có ngoại lệ lợi ích chính đáng (legitimate-interest), khi bất kỳ trường dữ liệu cá nhân nào được thu thập (bao gồm DOB, vốn là dữ liệu cơ bản — không nhạy cảm — theo `data-classification.md` mục 7), thì việc thu thập đồng ý theo AC1 vẫn áp dụng cho nó.

**US-H3: Cơ chế quyền của chủ thể dữ liệu (Điều 9/10)**
Là một khách truy cập mà OmniGem đang nắm giữ dữ liệu, tôi muốn được thông báo, truy cập dữ liệu của mình, rút lại đồng ý, yêu cầu xóa, hạn chế, hoặc phản đối việc xử lý, để các quyền PDPD của tôi được tôn trọng.
- AC1: Tại trang Chính sách Quyền riêng tư, khi một khách muốn thực hiện một quyền, thì nó nêu một kênh cụ thể (ví dụ một địa chỉ email hoặc yêu cầu qua Zalo OA) cho: quyền được thông báo, quyền truy cập, quyền rút lại đồng ý, quyền xóa, quyền hạn chế xử lý, và quyền phản đối.
- AC2: Khi một yêu cầu quyền-chủ-thể-dữ-liệu đến qua kênh đó, lúc nhận được, thì nó được ghi log và định tuyến tới một chủ sở hữu có trách nhiệm (Product Owner/Compliance — placeholder chờ một vai trò được đặt tên) với một SLA nội bộ đã định (SLA chính xác chưa xác định, câu hỏi mở cho Product Owner).
- AC3: Khi một khách rút lại đồng ý hoặc yêu cầu xóa, lúc được thực thi, thì bản ghi của họ được xóa/ẩn danh khỏi kho lead và, nơi khả thi về kỹ thuật, được đánh dấu để gỡ khỏi kho lưu bản ghi chat — tùy thuộc vào bất kỳ nghĩa vụ lưu giữ pháp lý nào đè lên (ví dụ một bản ghi DPIA/kiểm toán đang hoạt động).

### Epic D — AI Chatbot

**US-D1: Tư vấn phong thủy qua chatbot**
Là một khách truy cập, tôi muốn hỏi chatbot gợi ý về đá/màu sắc dựa trên ngày sinh của tôi, để tôi nhận được một đề xuất cá nhân hóa.
- AC1: Tại widget chat, khi một khách cung cấp ngày sinh, thì bot trả về một ánh xạ hành (ngũ hành) và gợi ý các danh mục sản phẩm khớp (research doc §4.1).
- AC2: Khi bot gợi ý sản phẩm, lúc nó phản hồi, thì nó dùng công cụ `search_products(mệnh, loại, giá)` truy vấn dữ liệu CMS trực tiếp (live) — không phải tồn kho bịa ra (research doc §4, yêu cầu RAG).

**US-D2: FAQ niềm tin qua chatbot**
Là một khách truy cập, tôi muốn chatbot trả lời các câu hỏi về tính xác thực/chứng nhận/chính sách đổi trả, để tôi không phải chờ người thật cho các câu hỏi cơ bản.
- AC1: Khi một khách hỏi một câu hỏi dạng FAQ (ví dụ "có phải ngọc thật không", "chính sách đổi trả là gì"), lúc bot phản hồi, thì nó dùng cùng các cam kết như `create-facebook-fanpage.md` §2 (đổi lỗi trong 7 ngày, dịch vụ xâu/đánh bóng trọn đời, chứng nhận lab) — nội dung phải nhất quán giữa Fanpage, chatbot, và copy web.

**US-D3: Human handoff bắt buộc**
Là một khách thể hiện ý định mua, tôi muốn được kết nối với một người thật, để tôi thực sự có thể chốt một giao dịch giá trị cao.
- AC1: Khi khách thể hiện quan tâm tới một sản phẩm cụ thể hoặc yêu cầu mua, lúc được phát hiện, thì bot gọi `handoff_to_zalo()`, tạo một link Zalo, và lưu lead (tên, nhu cầu, mệnh, sản phẩm quan tâm) — research doc §4 "Human handoff bắt buộc."
- AC2: Khi một handoff xảy ra, lúc lead được lưu, thì nó đi vào cùng kho lead như US-C1 (nguồn sự thật duy nhất cho lead), được gắn tag nguồn = "chatbot."
- AC3: Khi bot không thể trả lời một cách tự tin (ví dụ thương lượng giá, đơn đặt riêng), lúc được phát hiện, thì nó mặc định đề nghị handoff thay vì đoán mò.

### Epic E — SEO / Khả năng được tìm thấy (Discoverability)

**US-E1: Dữ liệu có cấu trúc & metadata**
Là doanh nghiệp, tôi muốn có schema markup sản phẩm và tổ chức, để công cụ tìm kiếm tạo ra rich results.
- AC1: Tại bất kỳ trang Product Detail nào, khi được crawl, thì schema.org JSON-LD `Product` hiện diện với name, image, description (research doc §6b).
- AC2: Tại website, khi được crawl, thì schema `Organization` hiện diện toàn website.

**US-E2: Nội dung SEO (blog) — khía cạnh SEO**
Là một khách mua tiềm năng đang tìm hiểu về ngọc bích, tôi muốn có nội dung giáo dục (ví dụ "cách phân biệt ngọc thật"), để tôi tìm thấy OmniGem qua tìm kiếm tự nhiên.
- AC1: Tại CMS, khi một biên tập viên nội dung xuất bản một bài blog, thì nó có thể truy cập tại một URL ổn định, index được, và liên kết nội bộ tới các collections/sản phẩm liên quan.
- *Ghi chú: US-E2 chỉ nêu yêu cầu SEO của blog. Toàn bộ feature blog (trang danh sách, trang chi tiết bài, phân loại, bài liên quan, CTA phễu, schema Article, quy trình xuất bản) được đặc tả đầy đủ ở **Epic I — Blog / Tiếp thị nội dung** bên dưới.*

### Epic I — Blog / Tiếp thị nội dung (Content Marketing)

Lý do nâng thành epic riêng: SEO là kênh tăng trưởng cốt lõi của dự án (G4, giai đoạn P4 Growth), và blog là cỗ máy nội dung chính nuôi kênh đó. Ngoài vai trò SEO, mỗi bài blog cũng là một **điểm vào phễu** (giống triết lý "không ngõ cụt" của US-A2 AC3): bài viết phải dẫn người đọc về collections/sản phẩm và về Zalo/chatbot.

**US-I1: Trang danh sách blog (Blog listing/index)**
Là một khách đang tìm hiểu về ngọc bích/phong thủy, tôi muốn duyệt và lọc các bài viết, để tôi tìm được nội dung liên quan.
- AC1: Tại `/blog`, khi trang tải, thì nó hiển thị danh sách bài đã xuất bản (tiêu đề, ảnh thumbnail, đoạn trích, ngày đăng, danh mục), mới nhất trước, có phân trang, render SSG+ISR và crawl được.
- AC2: Khi khách chọn một danh mục/tag, thì chỉ các bài thuộc phân loại đó hiển thị; mỗi bài liên kết tới trang chi tiết của nó.
- AC3: Khi được crawl, thì trang danh sách và các trang archive theo danh mục có metadata SEO và được đưa vào sitemap.

**US-I2: Trang chi tiết bài viết (Article detail)**
Là một độc giả, tôi muốn đọc một bài viết đầy đủ, dễ đọc trên mobile, để tôi tin tưởng chuyên môn của thương hiệu và muốn tìm hiểu sản phẩm.
- AC1: Tại `/blog/[slug]`, khi trang tải, thì nó render nội dung đầy đủ (rich text, hình ảnh qua Cloudinary, tiêu đề mục để tạo mục lục), kèm tác giả, ngày đăng/cập nhật, thời gian đọc ước tính, danh mục/tag.
- AC2: Khi được crawl, thì schema.org JSON-LD `Article`/`BlogPosting` hiện diện (mở rộng US-E1).
- AC3: Khi bài viết render, thì nó chứa (a) liên kết nội bộ tới collections/sản phẩm liên quan, và (b) một khối CTA cuối bài dẫn vào phễu ("Nhắn Zalo" + "Hỏi AI") — không để bài viết thành ngõ cụt.
- AC4: Khi bài được chia sẻ, thì nó có thẻ OpenGraph/metadata mạng xã hội để hiển thị đẹp khi share.

**US-I3: Phân loại & bài liên quan (Taxonomy & related posts)**
Là một độc giả, tôi muốn thấy các bài liên quan, để tôi đọc tiếp và ở lại lâu hơn.
- AC1: Tại built-in CMS, khi biên tập viên gán danh mục/tag cho bài, thì các trang archive theo danh mục có thể truy cập và index được.
- AC2: Tại cuối trang chi tiết bài, khi render, thì một khối "bài liên quan" (theo danh mục/tag) hiển thị để tăng thời gian trên trang và liên kết nội bộ.

**US-I4: Xuất bản blog bởi người không chuyên kỹ thuật (liên kết Epic F)**
Là biên tập viên nội dung, tôi muốn soạn, xem trước và xuất bản bài mà không cần lập trình viên, để blog luôn có nội dung mới.
- AC1: Tại built-in CMS (React-Admin — xem §4.1), khi biên tập viên tạo/sửa bài, thì có thể lưu nháp (draft, không công khai), xem trước, hẹn giờ/xuất bản và gỡ xuất bản; khi xuất bản, ISR revalidate để bài lên live trong cửa sổ revalidation mà không cần redeploy.
- AC2: Khi bài blog được xử lý, thì nó không chứa PII; quyền và phân tách nhiệm vụ (SoD) tuân theo Epic F (Content Editor read/write nội dung, không chạm kho lead).

*Lưu ý phạm vi theo phase: **hạ tầng blog** (model nội dung trong CMS, trang `/blog` và `/blog/[slug]`, schema Article) thuộc P1/P2; **sản xuất nội dung blog quy mô** (viết bài SEO đều đặn, remarketing) thuộc P4 Growth như roadmap §4.*

### Epic F — Quản lý nội dung (Content Management)

**US-F1: Quản lý catalog cho người không chuyên kỹ thuật**
Là nhân viên cửa hàng, tôi muốn thêm/sửa sản phẩm và giá mà không cần lập trình viên, để catalog luôn cập nhật.
- AC1: Tại trang admin built-in CMS (dashboard React-Admin — xem §4.1), khi nhân viên tạo/sửa một mục sản phẩm (hình ảnh, thông số, giá hoặc "liên hệ để biết giá", tag phong thủy), thì thay đổi phản ánh trên website trực tiếp trong cửa sổ revalidation của CMS (ISR — research doc §3) mà không cần redeploy toàn bộ.
- AC2: Khi trang quản trị được dựng bằng React-Admin (theo quyết định §4.1), thì nhân viên quản lý được cả catalog và lead qua cùng một giao diện admin React thuần.

### Epic G — Analytics

**US-G1: Đo lường phễu (Funnel instrumentation)**
Là doanh nghiệp, tôi muốn đo lường phễu từ page view đến lead, để tôi có thể tối ưu chi tiêu và nội dung.
- AC1: Khi GA4 + Meta Pixel đã được cài đặt, lúc một khách xem một sản phẩm, click Zalo, gửi form lead, hoặc nhận một handoff chatbot, thì mỗi hành vi được kích hoạt như một sự kiện riêng biệt, có tên (`view_product`, `click_zalo`, `submit_lead_form`, `chatbot_handoff`).

## 6. Yêu cầu phi chức năng (Non-Functional Requirements)

- **NFR-1 Hiệu năng/Core Web Vitals:** Vì hình ảnh ngọc bích lớn/độ phân giải cao, các trang vẫn phải đạt ngưỡng "tốt" của Core Web Vitals (LCP, INP, CLS) trên mobile 4G — giá trị mục tiêu chưa xác định với Product Owner/Architect (câu hỏi mở, §8). Yêu cầu hình ảnh được phục vụ qua CDN, lazy-load, responsive (Cloudinary/`next/image`, theo research doc §3).
- **NFR-2 Ưu tiên mobile (Mobile-first):** Phần lớn traffic Việt Nam là mobile (research doc §6b) — mọi luồng (lọc, CTA, chatbot, form) phải dùng được đầy đủ ở các bề rộng viewport mobile, CTA ở phía trên nếp gấp.
- **NFR-3 SEO:** Render SSG/ISR để crawler nhận được HTML đầy đủ (research doc §3); chiến lược render mục tiêu được xác nhận ở giai đoạn thiết kế.
- **NFR-4 Tính sẵn sàng (Availability):** Website marketing — uptime mục tiêu chưa xác định (đề xuất 99.5% làm điểm khởi đầu để Product Owner xác nhận; không phải hệ thống giao dịch nên thấp hơn mức banking-grade là chấp nhận được).
- **NFR-5 Khả năng tiếp cận (Accessibility):** Không có mục tiêu rõ ràng trong research doc — đề xuất baseline WCAG 2.1 AA cho các phần tử form/CTA; câu hỏi mở cho Product Owner (§8).
- **NFR-6 Chi phí (Cost):** Stack phải nằm trong ý định "quảng bá + SEO + tốc độ + chi phí thấp" (research doc §3) — mức dùng Vercel/Cloudinary cần được theo dõi so với một trần ngân sách (số lượng chưa xác định, §8).
- **NFR-7 Bảo vệ dữ liệu:** PII thu thập (tên, số điện thoại, DOB, bản ghi chat) phải được xử lý theo PDPD Việt Nam (Nghị định 13/2023/NĐ-CP) — xem `data-classification.md` và `compliance-scope.md`.

## 7. Chỉ số thành công / KPI (Success Metrics / KPIs)

- Số lượng lead: số lượt gửi form + handoff chatbot mỗi tuần/tháng.
- Tỷ lệ click-through Zalo: số sự kiện `click_zalo` / số phiên trang sản phẩm.
- Tỷ lệ handoff chatbot: số hội thoại dẫn tới `handoff_to_zalo()` / tổng số hội thoại.
- Tăng trưởng traffic tự nhiên: số phiên từ tìm kiếm organic, tháng qua tháng, cộng với xếp hạng từ khóa cho các từ khóa ngọc bích/phong thủy mục tiêu.
- Tỷ lệ pass Core Web Vitals (dữ liệu thực địa) theo NFR-1.
- (Hạ nguồn, không đo trực tiếp trên website được) Tỷ lệ Zalo-đến-chốt-đơn — đòi hỏi báo cáo thủ công từ nhân viên bán hàng vì việc chốt đơn diễn ra ngoài nền tảng.

## 8. Câu hỏi mở / Giả định (phải được giải quyết trước/tại Gate G0 hoặc G1)

1. **Cú pháp prefill deep-link Zalo** được đánh dấu rõ ràng là chưa kiểm chứng trong research doc (§5.3, §Nguồn tham khảo) — phải được xác nhận với tài liệu Zalo Developer hiện hành trước khi build ở giai đoạn Thiết kế.
2. **Nội dung và cơ chế thông báo đồng ý/quyền riêng tư cho việc thu thập DOB** — research doc không nêu rõ UX đồng ý; PRD này giả định một checkbox/thông báo là bắt buộc, chờ xác nhận của compliance-checker (PDPD).
3. **Lựa chọn CMS — ĐÃ CHỐT (không còn là câu hỏi mở).** Research doc ban đầu đưa ra Sanity vs Payload như hai lựa chọn headless CMS. Quyết định cuối: **dùng built-in CMS** — tức tầng quản lý nội dung tự xây bằng React-Admin trên backend/DB của chính dự án (xem §4.1), **không** dùng headless CMS bên thứ ba. Lý do: thống nhất hệ sinh thái React cho dev, nội dung (catalog/blog) và lead nằm chung một backend do dự án kiểm soát, tránh phụ thuộc vendor và tránh thêm một cross-border processor mới. *(Backend/DB cụ thể đứng sau React-Admin vẫn chốt ở giai đoạn Thiết kế.)*
4. **Vendor AI chatbot — ĐÃ CHỐT (không còn là câu hỏi mở).** Research doc ban đầu đề xuất Claude API, với Coze/Botpress/Manychat là con đường no-code (§4). Quyết định cuối: **dùng Gemini API (Google)** cho chatbot. Lưu ý compliance: Gemini API xử lý dữ liệu ngoài Việt Nam, nên nghĩa vụ chuyển dữ liệu xuyên biên giới (PDPD Art. 25) **vẫn áp dụng** — chỉ đổi bên xử lý từ Anthropic sang Google trong hồ sơ Nghĩa vụ B (xem `compliance-scope.md`); Design phase phải xác minh điều khoản lưu giữ/xử lý dữ liệu của Gemini API (bao gồm chính sách dùng dữ liệu để huấn luyện) và điều chỉnh guardrail minimization cho đúng.
5. **Tài khoản người mua** — không có yêu cầu nào nêu về đăng nhập/tài khoản; giả định ngoài phạm vi (NG3) — xác nhận với Product Owner.
6. **Chính sách hiển thị giá — ĐÃ CHỐT (reconcile R1, xem `design/adr/ADR-002`).** Band giá 500k–5tr ⇒ **mặc định hiện giá**; "liên hệ để biết giá" chỉ là override cho hàng độc bản. Ngưỡng cắt là **cấu hình động** (`SiteConfig.price_contact_threshold_vnd`, sửa qua React-Admin, không redeploy) thay vì một mốc VND hard-code — nên không còn chờ con số cứng từ PO.
7. **Kho lead — ĐÃ CHỐT (không còn là câu hỏi mở).** Research doc ban đầu nhắc tới "CRM/Google Sheet" (§4) như thể hoán đổi được cho nhau. Quyết định cuối: **xây CRM built-in** — kho lead là một backend/DB của chính dự án, quản lý qua dashboard React-Admin (xem §4.1). **Bỏ hoàn toàn phương án Google Sheet.** Điều này xử lý trực tiếp rủi ro **R-03** (Google Sheet có kiểm soát truy cập/audit/mã hóa yếu) — CRM built-in cho phép kiểm soát truy cập, audit log và mã hóa đúng mực. *(Backend/DB cụ thể đứng sau CRM vẫn chốt ở giai đoạn Thiết kế.)*
8. **Mục tiêu uptime/accessibility/ngân sách** — không có mục tiêu số nào được nêu trong research doc; các placeholder đề xuất ở trên cần Product Owner phê duyệt.
9. **Hosting xuyên biên giới** — hạ tầng Vercel, Cloudinary, và Gemini API (Google) có thể xử lý/lưu trữ dữ liệu cá nhân của người Việt Nam bên ngoài Việt Nam; các quy tắc chuyển dữ liệu xuyên biên giới của PDPD cần compliance-checker review (xem `compliance-scope.md`).
10. **Chân dung (§3) được suy luận, không lấy từ nghiên cứu người dùng** — được đánh dấu là một giả định; cân nhắc một đợt kiểm chứng nhẹ (ví dụ phỏng vấn khách hàng) nếu thời gian cho phép.

## 9. Phụ thuộc (Dependencies)

- Tài khoản Zalo Official Account (OA) đã xác minh — cần trước công việc deep-link/widget (research doc §5.1).
- API key/tài khoản Gemini API (Google) — cần cho chatbot (theo quyết định Câu hỏi mở 4 đã chốt).
- Built-in CMS: React-Admin (tầng quản trị nội bộ cho catalog/blog + lead dashboard — theo quyết định §4.1 và Câu hỏi mở 3 đã chốt) + backend/API và kho dữ liệu có kiểm soát truy cập (backend/DB cụ thể chốt ở giai đoạn Thiết kế). Không dùng headless CMS bên thứ ba (Sanity/Payload).
- Tài khoản Cloudinary (hoặc CDN/pipeline ảnh tương đương).
- Domain + project Vercel.
- Property GA4 + Meta Pixel + thiết lập theo dõi chuyển đổi Zalo.
- Tài sản thương hiệu (logo, hình bìa) — theo `documents/create-facebook-fanpage.md` §3, tái sử dụng được giữa Fanpage và website.
- CRM built-in làm đích lưu trữ lead (backend/DB của dự án, quản lý qua React-Admin — theo Câu hỏi mở 7 đã chốt). Không dùng Google Sheet.

## 10. Checklist Gate G0 đề xuất (theo tiêu chí G0 của `sdlc-graph.yaml`)

- [ ] Luận chứng kinh doanh được phê duyệt — bản nháp ở trên, chờ Product Owner ký duyệt.
- [ ] Rủi ro được xếp hạng — xem `risk-register.md`.
- [ ] Phạm vi compliance (SBV/PCI/ISO) được xác định — xem `compliance-scope.md` (đề xuất: PDPD có, PCI/SBV ngoài phạm vi, ISO tùy chọn) — chờ compliance-checker xác nhận.
- [ ] Dữ liệu được phân loại (có dữ liệu thẻ/PII nào không?) — xem `data-classification.md` (đề xuất: PII có — tên/số điện thoại/DOB/bản ghi chat; dữ liệu thẻ — không).
- [ ] Không có rào cản pháp lý — chờ compliance-checker review các hạng mục đồng ý/xuyên biên giới PDPD (Câu hỏi mở 2, 9).
