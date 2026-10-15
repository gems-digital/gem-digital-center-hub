# OmniGem Gallery — Tài liệu Use Case (Phân rã chức năng hệ thống)

Trạng thái: **ĐÃ CHỐT (APPROVED)** — chốt ngày 2026-10-15. Nguồn: `omni-web/prd/PRD.md` (các Epic A, B, C, D, E, F, G, H, I).
Mục đích: cho cái nhìn tổng quan *ai (actor) làm gì (use case) với hệ thống*, phân rã theo **2 boundary** (ranh giới hệ thống) và chi tiết hóa từng use case lớn thành use case con.

> Ghi chú đồng bộ: bám theo PRD đã cập nhật — chatbot dùng **Gemini API**, nội dung quản lý qua **built-in CMS (React-Admin)**, lead lưu ở **CRM built-in**.

---

## 1. Phạm vi & triết lý

OmniGem Gallery = showroom số + cỗ máy niềm tin + phễu dẫn về Zalo/Chatbot (không phải sàn TMĐT). Phần lớn use case của khách đều **kết thúc bằng một lối vào phễu** (Zalo/chatbot/form). Mọi use case thu thập dữ liệu cá nhân đều **include** bước cấp đồng ý (consent).

## 2. Hai Boundary (ranh giới hệ thống)

| Boundary | Hệ thống kỹ thuật | Actor tương tác | Đặc tính |
|---|---|---|---|
| **Boundary 1 — Customer-facing** | Website công khai (Next.js, SSG/ISR) + widget | Khách truy cập | Công khai, không đăng nhập, SEO-first, không chứa PII ở trang tĩnh |
| **Boundary 2 — Internal** | Trang quản trị (React-Admin): built-in CMS + CRM | Biên tập viên, NV kinh doanh, Compliance/DPO | Có xác thực, phân quyền (SoD), chạm PII, audit log |

## 3. Danh sách Actor

### Actor chính (Primary)
| Actor | Boundary | Mô tả | Persona (PRD §3) |
|---|---|---|---|
| **Khách truy cập** (Visitor) | 1 | Duyệt web, đọc blog, hỏi chatbot, để lại lead | Khách phong thủy · Khách mua quà · Nhà sưu tầm |
| **Biên tập viên nội dung** (Content Editor) | 2 | Quản lý catalog & blog qua CMS | — |
| **Nhân viên kinh doanh** (Sales Staff) | 2 | Tiếp nhận & chốt lead trên Zalo, truy cập CRM | — |
| **Compliance / DPO** | 2 | Xử lý DSR, export bản ghi đồng ý | — |

### Actor phụ / hệ thống ngoài (Secondary / External)
Gemini API (Google) · Zalo (OA + widget) · GA4 / Meta Pixel · Cloudinary · (Built-in CMS/CRM là nội bộ, không phải hệ thống ngoài).

---

## 4. Sơ đồ Use Case TỔNG QUAN (2 boundary)

```mermaid
flowchart TB
    visitor(("Khách<br/>truy cập")):::actor
    editor(("Biên tập<br/>nội dung")):::actor
    sales(("NV Kinh<br/>doanh")):::actor
    dpo(("Compliance<br/>/ DPO")):::actor

    subgraph CF["BOUNDARY 1 — CUSTOMER-FACING (Next.js public)"]
      direction TB
      a_browse(["UC-01..04<br/>Duyệt & khám phá sản phẩm"]):::uc
      a_blog(["UC-05..07<br/>Đọc blog"]):::uc
      a_funnel(["UC-08..10<br/>Liên hệ qua phễu"]):::uc
      a_bot(["UC-11..13<br/>Tư vấn chatbot AI"]):::uc
      a_priv(["UC-14..16<br/>Quyền riêng tư & DSR"]):::uc
      a_track(["UC-21<br/>Phát sinh sự kiện analytics (cross-cutting)"]):::ucx
    end

    subgraph INT["BOUNDARY 2 — INTERNAL (React-Admin: built-in CMS + CRM)"]
      direction TB
      i_cat(["UC-17<br/>Quản lý catalog"]):::uc
      i_blog(["UC-18<br/>Quản lý blog"]):::uc
      i_crm(["UC-19<br/>Quản lý lead (CRM)"]):::uc
      i_dsr(["UC-20<br/>Xử lý DSR back-office"]):::uc
    end

    visitor --> a_browse & a_blog & a_funnel & a_bot & a_priv
    editor --> i_cat & i_blog
    sales --> i_crm
    dpo --> i_dsr & i_crm

    a_funnel -. "lead đổ về" .-> i_crm
    a_bot -. "handoff tạo lead" .-> i_crm
    a_priv -. "DSR chuyển tiếp" .-> i_dsr
    i_cat -. "nội dung hiển thị ra" .-> a_browse
    i_blog -. "bài hiển thị ra" .-> a_blog

    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef ucx fill:#ede7f6,stroke:#6a1b9a,stroke-dasharray:4 3;
```

