# Báo cáo phân tích & tìm lỗi — `to-n-9/index.html` (đã sửa các lỗi đã xác nhận ✅)

> Cập nhật lần cuối: **6 lỗi ban đầu và các lỗi bổ sung đã được sửa/kiểm chứng**. Mục 2 giữ lại mô tả lỗi gốc để đối chiếu.

---

## 1. Tổng quan

- **Kích thước hiện tại:** ~8.370 dòng, ~494 KB.
- **Kiểm tra tự động sau khi sửa:**
  - `node --check` script chính → cú pháp JS **hợp lệ** ✅
  - Không có `id` trùng lặp (248/248 id duy nhất) ✅
  - HTML cân bằng thẻ: `<div>` 316/316, `<select>` 13/13, `<span>` 57/57 ✅
  - Mọi hàm gọi trong `onclick/onchange/onkeydown...` đều tồn tại ✅
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

---

## 5. Tối ưu hiệu năng (thêm mới — không thay đổi hành vi/tính năng)

| # | Khu vực | Vấn đề | Tối ưu đã làm |
|---|---------|--------|---------------|
| 1 | Khởi động | Tải **pyodide (~2.5MB) không dùng ở cấp ứng dụng** + `tesseract.js`/`html2canvas` tải sẵn từ đầu → trang mở rất chậm, bị chặn parse HTML | Bỏ hẳn pyodide ở app (iframe tự nạp khi chạy Python); `tesseract`/`html2canvas` → **lazy-load** đúng lúc dùng (OCR / chụp ảnh) qua `loadScriptOnce()`; thêm `defer` cho JSZip + Monaco loader để không chặn dựng trang |
| 2 | Gõ code | `updateHttpsHighlighting()` quét toàn bộ code **mỗi keystroke** | Debounce 250ms **theo từng editor** (Map theo `getId()`) — highlight vẫn đủ, chỉ bớt việc quét liên tục |
| 3 | Gõ code | `showSaveStatus()` tạo timeout mới mỗi phím (churn timer) | Dùng 1 timer dùng chung, `clearTimeout` trước khi set mới |
| 4 | Chạy code | `runCode()` gán lại `srcdoc` kể cả khi source không đổi → iframe re-render thừa | Chỉ gán khi `viewer.srcdoc !== source` (không đổi hành vi, đỡ re-render lặp) |
| 5 | Console | Bảng log phình vô hạn + `innerHTML.includes` quét toàn bộ mỗi dòng log → chậm dần O(n²) | Giữ tối đa **1500 dòng** (xóa dòng đầu khi vượt); kiểm tra dòng "Hệ thống..." qua `firstChild` thay vì quét toàn DOM |
| 6 | Kéo khung AI web | `mousemove` set `style.width` mỗi event → nhiều layout/frame thừa | Throttle bằng `requestAnimationFrame` (chỉ 1 lần/frame), `endDrag` huỷ rAF đang chờ |
| 7 | Log Boss Bot | `textContent +=` không giới hạn → chuỗi phình vô hạn | Buffer có giới hạn (~20KB, giữ 12KB cuối); tự đồng bộ nếu nơi khác ghi đè trực tiếp |
| 8 | Gộp tính năng | Ô tìm kiếm render lại toàn bộ danh sách mỗi keystroke | Debounce 150ms qua `scheduleComboFeaturePickerRender()` |

**Kiểm chứng (đều PASS):**
- `node --check` hợp lệ; 248 id duy nhất; thẻ cân bằng (`div` 316/316, `select` 13/13, `span` 57/57, `iframe` 4/4); mọi handler & `getElementById` tồn tại.
- Mô phỏng Node: `runCode` chỉ gán srcdoc khi khác (3 lần gọi → 1 lần gán); `webUrlLog` rút gọn đúng (~12.350 ký tự sau 3000 dòng) + đồng bộ khi bị ghi đè ngoài; console giữ đúng 1500 dòng; lazy-loader chỉ chèn 1 script, lỗi thì reject và tải lại được; highlight debounce gõ liên tục chỉ chạy 1 lần/250ms/editor.
- Tính năng giữ nguyên: pyodide & JSCPP vẫn được nhúng trong trang kết quả, JSZip vẫn hoạt động, OCR/chụp ảnh có fallback thông báo khi không tải được thư viện.

---

## 6. Sửa "AI Trợ Lý bị lỗi" (không trả lời dù key đúng — đặc biệt trên điện thoại)

