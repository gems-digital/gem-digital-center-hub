# OmniGem Gallery — Nghiên cứu Hệ thống 1 (Web quảng bá)

> Nghiên cứu chiến lược & kỹ thuật cho website trưng bày trang sức ngọc / đá quý tự nhiên, tích hợp Chatbot AI và Zalo để điều hướng khách hàng. Tài liệu này là bản gợi ý ban đầu để review, không phải bản kế hoạch chốt.

---

## 1. Định vị chiến lược — vì sao "Gallery" là lựa chọn đúng

| Yếu tố | Web bán hàng thường (e-commerce) | **OmniGem Gallery** |
|---|---|---|
| Tâm lý khách | "Săn giá rẻ, so sánh" | "Chiêm ngưỡng, tin tưởng, sở hữu tác phẩm" |
| Trọng tâm | Giỏ hàng, thanh toán ngay | Câu chuyện sản phẩm, cảm xúc, tư vấn |
| Rào cản đơn giá cao | Rất cao (khách ngại bấm "mua" 20–200tr) | Thấp — điều hướng sang **tư vấn 1-1** |
| Vai trò web | Chốt đơn | **Tạo niềm tin + phễu dẫn về Chatbot/Zalo** |

**Insight cốt lõi:** Mặt hàng vòng ngọc / mặt dây / tượng điêu khắc giá trị cao **không chốt đơn trên web**. Khách cần: (1) tin đây là ngọc thật, (2) tin người bán uy tín, (3) được tư vấn hợp mệnh/phong thủy. Web **không phải cửa hàng — mà là phòng trưng bày + máy tạo niềm tin**, còn việc chốt đơn được đẩy về Zalo/Chatbot nơi có tương tác con người.

> "Gallery" đúng vì nó hạ kỳ vọng "mua ngay" và nâng kỳ vọng "chiêm ngưỡng & được tư vấn".

---

## 2. Kiến trúc thông tin (IA) — Sitemap đề xuất

```
OmniGem Gallery
├── Trang chủ (Hero cinematic + tuyển chọn tác phẩm nổi bật)
├── Bộ sưu tập (Collections)
│   ├── Vòng ngọc (Jade Bangles)
│   ├── Mặt dây chuyền (Pendants)
│   ├── Tượng điêu khắc (Sculptures)
│   └── Theo mệnh / Ngũ hành (filter phong thủy) ★ điểm khác biệt
├── Chi tiết tác phẩm (Product Detail — trọng tâm chuyển đổi)
├── Câu chuyện thương hiệu (Vì sao tin — nguồn gốc, giám định)
├── Kiến thức ngọc (Blog/SEO — "cách phân biệt ngọc thật")
├── Kiểm định & Cam kết (Certificate showcase)
├── Liên hệ / Tư vấn (Zalo, Hotline, Form)
└── [Floating] Chatbot AI + nút Zalo (mọi trang)
```

**Trang quan trọng nhất = Chi tiết tác phẩm.** Cấu trúc đề xuất:

1. Gallery ảnh/video độ phân giải cao (zoom soi vân, ánh sáng xuyên) — *bằng chứng ngọc thật*
2. Tên + mã tác phẩm (tính "độc bản")
3. Thông số: loại ngọc, kích thước, nguồn gốc, chứng nhận
4. **Khối phong thủy**: hợp mệnh nào, ý nghĩa
5. Giá (hoặc "Liên hệ tư vấn" — xem mục 6)
6. **CTA kép**: `Nhắn Zalo về tác phẩm này` + `Hỏi AI về tác phẩm`
7. Social proof: feedback khách, video khách nhận hàng

---

## 3. Kiến trúc kỹ thuật đề xuất

Mục tiêu là **quảng bá + SEO + tốc độ + chi phí thấp**, không phải giao dịch phức tạp.

**Khuyến nghị chính: Next.js (App Router) + Headless CMS + Vercel**

| Tầng | Lựa chọn đề xuất | Lý do |
|---|---|---|
| Frontend | **Next.js + React + TypeScript** | SSG/ISR cho SEO tốt — crawler nhận HTML đầy đủ, tải nhanh; ISR cập nhật sản phẩm/giá nhanh mà không rebuild toàn site |
| Styling | Tailwind CSS + Framer Motion | Hiệu ứng "gallery" mượt, thẩm mỹ cao cấp |
| CMS | **Sanity** hoặc **Payload CMS** | Chủ shop tự thêm/sửa tác phẩm không cần code |
| Ảnh/Video | Cloudinary hoặc `next/image` + CDN | Ảnh ngọc rất nặng → phải tối ưu, lazy-load, zoom |
| Hosting | **Vercel** | Deploy tự động, CDN toàn cầu |
| Chatbot AI | Claude API (Anthropic) hoặc widget | Xem mục 4 |
| Zalo | Zalo OA + Zalo Chat Widget | Xem mục 5 |
| Analytics | GA4 + Meta Pixel + Zalo conversion | Đo phễu |

