# Báo cáo phân tích & tìm lỗi — `to-n-9/index.html` (cập nhật)

> Cập nhật sau khi thêm: (1) kéo chỉnh độ rộng khung Trang Web AI, (2) menu mở Trang Web AI (chọn AI / nhập link / mở khung–cửa sổ–tab).

## 1. Tổng quan

- **Kích thước hiện tại:** 6.839 dòng, ~395 KB.
- **Kiểm tra tự động đã chạy:**
  - `node --check` → cú pháp JS **hợp lệ** ✅
  - Không có `id` trùng lặp (206/206 id duy nhất) ✅
  - HTML cân bằng thẻ: `<div>` 209/209, `<select>` 12/12 ✅
  - Mọi hàm gọi trong `onclick/onchange/onkeypress...` đều tồn tại ✅
  - `getElementById(...)` trỏ tới phần tử tồn tại (3 id "thiếu" là phần tử tạo động lúc chạy: `copy-toast`, `python-output`, `cpp-output` — không phải lỗi) ✅

**Kết luận:** các thay đổi mới (menu + kéo khung) **không gây lỗi cú pháp/hỏng DOM**. Các lỗi cũ tìm được trước đó **vẫn còn nguyên** (chưa sửa) — xem mục 2.

---

## 2. Các lỗi vẫn còn tồn tại (số dòng đã cập nhật)

### 🔴 Lỗi 1 (CAO) — `swRegisterScript` có `<\/script>` bị nhân đôi dấu `\` (dòng 5795)

```js
const swRegisterScript = `<script>... }); }<\\/script>`;
```

- Có **2 dấu `\`**, nên giá trị thật là `<\/script>` (còn nguyên `\`) → trình duyệt **không hiểu đó là thẻ đóng**.
- Chèn vào `index.html` của gói PWA (dòng 5796) → thẻ `<script>` đăng ký service worker **không bao giờ đóng**, phần còn lại trang bị nuốt vào script → **gói PWA tạo ra bị vỡ**.
- **Sửa:** chỉ giữ **1 dấu `\`**: `<\/script>` (giống dòng 2132, 5028...).

### 🔴 Lỗi 2 (CAO) — Regex tách khối code AI lấy nhầm khối đầu tiên (dòng 3450–3452)

```js
const htmlMatch = text.match(/```(?:html)?\s*([\s\S]*?)```/i);
const cssMatch  = text.match(/```(?:css)?\s*([\s\S]*?)```/i);
const jsMatch   = text.match(/```(?:javascript|js)?\s*([\s\S]*?)```/i);
```

- Nhóm ngôn ngữ là **tùy chọn**, nên cả 3 `match()` đều bắt **khối code fence đầu tiên** (đã kiểm chứng: tab CSS & JS nhận đúng nội dung khối HTML).
- **Hậu quả:** "lấy code web bằng AI" điền sai code vào 3 tab.
- **Sửa:** quét toàn bộ khối bằng regex có cờ `g` rồi phân loại theo ngôn ngữ thật.

### 🟠 Lỗi 3 (TB) — `downloadSavedProject` tạo HTML thừa khối script rỗng + title không escape đủ (dòng 2132)

- File tải từ "bản lưu" chứa **khối `<script></script>` rỗng thừa**.
- `<title>` chỉ loại bỏ `<` (không escape `&`, `>`, `"`).
- `${d.js}` nhúng thô vào `<script>` — nếu JS người dùng chứa `</script>` thì file bị vỡ.

### 🟡 Lỗi 4 (THẤP) — Tên biến quảng cáo video ngược nghĩa (dòng 6359–6360)

```js
const totalTime = (adsLocation === 'video-skippable') ? 5 : 60;
const skipAfter = (adsLocation === 'video-skippable') ? 5 : 15;
```
- `video-skippable` thực chất là **bắt buộc xem hết, không bỏ qua được**; `video-non-skippable` lại **bỏ qua được sau 15s** → tên ngược nghĩa, dễ nhầm khi bảo trì.

### 🟡 Lỗi 5 (THẤP) — `runCode()` ép `setMode('full-view')` mỗi lần chạy khi bật quảng cáo video (dòng 6655)

- Mỗi lần auto-run (gõ code) lại kéo sang chế độ "Xem Full Web", phá bố cục người dùng đang dùng.
- **Sửa:** chỉ chuyển `full-view` một lần khi *bật* chế độ quảng cáo.

### 🟡 Lỗi 6 (THẤP) — `querySelector('option[value="..."]')` chưa escape dấu `'` (dòng 3774, 3832)

- URL chứa dấu nháy đơn sẽ làm hỏng selector → ném `DOMException` (hiếm gặp).
- **Sửa:** dùng `CSS.escape(url)` hoặc duyệt `sel.options`.

---

## 3. Ghi chú nhỏ về code MỚI thêm (không phải lỗi nghiêm trọng)

1. **Mở web bằng "Cửa sổ" / "Tab mới" không ghi nhớ URL lần cuối** (`aiWebRememberLastUrl` chỉ được gọi trong `aiWebLoadUrl`). Nghĩa là ô link trong khung nhúng chỉ cập nhật khi "Mở trong khung". → chỉ là sự không nhất quán nhỏ, không ảnh hưởng hoạt động.
2. **Menu Trang Web AI không đóng bằng phím Escape** — handler Escape toàn cục hiện chỉ xử lý hộp thoại xác nhận/nhập. Bấm ra ngoài hoặc nút ✖ vẫn đóng bình thường.
3. **`openAIWebInFrame()`** gọi `toggleAIWebMode(true)` rồi `aiWebLoadUrl()` → iframe bị set `src` 2 lần (thừa nhưng vô hại).
4. `normalizeAIWebUrl()` đã loại bỏ được các chuỗi `javascript:...`, `file:...` (vì `new URL('https://javascript:...')` bị lỗi cổng không hợp lệ) — đủ an toàn cho `window.open`.

---

## 4. Tóm tắt ưu tiên sửa

| # | Dòng | Mức độ | Mô tả ngắn |
|---|------|--------|-----------|
| 1 | 5795 | 🔴 Cao | `<\/script>` gấp đôi `\` → gói PWA bị vỡ |
| 2 | 3450–3452 | 🔴 Cao | Regex tách khối AI lấy nhầm khối đầu tiên |
| 3 | 2132 | 🟠 TB | File tải bản lưu có script rỗng thừa, title không escape đủ |
| 4 | 6359–6360 | 🟡 Thấp | Tên `video-skippable`/`video-non-skippable` ngược nghĩa |
| 5 | 6655 | 🟡 Thấp | `runCode()` ép `full-view` mỗi lần chạy khi bật quảng cáo video |
| 6 | 3774, 3832 | 🟡 Thấp | Selector `option[value="..."]` chưa escape `'` |

*(Ngoài ra: API key AI & GitHub token vẫn lưu plain-text trong `localStorage`; `shareCode()` nhét toàn bộ code vào URL dễ vượt giới hạn độ dài — lưu ý bảo mật.)*