**Đọc sơ đồ:** actor nằm ngoài boundary. Mũi tên nét đứt giữa 2 boundary thể hiện dòng dữ liệu (lead từ customer-facing đổ về CRM internal; nội dung từ CMS internal hiển thị ra public). UC-21 vẽ nét đứt vì là chức năng cross-cutting (đính kèm nhiều use case khác).

---

## 5. Phân rã Use Case (catalog phân cấp) + sơ đồ chi tiết

Mỗi use case lớn dưới đây được phân rã thành use case con (UC-xx.y) kèm một sơ đồ chi tiết thể hiện quan hệ `«include»` (bắt buộc kèm theo) / `«extend»` (mở rộng có điều kiện).

### BOUNDARY 1 — CUSTOMER-FACING

#### UC-01..04 — Duyệt & khám phá sản phẩm (Epic A)

```mermaid
flowchart TB
    v(("Khách")):::actor
    subgraph B1A["Duyệt & khám phá"]
      u1(["UC-01 Xem trang chủ"]):::uc
      u11(["UC-01.1 Xem hero/featured"]):::sub
      u12(["UC-01.2 Điều hướng tới chi tiết SP"]):::sub

      u2(["UC-02 Duyệt collections & lọc"]):::uc
      u21(["UC-02.1 Chọn danh mục"]):::sub
      u22(["UC-02.2 Lọc theo mệnh (ngũ hành)"]):::sub
      u23(["UC-02.3 Empty state → CTA phễu"]):::sub

      u3(["UC-03 Xem chi tiết sản phẩm"]):::uc
      u31(["UC-03.1 Xem gallery ảnh/video zoom"]):::sub
      u32(["UC-03.2 Xem thông số/xuất xứ/chứng nhận"]):::sub
      u33(["UC-03.3 Xem khối độ khớp phong thủy"]):::sub
      u34(["UC-03.4 Xem social proof"]):::sub
      u35(["UC-03.5 Kích hoạt CTA (Zalo/Hỏi AI)"]):::sub

      u4(["UC-04 Xem brand & certification"]):::uc
      u41(["UC-04.1 Đọc câu chuyện nguồn gốc"]):::sub
      u42(["UC-04.2 Xem giải thích kiểm định lab"]):::sub
    end
    v --> u1 & u2 & u3 & u4
    u1 --> u11 & u12
    u2 --> u21 & u22
    u2 -. extend .-> u23
    u3 --> u31 & u32 & u33 & u35
    u3 -. extend .-> u34
    u4 --> u41 & u42
    u12 -. tới .-> u3
    u23 -. tới .-> u35
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

#### UC-05..07 — Đọc blog (Epic I)

```mermaid
flowchart TB
    v(("Khách")):::actor
    subgraph B1B["Đọc blog"]
      u5(["UC-05 Duyệt danh sách blog"]):::uc
      u51(["UC-05.1 Xem danh sách phân trang"]):::sub
      u52(["UC-05.2 Lọc theo danh mục/tag"]):::sub

      u6(["UC-06 Đọc bài viết"]):::uc
      u61(["UC-06.1 Đọc nội dung + mục lục"]):::sub
      u62(["UC-06.2 Xem liên kết SP/collection liên quan"]):::sub
      u63(["UC-06.3 CTA phễu cuối bài"]):::sub
      u64(["UC-06.4 Chia sẻ mạng xã hội (OpenGraph)"]):::sub

      u7(["UC-07 Bài theo danh mục & liên quan"]):::uc
      u71(["UC-07.1 Xem archive danh mục"]):::sub
      u72(["UC-07.2 Xem bài liên quan"]):::sub
    end
    v --> u5 & u6 & u7
    u5 --> u51 & u52
    u6 --> u61 & u62 & u64
    u6 -. extend .-> u63
    u7 --> u71 & u72
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