**Nguyên nhân chẩn đoán được:**
1. **Gemini API chặn theo khu vực** — Việt Nam không nằm trong danh sách quốc gia được hỗ trợ → API trả 403 `User location is not supported` dù key đúng.
2. **CORS** — OpenAI / DeepSeek / xAI không cho trình duyệt gọi thẳng (không gửi `Access-Control-Allow-Origin`) → `Failed to fetch`.
3. **Bàn phím điện thoại** — `onkeypress` (cũ, đã deprecated) nhiều trình duyệt di động không kích hoạt → bấm Enter không gửi.

**Đã sửa:**
| Thay đổi | Chi tiết |
|----------|----------|
| 🔌 **Nút "Kiểm tra kết nối"** trong modal 🔑 Key | Gửi 1 câu thử tới provider đang nhập → hiện "✅ Kết nối thành công" hoặc lỗi chi tiết — biết ngay key/CORS có vấn đề |
| 🌐 **CORS Proxy cho từng key** (ô mới "CORS Proxy (tùy chọn)") | Hỗ trợ `https://corsproxy.io/?url=` hoặc dạng `{url}`. Mọi request AI đi qua proxy khi cấu hình |
| 🔄 **Tự fallback qua proxy** | Nếu gọi trực tiếp fail (CORS/mạng) và key có proxy → tự thử lại qua proxy |
| ⏰ **Timeout 60s** mỗi request | Không treo vô hạn; báo rõ "Quá thời gian chờ" |
| 💡 **Thông báo lỗi rõ ràng** | Phân biệt: CORS/mạng → hướng dẫn điền proxy hoặc dùng OpenRouter/Groq/Claude; Gemini chặn vùng → cảnh báo "Nếu bạn ở Việt Nam..."; thiếu key → nút "🔑 Thêm / Chọn Key ngay" mở thẳng modal |
| 📱 **Enter trên điện thoại** | Đổi toàn bộ `onkeypress` → `onkeydown` (6 ô input: chat AI, URL web, menu web, lưu bản lưu, prompt, lấy code web) |
| 📝 **Ghi chú trong modal key** | Hướng dẫn ngay trong form: nên dùng OpenRouter/Groq/Claude hoặc điền CORS Proxy |

**Kiểm chứng (jsdom + fetch mock, đều PASS):**
- Chat trả lời đúng; lỗi CORS → thông báo có hướng dẫn (không "❌ ❌" kép); Gemini 403 vùng → có hint Việt Nam; có proxy + lỗi trực tiếp → tự gọi proxy thành công; Enter (keydown) gửi tin nhắn; nút kiểm tra kết nối hiện kết quả đúng; `node --check` PASS; id duy nhất, thẻ cân bằng.

---

## 7. Sửa bổ sung trong phiên hiện tại

Các thay đổi dưới đây giữ nguyên các tính năng hiện có và chỉ bổ sung kiểm tra/an toàn hoặc sửa hành vi lỗi:

1. **Bảo vệ `postMessage`:** chỉ nhận log/video message từ `#viewer` và chỉ nhận `cf-api` từ hai iframe custom feature; kiểm tra cả `event.source` và origin. Website mở trong khung AI không còn tự ý gọi API nội bộ, sửa code hoặc tắt video ad.
2. **Không phát tán credential custom server:** `CodeSpace.fetch()` không còn tự động gửi `Authorization`/extra headers tới URL tùy ý; gửi feature nội bộ vẫn giữ cơ chế auth cũ.
3. **Autosave theo từng editor:** thay timer dùng chung bằng `Map` theo page/tab; thêm flush khi rời trang để tránh mất chỉnh sửa chưa kịp debounce.
4. **AI Result chống request trùng:** thao tác toggle/chuyển trang không còn tạo request AI trùng; request preview cũ được hủy khi có request mới hoặc khi tắt AI Result. Nút **Chạy** vẫn cho phép chạy lại cưỡng bức.
5. **ZIP chạy được ngay:** entry point `index.html` và các entry page dùng source đã gộp, nhưng các file HTML/CSS/JS/Python/C++ riêng vẫn được giữ lại trong ZIP.
6. **Tải ZIP bền hơn:** ZIP/PWA tự nạp JSZip dự phòng nếu CDN defer chưa kịp tải hoặc CDN chính lỗi.
7. **PWA escape tên app:** tên app trong `<title>` và meta tag được escape đầy đủ.
8. **Ghi nhớ Auto-run:** trạng thái Tự Chạy được khôi phục sau khi reload.
9. **Mở tab ngoài an toàn hơn:** các nút mở AI Web dùng `noopener,noreferrer`.

**Kiểm chứng sau thay đổi:** `node --check` script chính PASS, `git diff --check` PASS, 248 ID vẫn duy nhất, không thêm/xóa tính năng UI.
