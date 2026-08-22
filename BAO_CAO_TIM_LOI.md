# Báo cáo phân tích & tìm lỗi — `to-n-9/index.html` (đã sửa toàn bộ ✅)

> Cập nhật lần cuối: **tất cả 6 lỗi đã được sửa và kiểm chứng** (xem mục 3). Mục 2 giữ lại mô tả lỗi gốc để đối chiếu.

---

## 1. Tổng quan

- **Kích thước hiện tại:** ~6.850 dòng, ~395 KB.
- **Kiểm tra tự động sau khi sửa:**
  - `node --check` script chính → cú pháp JS **hợp lệ** ✅
  - Không có `id` trùng lặp (206/206 id duy nhất) ✅
  - HTML cân bằng thẻ: `<div>` 263/263, `<select>` 12/12, `<span>` 48/48 ✅
  - Mọi hàm gọi trong `onclick/onchange/onkeypress...` đều tồn tại ✅
  - `getElementById(...)` trỏ tới phần tử tồn tại ✅

---

## 2. Các lỗi đã tìm được (trạng thái: ĐÃ SỬA)

### 🔴 Lỗi 1 (CAO) — `swRegisterScript` thừa dấu `\` trước `/script` (dòng 5795 cũ)

```js
// TRƯỚC: }); }<\\/script>`;   (2 dấu \ → giá trị string là "<\/script>" còn nguyên backslash)
// SAU : }); }<\/script>`;    (1 dấu \ → giá trị string là "</script>" — thẻ đóng thật)
```

- Giá trị string cũ chèn vào `index.html` của gói PWA tạo ra chuỗi `<\\/script>` → trình duyệt **không hiểu là thẻ đóng** → thẻ `<script>` đăng ký service worker không bao giờ đóng, phần còn lại trang bị nuốt vào script → **gói PWA bị vỡ**.
- **Đã sửa:** giữ đúng 1 dấu `\` (giống các nơi khác trong file: dòng CDN_PACKAGES, `buildFullSource`...).
- **Kiểm chứng:** chạy Node lấy giá trị string thật → kết thúc bằng `</script>` không còn backslash. ✅

### 🔴 Lỗi 2 (CAO) — Regex tách khối code AI lấy nhầm khối đầu tiên (dòng 3450 cũ)

```js
// TRƯỚC: 3 regex riêng, nhóm ngôn ngữ tùy chọn → cả 3 đều bắt khối fence ĐẦU TIÊN
// (đã kiểm chứng: tab CSS & JS nhận đúng nội dung khối HTML)
```

- **Hậu quả:** "lấy code web bằng AI" điền sai code vào 3 tab.
- **Đã sửa:** quét toàn bộ khối fence bằng một regex có cờ `g`, phân loại theo ngôn ngữ thật (`html/htm/xml`, `css`, `javascript/js`); khối **không gắn nhãn** được ưu tiên cho tab HTML (giữ hành vi cũ).
- **Kiểm chứng:** test với output AI 3 khối `html + css + javascript` → mỗi tab nhận đúng nội dung; khối không nhãn → vào tab HTML. ✅

### 🟠 Lỗi 3 (TB) — `downloadSavedProject` tạo file bị lỗi (dòng 2132 cũ)

- **Trước:** file tải về chứa khối `<script></script>` rỗng thừa; `<title>` chỉ bỏ `<` (không escape `&`, `>`, `"`); `${d.js}` nhúng thô → JS chứa `</script>` làm vỡ file.
- **Đã sửa:** template gọn 1 dòng; title dùng `escapeHtml()` (escape đủ `& < > " '`); JS người dùng được thay `</script` → `<\/script` (chuẩn chống phá thẻ).
- **Kiểm chứng:** chạy Node với tên bản lưu `Demo & <x> "q"` và JS chứa `</script>` → output title đã escape, script đóng đúng, không còn khối rỗng. ✅

### 🟡 Lỗi 4 (THẤP) — Tên giá trị quảng cáo video ngược nghĩa (dòng 6359 cũ)

