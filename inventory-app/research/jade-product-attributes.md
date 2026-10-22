# Đặc điểm & thông số sản phẩm đá ngọc (cẩm thạch/phỉ thúy) — Nghiên cứu cho app quản lý kho

> Mục đích: Xác định bộ trường nhập liệu cho app quản lý kho nội bộ, cho 4 loại sản phẩm:
> (1) Vòng tay — gồm **vòng cẩm thạch/bangle** (niềng liền khối) & **vòng charm/lắc** (hạt + charm, dây thun/dây kim loại);
> (2) Nhẫn; (3) Chuỗi hạt (vòng chuỗi); (4) Mặt dây (mặt dây chuyền).
> Tầm giá 500k–5tr VND. Khách quan tâm phong thủy/ngũ hành.
>
> ⚠️ **Lưu ý độ tin cậy:** Web search bị giới hạn nhiều về nguồn tiếng Việt chuyên sâu (nhiều kết quả trả về là shop quốc tế/Walmart/eBay). Các con số quy đổi "ly = mm", bảng size bangle, dải size nhẫn đã được xác minh từ nguồn. Phần thuật ngữ chuyên ngành phỉ thúy (chủng/nước/màu) bổ sung từ kiến thức ngành — **nên để người bán rà lại từ ngữ niêm yết thực tế của shop.**

---

## 1. Giải thích đơn vị "ly" và "ni" (QUAN TRỌNG)

### "ly" (còn viết "li") = **1 milimét (mm)**
- Trong nghề đá quý/trang sức VN, **1 ly = 1 mm**. Đây là đơn vị khẩu ngữ phổ biến để nói kích thước.
- Dùng cho: **đường kính hạt** ("hạt 10 ly" = hạt 10mm), **bản vòng** ("bản 12 ly" = rộng 12mm), **độ dày**, kích thước mặt đá.
- Xác minh: bảng quy đổi carat ↔ ly trong ngành kim cương/đá quý VN quy ước ly = mm (vd 0.5ct ≈ 5.0 ly, 1.0ct ≈ 6.5 ly). Nguồn: luatminhkhue.vn.
- ➜ **Khuyến nghị app:** lưu trong DB bằng **mm** (số thực). "ly" chỉ là nhãn hiển thị/nhập liệu; cho phép nhập "ly" rồi map 1:1 sang mm.

### "ni" = **cỡ/size (chỉ số vòng đeo khít)** — chủ yếu cho nhẫn & vòng bangle
- "Ni" (đo "ni", lấy "ni") = lấy số đo cỡ để sản phẩm vừa khít ngón tay / cổ tay.
- **Ni nhẫn:** mã size nhẫn. Ở VN phổ biến **size 8–23**, tương ứng **đường kính trong ~15mm → ~22mm+**.
  - Nữ thường ni 9–15 (ĐK trong ~15.7–17.7mm); Nam thường ni 15–21 (ĐK trong ~17.7–20.0mm).
  - Quan hệ: Chu vi ngón = π × đường kính (chia chu vi cho 3.14 ra đường kính).
- **Ni vòng/niềng vòng (bangle):** đường kính **trong** của vòng (mm) — đây là số đo quyết định đeo vừa. Shop thường ghi thẳng "niềng 56", "ni 56" = đường kính trong 56mm.
  - Bảng size bangle tham chiếu (theo đường kính trong): XS 50–54mm · S 54–56mm · M 56–58mm · L 58–60mm · XL 60–64mm (nguồn Baikalla).
- ➜ **Khuyến nghị app:** với nhẫn lưu cả **ni (mã số)** + **đường kính trong (mm)**; với bangle lưu **đường kính trong (mm)** là chính (ni = cùng giá trị).

---

## 2. Bảng đặc điểm theo từng loại sản phẩm

Ký hiệu: ✅ = Bắt buộc (cần để quản lý kho + bán) · ⬜ = Tùy chọn (tăng chất lượng mô tả) · *đơn vị lưu DB in đậm*