**Phương án nhẹ hơn (ngân sách tối thiểu / chủ shop tự vận hành):** Astro + Tailwind (tĩnh, siêu nhanh, SEO tốt) hoặc WordPress + Elementor.

> ISR đặc biệt hợp với catalog thay đổi thường xuyên (thêm hàng, đổi giá): revalidate theo lịch hoặc webhook thay vì rebuild toàn bộ — vừa SEO tốt vừa tiết kiệm chi phí so với SSR.

---

## 4. Tích hợp Chatbot AI — "chuyên gia tư vấn ngọc ảo"

Chatbot không chỉ trả lời FAQ, mà đóng vai **dẫn khách xuống phễu**.

**3 nhiệm vụ:**
1. **Tư vấn phong thủy/mệnh** — hỏi ngày sinh → gợi ý loại ngọc/màu hợp ngũ hành → dẫn tới sản phẩm cụ thể
2. **Giải đáp niềm tin** — "làm sao biết ngọc thật?", "có kiểm định không?", "đổi trả thế nào?"
3. **Điều hướng chốt** — khi khách quan tâm 1 tác phẩm → xin Zalo → **chuyển sang người thật**

**Kiến trúc kỹ thuật gợi ý:**

```
User → Chat Widget (web)
     → API route (Next.js)
     → Claude API (system prompt = "chuyên gia ngọc OmniGem")
         + Tool: search_products(mệnh, loại, giá)  → truy vấn CMS
         + Tool: handoff_to_zalo()  → tạo link Zalo + lưu lead
     → Trả lời + thẻ sản phẩm + nút "Nhắn Zalo"
```

- **RAG nhẹ**: nạp catalog + FAQ + kiến thức ngọc vào context/vector store → bot trả lời đúng hàng đang có.
- **Human handoff bắt buộc**: đơn giá cao cần chốt qua người thật — bot phải biết khi nào nhường.
- Thu **lead**: mỗi hội thoại lưu (tên, nhu cầu, mệnh, sản phẩm quan tâm) → CRM/Google Sheet.

*(Muốn nhanh, không tự code: dùng Coze / Botpress / Manychat có widget sẵn. Tự host qua Claude API kiểm soát chất lượng tư vấn tốt hơn.)*

---

## 5. Tích hợp Zalo — kênh chốt đơn chính tại Việt Nam

1. **Zalo OA (Official Account)** — tài khoản doanh nghiệp xác thực (tạo niềm tin).
2. **Zalo Chat Widget** — nút chat nổi góc phải mọi trang; nhúng qua script chính thức của Zalo, khách bấm mở Zalo (app trên điện thoại, Zalo Web trên desktop).
3. **Deep-link theo sản phẩm** — nút "Nhắn Zalo về tác phẩm này" mở sẵn tin nhắn kèm mã SP để nhân viên biết ngay khách hỏi món nào. *(Tham số prefill message: kiểm tra cú pháp mới nhất trong Zalo Developer docs — xem citation.)*
4. **Zalo ZNS** (Notification Service) — tin chăm sóc/xác nhận (giai đoạn sau).
5. **Đo lường** — gắn event `click_zalo` vào GA4/Pixel để biết trang/sản phẩm nào tạo lead tốt.

**Nguyên tắc điều hướng:** Web tạo niềm tin & cảm xúc → mọi con đường dẫn về Zalo hoặc Chatbot. Chatbot lọc & hâm nóng → Zalo chốt bằng người thật.

---

## 6. Chiến lược Marketing gắn với web

**a. Giá — hiển thị hay "Liên hệ"?**
- Hàng phổ thông (vài triệu): **hiện giá** → giảm ma sát.
- Hàng cao cấp/độc bản (vài chục–vài trăm triệu): **"Liên hệ tư vấn"** → tạo cảm giác quý hiếm + buộc khách vào hội thoại. Khuyến nghị **mô hình lai** theo phân khúc.

**b. SEO (nguồn traffic miễn phí dài hạn) — bắt buộc:**
- Blog kiến thức: *"cách phân biệt ngọc phỉ thật giả"*, *"vòng ngọc hợp mệnh Mộc"*, *"ý nghĩa mặt Phật Di Lặc"*.
- Schema.org `Product` + `Organization` → rich results.
- Tốc độ + mobile-first (đa số khách VN xem qua điện thoại).