- **Trước:** `video-skippable` lại là **bắt buộc xem hết 5s**; `video-non-skippable` lại **bỏ qua được sau 15s** → tên ngược nghĩa, dễ nhầm khi bảo trì.
- **Đã sửa:** đổi tên giá trị cho đúng nghĩa — `video-skippable` → **`video-forced`** (5s, không bỏ qua), `video-non-skippable` → **`video-skippable`** (60s, bỏ qua sau 15s). Cập nhật toàn bộ điều kiện, comment, nhãn UI; **không đổi hành vi**.
- **Di trú:** giá trị cũ lưu trong `localStorage` được ánh xạ tự động khi tải trang (`video-skippable` cũ → `video-forced`, `video-non-skippable` cũ → `video-skippable`).
- **Kiểm chứng:** mô phỏng 2 chế độ → `video-forced`: 5s/KHÔNG THỂ BỎ QUA; `video-skippable`: 60s/CÓ THỂ BỎ QUA SAU 15s. ✅

### 🟡 Lỗi 5 (THẤP) — `runCode()` ép `setMode('full-view')` mỗi lần chạy (dòng 6655 cũ)

- **Trước:** mỗi lần auto-run (gõ code) lại kéo sang chế độ "Xem Full Web", phá bố cục đang dùng.
- **Đã sửa:** bỏ ép buộc trong `runCode()`; chỉ chuyển `full-view` **một lần** khi (a) người dùng bật chế độ quảng cáo video trong cài đặt, và (b) khi tải trang nếu đang bật chế độ video (giữ đúng hành vi cũ lúc khởi động).
- **Kiểm chứng:** `runCode()` không còn gọi `setMode`; `onAdsLocationChange` chuyển full-view đúng 1 lần. ✅

### 🟡 Lỗi 6 (THẤP) — `querySelector('option[value="..."]')` chưa escape dấu `'` (dòng 3774, 3832 cũ)

- **Trước:** URL chứa dấu nháy đơn làm hỏng selector → ném `DOMException` (hiếm gặp).
- **Đã sửa:** thêm helper `aiWebSetPresetSelect(url)` duyệt `sel.options` so sánh giá trị trực tiếp — an toàn với mọi ký tự URL (kể cả `'` và `"`).
- **Kiểm chứng:** test URL `https://x.com/a'b` → khớp đúng option; URL lạ → xóa chọn. ✅

---

## 3. Tóm tắt thay đổi

| # | Mức độ | Mô tả ngắn | Trạng thái |
|---|--------|-----------|-----------|
| 1 | 🔴 Cao | `swRegisterScript` thừa `\` → gói PWA vỡ | ✅ Đã sửa |
| 2 | 🔴 Cao | Regex tách khối AI lấy nhầm khối đầu tiên | ✅ Đã sửa |
| 3 | 🟠 TB | File tải bản lưu: script rỗng thừa, title không escape, JS thô | ✅ Đã sửa |
| 4 | 🟡 Thấp | Tên `video-skippable`/`video-non-skippable` ngược nghĩa | ✅ Đã sửa (đổi tên + di trú localStorage) |
| 5 | 🟡 Thấp | `runCode()` ép `full-view` mỗi lần chạy | ✅ Đã sửa |
| 6 | 🟡 Thấp | Selector `option[value="..."]` chưa escape `'` | ✅ Đã sửa |

---

## 4. Ghi chú còn lại (không phải lỗi nghiêm trọng, chưa xử lý)

1. **API key AI & GitHub token** vẫn lưu plain-text trong `localStorage` — lưu ý bảo mật.
2. **`shareCode()`** nhét toàn bộ code vào URL — dễ vượt giới hạn độ dài URL.
3. **Menu Trang Web AI** không đóng bằng phím Escape (bấm ra ngoài / nút ✖ vẫn đóng được).
4. **Mở web bằng "Cửa sổ"/"Tab mới"** không ghi nhớ URL lần cuối vào ô link (chỉ "Mở trong khung" ghi nhớ) — không nhất quán nhỏ, không ảnh hưởng hoạt động.
5. `openAIWebFrameInFrame()` gán `src` iframe 2 lần (thừa nhưng vô hại).