### 2.1. Vòng tay — Vòng cẩm thạch / bangle (niềng liền khối)
| Trường | Đơn vị | Bắt buộc | Ghi chú |
|---|---|---|---|
| Đường kính trong (niềng/ni vòng) | **mm** | ✅ | Số đo đeo vừa; map sang size XS–XL |
| Bản vòng (bề rộng) | **mm** (ly) | ✅ | VD 9–13mm; "bản 12 ly" |
| Độ dày | **mm** (ly) | ✅ | VD 8–11mm |
| Dáng mặt cắt | enum | ⬜ | Tròn (bản tròn) / Dẹt (bản vuông) / Trúc tiết / Oval |
| Hình niềng trong | enum | ⬜ | Tròn / Oval |
| Trọng lượng | **gram** | ⬜ | Tham khảo, chống nhầm hàng |

### 2.2. Vòng tay — Vòng charm / lắc (hạt + charm, dây thun/dây)
| Trường | Đơn vị | Bắt buộc | Ghi chú |
|---|---|---|---|
| Đường kính hạt | **mm** (ly) | ✅ | VD "hạt 8 ly" = 8mm |
| Số hạt | cái | ⬜ | Có thể suy từ chiều dài |
| Chu vi/chiều dài vòng | **mm/cm** | ✅ | Cỡ cổ tay; dây thun thường ~14–18cm |
| Kiểu dây | enum | ⬜ | Thun / Dây rút / Dây kim loại / Chỉ |
| Charm/phụ kiện đính kèm | text | ⬜ | VD charm vàng, charm bạc, hạt chặn |
| Trọng lượng | **gram** | ⬜ | |

### 2.3. Nhẫn
| Trường | Đơn vị | Bắt buộc | Ghi chú |
|---|---|---|---|
| Ni nhẫn (mã size) | số (8–23) | ✅ | Chuẩn size VN |
| Đường kính trong | **mm** | ✅ | ~15–22mm; map từ ni |
| Bản mặt / bề rộng mặt đá | **mm** (ly) | ✅ | Kích thước phần đá/ổ |
| Dài × rộng mặt đá | **mm × mm** | ⬜ | Nếu mặt đá hình chữ nhật/oval |
| Chất liệu ổ/đai | enum | ⬜ | Bạc / Bạc mạ / Hợp kim / Vàng / Nguyên khối đá |
| Trọng lượng | **gram** | ⬜ | |

### 2.4. Chuỗi hạt (vòng chuỗi)
| Trường | Đơn vị | Bắt buộc | Ghi chú |
|---|---|---|---|
| Đường kính hạt | **mm** (ly) | ✅ | VD "hạt 10 ly"; hạt đều hay nhiều cỡ |
| Số hạt | cái | ✅ | Chuỗi tâm linh hay dùng số có ý nghĩa (18, 108…) |
| Chiều dài chuỗi | **cm** | ✅ | Đo tổng chiều dài khi khép |
| Hình dạng hạt | enum | ⬜ | Tròn / Trụ / Tỳ hưu / Charm xen kẽ |
| Phụ kiện | text | ⬜ | Hạt chặn, charm, khóa |
| Trọng lượng | **gram** | ⬜ | |

### 2.5. Mặt dây (mặt dây chuyền)
| Trường | Đơn vị | Bắt buộc | Ghi chú |
|---|---|---|---|
| Dài | **mm** (ly) | ✅ | Chiều cao mặt |
| Rộng | **mm** (ly) | ✅ | |
| Dày | **mm** (ly) | ✅ | |
| Trọng lượng | **gram** | ✅ | Mặt dây thường niêm yết theo gram |
| Chủ đề/hình khắc | enum/text | ⬜ | Phật Di Lặc, Quan Âm, Tỳ hưu, Hồ ly, Lá, Giọt, Trơn… |
| Có kèm dây/khoen? | bool + text | ⬜ | Bọc vàng/bạc, khoen treo |

---

## 3. Đặc điểm phi-kích-thước (dùng chung mọi loại)

| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| Loại đá | enum | ✅ | Cẩm thạch (jadeite/phỉ thúy), ngọc bích (nephrite), ngọc Jade khác… |
| Màu sắc chủ đạo | enum/multi | ✅ | Lục (xanh lá), trắng, tím (tử/lavender), vàng, đỏ/huyết, đen, đa sắc |
| "Nước" (độ trong) | enum | ⬜ | Thang: Đục → Bán trong → Trong (băng) → Rất trong (thủy/kính). "Nước" cao = giá cao |
| "Chủng" (độ mịn hạt) | enum | ⬜ | Chủng thô / mịn / băng / thủy (glassy). Thuật ngữ phỉ thúy |
| Phân loại xử lý | enum | ✅ | **Loại A** (tự nhiên, chỉ phủ sáp) / **Loại B** (tẩy acid + tiêm polymer) / **Loại C** (nhuộm màu) / **B+C** |
| Giấy kiểm định | bool + mã | ⬜→✅* | Có/không + đơn vị cấp + mã số. *Nên bắt buộc nếu bán là "loại A" |
| Nguồn gốc | enum | ⬜ | Myanmar (Miến Điện), Trung Quốc, VN, khác |
| Trọng lượng | gram (hoặc carat) | ⬜ | 1ct = 0.2g; mặt dây dùng gram |
| Ý nghĩa phong thủy / mệnh hợp | multi-enum/text | ⬜ | Ngũ hành: Kim/Mộc/Thủy/Hỏa/Thổ; màu ↔ mệnh (vd lục hợp Mộc). Hỗ trợ bán |
| Tình trạng | enum | ✅ | Mới / Đã qua chế tác / Có tì vết |
| Ghi chú tì vết/đặc điểm riêng | text | ⬜ | Vết nứt, điểm đá, vân |

### Về Loại A/B/C (quan trọng cho định giá & niềm tin KH)
- **Loại A:** tự nhiên, không xử lý (có thể phủ sáp tôn màu). Bền nhất, giá cao nhất. Có thể còn tì vết/điểm nâu-đen tự nhiên.
- **Loại B:** tẩy acid làm sạch + tiêm nhựa polymer tăng độ trong. Kém bền hơn (polymer phân hủy bởi nhiệt/chất tẩy rửa), có thể xuống màu.
- **Loại C:** nhuộm màu. **Phai màu theo thời gian/ánh nắng** — không ổn định.
- ➜ **Giấy kiểm định** rất quan trọng để phân biệt A vs B/C (cần phòng lab, máy quang phổ). Nếu shop quảng cáo "loại A" nên bắt buộc có mã kiểm định trong kho.

---

## 4. Đề xuất bộ trường cho form app + data model (JSON)

### Nguyên tắc
1. **Lưu mọi kích thước bằng mm** (số thực) trong DB; UI cho nhập "ly" (map 1:1) để hợp thói quen người bán.
2. **Kích thước tách theo loại** → dùng object `dimensions` có cấu trúc khác nhau theo `productType` (polymorphic), thay vì nhồi tất cả vào 1 bảng phẳng.
3. Trường chung (đá, màu, loại A/B/C, phong thủy, giá, tồn) nằm ở cấp product gốc.

### Data model gợi ý (JSON)
```json
{
  "sku": "VCT-0001",
  "productType": "bangle | charm_bracelet | ring | bead_string | pendant",
  "name": "Vòng cẩm thạch niềng 56 bản 12 ly",
  "stone": {
    "material": "jadeite",
    "primaryColor": "green",
    "colors": ["green", "white"],
    "transparency": "ice",          // opaque|semi|translucent|ice|glassy  (="nước")
    "texture": "fine",              // coarse|fine|ice|glassy               (="chủng")
    "treatment": "A",               // A|B|C|B+C
    "origin": "myanmar",
    "certificate": { "has": true, "lab": "PNJLab", "code": "ABC123" }
  },
  "weightGram": 58.2,
  "fengShui": { "elements": ["moc"], "motif": "di_lac" },
  "dimensions": { /* theo productType, xem dưới */ },
  "condition": "new",
  "notes": "Có điểm đá nhỏ ở mặt trong",
  "priceVnd": 1850000,
  "quantity": 1
}
```