**c. Phễu chuyển đổi hoàn chỉnh:**

```
Facebook Ads/Fanpage + SEO (nhận biết)
        ↓
OmniGem Gallery (chiêm ngưỡng + niềm tin)
        ↓
Chatbot AI (tư vấn mệnh, lọc nhu cầu) ── hoặc ──→ Zalo (người thật)
        ↓
Chốt đơn qua Zalo (video thực tế, cọc, ship COD)
        ↓
Chăm sóc lại (ZNS, remarketing) → khách quay lại/giới thiệu
```

**d. Yếu tố niềm tin phải nổi bật:** chứng nhận kiểm định, video soi ngọc dưới đèn, feedback thật, cam kết đổi trả/bảo hành. *(Tái sử dụng nội dung đã có trong `create-facebook-fanpage.md`.)*

---

## 7. Lộ trình triển khai (phased)

| Giai đoạn | Hạng mục | Kết quả |
|---|---|---|
| **P0 — Nền tảng** (tuần 1–2) | Setup Next.js + CMS + design system, sitemap, deploy Vercel | Web khung chạy được |
| **P1 — Gallery** (tuần 2–4) | Trang chủ, Collections, Product Detail, ảnh/video tối ưu, SEO cơ bản | Web trưng bày hoàn chỉnh |
| **P2 — Điều hướng** (tuần 4–5) | Zalo widget + deep-link, form lead, Analytics/Pixel | Bắt đầu ra lead |
| **P3 — Chatbot AI** (tuần 5–7) | Widget chat + Claude API + RAG catalog + human handoff | Tư vấn tự động 24/7 |
| **P4 — Tăng trưởng** | Blog SEO, remarketing, A/B test CTA, ZNS chăm sóc | Traffic & chuyển đổi tăng |

---

## Tóm tắt

**OmniGem Gallery = Phòng trưng bày số + Máy tạo niềm tin + Phễu dẫn về Zalo/Chatbot**, không phải sàn TMĐT. Ba trụ cột: (1) trình bày tác phẩm đẳng cấp để tạo cảm xúc & niềm tin, (2) SEO/nội dung hút traffic tự nhiên, (3) Chatbot AI lọc nhu cầu + Zalo chốt bằng người thật.

---

## Nguồn tham khảo (Citations)

**Nội bộ dự án:**
- `documents/create-facebook-fanpage.md` — guide dựng Fanpage/Messenger/Zalo hotline; nguồn cho định vị thương hiệu, kịch bản tư vấn theo mệnh, cam kết đổi trả/kiểm định được tái sử dụng ở mục 1, 5, 6.

**Zalo (tích hợp widget & OA) — mục 5:**
- Zalo Developers — Tạo Widget Chat: https://developers.zalo.me/docs/social/zalo-chat-widget
- Zalo Official Account: https://zalo.me/officialaccount?openChat=true
- Hướng dẫn nhúng Zalo Chat Widget vào website (FGC): https://www.fgc.vn/en/instructions-for-integrating-widget-zalo-chat-into-website-free.html
- Tích hợp AI Chatbot với Zalo OA (NOKASOFT): https://nokasoft.com/how-to-integrate-an-ai-chatbot-with-zalo-oa-a-complete-guide/

**Next.js / rendering & SEO — mục 3:**
- Next.js Learn — SEO: Rendering Strategies: https://nextjs.org/learn/seo/rendering-strategies
- Next.js for Ecommerce: Architecture, SEO, and Real Builds (Naturaily): https://naturaily.com/blog/nextjs-ecommerce
- SSR vs SSG in Next.js (Strapi): https://strapi.io/blog/ssr-vs-ssg-in-nextjs-differences-advantages-and-use-cases

**Chatbot AI — mục 4:**
- Claude API / Anthropic (nền tảng cho chatbot tư vấn tự host): https://docs.anthropic.com

**Ghi chú về tính xác thực:** Các mục 1, 2, 6, 7 (định vị, IA, chiến lược marketing, lộ trình) là **phân tích chiến lược** dựa trên context dự án và thực tiễn ngành, không trích từ nguồn ngoài. Các mục 3, 4, 5 (kỹ thuật) được đối chiếu với tài liệu ở trên; **cú pháp deep-link prefill message của Zalo cần kiểm tra lại trực tiếp trong Zalo Developer docs trước khi triển khai** vì có thể thay đổi theo phiên bản.