#### UC-08..10 — Liên hệ qua phễu (Epic B, C)

```mermaid
flowchart TB
    v(("Khách")):::actor
    zalo(("Zalo")):::sys
    subgraph B1C["Liên hệ qua phễu"]
      u8(["UC-08 Nhắn Zalo về 1 SP (deep-link)"]):::uc
      u9(["UC-09 Widget Zalo chung"]):::uc
      u10(["UC-10 Gửi form tư vấn/lead"]):::uc
      u101(["UC-10.1 Nhập thông tin liên hệ"]):::sub
      u102(["UC-10.2 Cấp đồng ý"]):::sub
      u103(["UC-10.3 Xác thực & lưu lead vào CRM"]):::sub
      u104(["UC-10.4 Lỗi lưu → fallback Zalo"]):::sub
      u15(["UC-15 Cấp đồng ý (Art.11)"]):::ucshared
    end
    v --> u8 & u9 & u10
    u8 -. tới .-> zalo
    u9 -. tới .-> zalo
    u10 --> u101 & u103
    u10 -. extend .-> u104
    u102 -. include .-> u15
    u10 --> u102
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef sys fill:#eceff1,stroke:#455a64;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef ucshared fill:#e8f5e9,stroke:#2e7d32;
```

#### UC-11..13 — Tư vấn chatbot AI (Epic D)

```mermaid
flowchart TB
    v(("Khách")):::actor
    gemini(("Gemini<br/>API")):::sys
    sales(("NV Kinh doanh")):::actor
    subgraph B1D["Chatbot AI"]
      u11(["UC-11 Tư vấn phong thủy theo DOB"]):::uc
      u111(["UC-11.1 Nhập DOB → map ngũ hành"]):::sub
      u112(["UC-11.2 Gợi ý SP (search_products, RAG)"]):::sub

      u12(["UC-12 Hỏi FAQ niềm tin"]):::uc
      u121(["UC-12.1 Hỏi chứng nhận/đổi trả (RAG FAQ)"]):::sub

      u13(["UC-13 Chuyển giao người thật (handoff)"]):::uc
      u131(["UC-13.1 Phát hiện ý định mua"]):::sub
      u132(["UC-13.2 Tạo Zalo deep-link"]):::sub
      u133(["UC-13.3 Tạo lead source=chatbot"]):::sub

      u15(["UC-15 Cấp đồng ý (lượt PII đầu)"]):::ucshared
      ubot(["(hỗ trợ) Bot challenge + sessionId ký + scrub PII"]):::ucx
    end
    v --> u11 & u12 & u13
    u11 --> u111 & u112
    u12 --> u121
    u13 --> u131 & u132 & u133
    u11 -. include .-> u15
    u13 -. include .-> u15
    u11 & u12 & u13 -. include .-> ubot
    u112 & u121 -. tới .-> gemini
    u133 -. tới .-> sales
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef sys fill:#eceff1,stroke:#455a64;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef ucshared fill:#e8f5e9,stroke:#2e7d32;
    classDef ucx fill:#ede7f6,stroke:#6a1b9a,stroke-dasharray:4 3;
```

#### UC-14..16 — Quyền riêng tư & DSR (Epic H, phía khách)

```mermaid
flowchart TB
    v(("Khách")):::actor
    subgraph B1E["Quyền riêng tư & dữ liệu"]
      u14(["UC-14 Đọc chính sách quyền riêng tư"]):::uc
      u15(["UC-15 Cấp đồng ý (Art.11, tái lập được)"]):::ucshared
      u16(["UC-16 Gửi yêu cầu DSR (Art.9/10)"]):::uc
      u161(["UC-16.1 Chọn loại quyền (access/withdraw/erasure/restrict/object/rectify/inform)"]):::sub
      u162(["UC-16.2 Nhập định danh (SĐT/Zalo/email)"]):::sub
      u163(["UC-16.3 Xác minh định danh (OTP tới kênh đã lưu)"]):::sub
    end
    v --> u14 & u16
    u16 --> u161 & u162
    u16 -. include .-> u163
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef ucshared fill:#e8f5e9,stroke:#2e7d32;
```