### `dimensions` theo từng `productType`
```json
// bangle (vòng cẩm thạch)
{ "innerDiameterMm": 56, "widthMm": 12, "thicknessMm": 9, "crossSection": "round", "innerShape": "round" }

// charm_bracelet (vòng charm/lắc)
{ "beadDiameterMm": 8, "beadCount": 18, "lengthMm": 160, "cordType": "elastic", "charms": "charm bạc" }

// ring (nhẫn)
{ "ringSizeVn": 12, "innerDiameterMm": 16.5, "faceWidthMm": 10, "faceLengthMm": 14, "setting": "silver" }

// bead_string (chuỗi hạt)
{ "beadDiameterMm": 10, "beadCount": 108, "lengthCm": 54, "beadShape": "round", "accessories": "hạt chặn" }

// pendant (mặt dây)
{ "lengthMm": 45, "widthMm": 28, "thicknessMm": 7, "motif": "di_lac", "hasBail": true, "bail": "bọc bạc" }
```

### Enum gợi ý cho UI
- `transparency` (Nước): Đục / Bán trong / Trong / Băng / Thủy (kính)
- `texture` (Chủng): Thô / Mịn / Băng / Thủy
- `treatment`: A (tự nhiên) / B (tẩm polymer) / C (nhuộm) / B+C
- `primaryColor`: Lục / Trắng / Tím / Vàng / Đỏ-huyết / Đen / Đa sắc
- `origin`: Myanmar / Trung Quốc / Việt Nam / Khác
- `fengShui.elements`: Kim / Mộc / Thủy / Hỏa / Thổ
- `crossSection` (bangle): Tròn / Dẹt / Trúc tiết / Oval
- `motif` (mặt dây/charm): Phật Di Lặc / Quan Âm / Tỳ hưu / Hồ ly / Lá / Giọt / Trơn / Khác

### Tối thiểu bắt buộc để chốt 1 sản phẩm vào kho
`sku`, `productType`, `name`, `stone.material`, `stone.primaryColor`, `stone.treatment`, các field kích thước ✅ của loại tương ứng, `condition`, `priceVnd`, `quantity`.
(Nếu quảng cáo là "loại A" ➜ thêm bắt buộc `stone.certificate`.)

---

## 5. Nguồn tham khảo
- Quy đổi "ly = mm" / carat ↔ ly: https://luatminhkhue.vn/carat-la-gi-1-carat-kim-cuong-bang-bao-nhieu-ly.aspx
- Hướng dẫn đo ni nhẫn (size VN 8–23, ĐK mm): https://hoanghamobile.com/tin-tuc/huong-dan-do-ni-nhan-chuan-xac-tai-nha-tu-a-den-z-de-chon-nhan-vua-van/
- Bảng size bangle (XS–XL theo đường kính trong): https://www.baikalla.com/pages/sizing-charts
- Ví dụ niêm yết vòng bangle thực (niềng 56, bản 9.9mm, dày 10.3mm, loại A, Myanmar): https://en.pinkoi.com/product/AUwYAiwB
- Phân loại phỉ thúy A/B/C (xử lý, độ bền, kiểm định): https://www.jewellerybusiness.com/?p=26861 ; https://hnfjewellery.com/education/
- Loại A/B/C (tiếng Việt, tẩm sáp/polymer/nhuộm): https://www.lukfook.com/en/page/jewellery-guide/jadeite
- Cảnh báo "mượn danh phỉ thúy" / nhận diện ngọc thật: https://www.sggp.org.vn/muon-danh-phi-thuy-moc-tui-nguoi-tieu-dung-post862969.html ; https://mytour.vn/vi/blog/bai-viet/bi-quyet-nhan-dien-ngoc-bich-that.html
- Máy tính số hạt chuỗi (bead mm ↔ số hạt ↔ chiều dài): https://miniwebtool.com/vi/may-tinh-mau-hat-cuom/