### BOUNDARY 2 — INTERNAL (React-Admin: built-in CMS + CRM)

#### UC-17 — Quản lý catalog (Epic F)

```mermaid
flowchart TB
    e(("Biên tập<br/>nội dung")):::actor
    subgraph B2A["Quản lý catalog (CMS)"]
      u17(["UC-17 Quản lý catalog"]):::uc
      u171(["UC-17.1 Tạo/sửa sản phẩm (ảnh, thông số)"]):::sub
      u172(["UC-17.2 Gán tag mệnh/phong thủy"]):::sub
      u173(["UC-17.3 Đặt giá / 'liên hệ để biết giá'"]):::sub
      u174(["UC-17.4 Publish → ISR revalidate"]):::sub
    end
    e --> u17
    u17 --> u171 & u172 & u173 & u174
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

#### UC-18 — Quản lý blog (Epic I)

```mermaid
flowchart TB
    e(("Biên tập<br/>nội dung")):::actor
    subgraph B2B["Quản lý blog (CMS)"]
      u18(["UC-18 Quản lý bài blog"]):::uc
      u181(["UC-18.1 Soạn/sửa bản nháp"]):::sub
      u182(["UC-18.2 Gán danh mục/tag"]):::sub
      u183(["UC-18.3 Xem trước (preview)"]):::sub
      u184(["UC-18.4 Hẹn giờ/Publish/Unpublish → ISR"]):::sub
    end
    e --> u18
    u18 --> u181 & u182 & u183 & u184
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

#### UC-19 — Quản lý lead / CRM (Epic C/D phía nội bộ)

```mermaid
flowchart TB
    s(("NV Kinh<br/>doanh")):::actor
    subgraph B2C["Quản lý lead (CRM built-in)"]
      u19(["UC-19 Truy cập & quản lý lead"]):::uc
      u191(["UC-19.1 Xem danh sách lead (least-privilege)"]):::sub
      u192(["UC-19.2 Cập nhật trạng thái/ghi chú"]):::sub
      u193(["UC-19.3 Export lead (có audit log)"]):::sub
    end
    s --> u19
    u19 --> u191 & u192 & u193
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

#### UC-20 — Xử lý DSR back-office (Epic H phía nội bộ)

```mermaid
flowchart TB
    dpo(("Compliance<br/>/ DPO")):::actor
    subgraph B2D["Xử lý DSR (maker–checker)"]
      u20(["UC-20 Xử lý yêu cầu DSR"]):::uc
      u201(["UC-20.1 Tiếp nhận + gán owner + bật SLA"]):::sub
      u202(["UC-20.2 Maker: chuẩn bị hành động xóa/rút"]):::sub
      u203(["UC-20.3 Checker: DPO phê duyệt"]):::sub
      u204(["UC-20.4 Thực thi + override lưu giữ pháp lý"]):::sub
      u205(["UC-20.5 Export bản ghi đồng ý (bằng chứng Art.11)"]):::sub
    end
    dpo --> u20
    u20 --> u201 & u202 & u203 & u204 & u205
    u202 -. precede .-> u203
    u203 -. precede .-> u204
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

### Cross-cutting — UC-21 Đo lường analytics (Epic G)

UC-21 không thuộc riêng một actor; nó được `«include»` bởi các use case tương tác then chốt ở Boundary 1 (sau khi có đồng ý analytics), phát sinh các sự kiện **không chứa PII**:

| Sự kiện | Phát sinh từ |
|---|---|
| `view_product` | UC-03 |
| `click_zalo` | UC-08, UC-09 |
| `submit_lead_form` | UC-10 |
| `chatbot_handoff` | UC-13 |

---

## 6. Bảng tổng hợp quan hệ include / extend

| Quan hệ | Chi tiết |
|---|---|
| `«include»` | UC-10.2, UC-11, UC-13 **→ UC-15** (cấp đồng ý; không consent thì không xử lý PII) |
| `«include»` | UC-13 **→ lõi lưu lead của UC-10** (lead chatbot & form vào cùng CRM — nguồn sự thật duy nhất) |
| `«include»` | UC-11/12/13 **→ bảo vệ phiên chatbot** (bot challenge + sessionId ký + scrub PII trước khi gọi Gemini) |
| `«include»` | UC-16 **→ UC-16.3** xác minh định danh (OTP) trước khi UC-20 thực thi |
| `«include»` | UC-01..03, UC-08..10, UC-13 **→ UC-21** phát sinh sự kiện analytics |
| `«extend»` | UC-02 **→ UC-02.3** empty state → CTA phễu (chỉ khi không có kết quả) |
| `«extend»` | UC-03 **→ UC-03.4** social proof (chỉ khi có media chứng thực) |
| `«extend»` | UC-06 **→ UC-06.3** CTA phễu cuối bài |
| `«extend»` | UC-10 **→ UC-10.4** fallback Zalo (chỉ khi CRM lỗi) |

---

## 7. Hai đặc tả Use Case mẫu (template để mở rộng)

> Các UC khác hiện ở mức danh mục + phân rã sơ đồ. Có thể mở rộng thành đặc tả đầy đủ theo template dưới.

### UC-10 — Gửi form tư vấn/lead

| Trường | Nội dung |
|---|---|
| **Boundary** | 1 — Customer-facing |
| **Actor chính** | Khách truy cập |
| **Mục tiêu** | Để lại thông tin liên hệ + nhu cầu để nhân viên follow up |
| **Tiền điều kiện** | Trang Liên hệ đã tải; chính sách quyền riêng tư (có version) hiển thị |
| **Hậu điều kiện (thành công)** | 1 Lead + 1 ConsentRecord ghi nguyên tử vào CRM; bắn `submit_lead_form` (không PII) |

**Luồng chính:** (UC-10.1) nhập tên/SĐT-Zalo/DOB/nhu cầu → tick đồng ý gắn version chính sách → submit → kiểm origin/CSRF → rate-limit → xác thực schema → **«include» UC-15** ghi ConsentRecord trong cùng transaction → (UC-10.3) ghi Lead (`source="lead_form"`, PII mã hóa) → trả 201 → bắn analytics.
**Ngoại lệ:** thiếu/sai trường → báo lỗi tại chỗ; `consent≠true` → từ chối, không ghi gì; (UC-10.4) CRM lỗi → 503 + CTA fallback Zalo, không commit một phần.
**Quy tắc:** Art.11 đồng ý bắt buộc cho mọi PII (gồm DOB); không PII trong analytics; không commit một phần.

### UC-13 — Chuyển giao người thật (chatbot handoff)

| Trường | Nội dung |
|---|---|
| **Boundary** | 1 — Customer-facing (lead đổ sang Boundary 2) |
| **Actor chính** | Khách truy cập → Nhân viên kinh doanh (tiếp nhận trên Zalo) |
| **Mục tiêu** | Kết nối khách có ý định mua tới người thật để chốt giao dịch giá trị cao |
| **Tiền điều kiện** | Phiên chatbot hợp lệ (bot challenge, sessionId ký HMAC); đã cấp đồng ý nếu lượt mang PII |
| **Hậu điều kiện (thành công)** | Zalo deep-link tạo ra; 1 lead `source="chatbot"` lưu CRM, liên kết ConsentRecord phiên |

**Luồng chính:** (UC-13.1) chatbot phát hiện ý định mua → gọi tool `handoff_to_zalo()` → (UC-13.2) build Zalo deep-link → **«include» lõi UC-10** (UC-13.3) tạo Lead (`source="chatbot"`, gắn ConsentRecord, rate-limit/dedupe theo phiên) → bắn `chatbot_handoff` → hiển thị link cho khách.
**Ngoại lệ:** bot không tự tin (giá/đơn custom) → đề nghị handoff thay vì đoán; Gemini API lỗi/timeout hoặc chạm trần chi tiêu → fallback tin nhắn + CTA Zalo.
**Quy tắc:** handoff bắt buộc trước mọi cam kết mua; lead chatbot & form vào cùng CRM; chống spam lead do injection.
