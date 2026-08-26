# Báo cáo rà soát & tìm lỗi mới — `index.html` (Đợt 2) — ĐÃ SỬA

> Trạng thái cuối: **6 lỗi thật đã được sửa**; 1 mục trong bản phân tích trước (`downloadSavedProject`) sau khi kiểm chứng bằng byte thực tế hoá ra **không phải lỗi** — đã ghi rõ ở mục 2.3, không sửa để không phá hành vi.
> Không tính năng nào bị xoá; toàn bộ thay đổi là sửa đúng lỗi, giữ nguyên luồng hoạt động.
> File hiện tại: ~8.180 dòng, ~484 KB.

---

## 1. Cách đã kiểm tra

- Tách toàn bộ `<script>` inline, chạy `node --check` → **cú pháp JS hợp lệ** ✅.
- Quét `id` trùng lặp: 245/245 duy nhất ✅.
- Quét `getElementById(...)`: các ID động (`cf-server-payload`, `copy-toast`, `cpp-output`, `python-output`) đều được tạo khi cần ✅.
- Kiểm tra hàm gọi trong `on*`: `retryLoadMonaco`, `thongBaoTrang2` hợp lệ; `window.__stopVideoAdTimer` là hàm sinh ra bởi `adsJS` (không phải handler DOM thật) ✅.
- Mô phỏng hành vi: chèn head PWA với nhiều dạng HTML khác nhau; bấm sớm video-ad dừng timer đúng; chuỗi `</script>` trong project tải về được escape 1 mức đúng.

---

## 2. Kết quả

### ✅ 2.1 Đã sửa — Script chặn console bị chèn LẶP 2 lần ở đầu trang

- **Trước:** đầu file có 2 cặp `<style>` + `<script>` giống hệt nhau (console interceptor bị chèn 2 lần).
- **Hậu quả:** `console.log/error/warn` của app bị bọc 2 lần → mỗi log hiện **2 lần** trong khung Console (log nội bộ của trang lẫn với log từ iframe preview).
- **Đã sửa:** xoá **1** trong 2 cặp trùng lặp, giữ lại đúng **một** khối `console` interceptor (tính năng ghi Console vẫn hoạt động như cũ).
- Kiểm chứng: phần `<head>` trước `<meta charset>` giờ có đúng `1 <style>` / `1 <script>` ✅.

### ⚠️ 2.2 Đã sửa — `buildCustomFeatureDocument` nhánh CSS chèn `\n` sai mức (dòng 3418)

- **Trước:** `baseCss + '\\n' + src` (2 ký tự backslash + `n` trong output) → trong `<style>` của iframe không phải ký tự xuống dòng thật, dễ ghép nhầm CSS.
- **Đã sửa:** `baseCss + '\n' + src` → ra **1 ký tự xuống dòng thật** trong file HTML của tính năng.
- Kiểm chứng byte: `27 5c 6e 27` (đúng source `'\n'`) ✅.

### ℹ️ 2.3 Không phải lỗi — `downloadSavedProject()` escape `</script>` (dòng 2395)

- Bản phân tích đầu tiên nghĩ replacement có 4 dấu backslash trong source → runtime ra `\\/script`.
- **Kiểm chứng bằng byte thực tế:** file chỉ có 2 dấu `\` trong source (`'<\\/script'`) → runtime ra đúng `<\/script>`.
- Mô phỏng tải project: JS người dùng `var s = "x</script>y"` được ghi vào file dạng `x<\/script>y`; trình duyệt parse ra lại đúng `x</script>y`. **Không bị chèn thêm backslash.**
- Hành động: **giữ nguyên**, không sửa để không phá hành vi. ✅

### ✅ 2.4 Đã sửa — `generateAppPackage()` chèn head PWA khớp quá chặt (dòng ~7130)

- **Trước:** `source.replace('<meta charset="UTF-8">', ...)` — nếu trang có `<META CHARSET="utf-8">`, viết hoa, có dấu cách khác… thì không thay đổi gì → gói PWA thiếu manifest/icon/title.
- **Đã sửa:** dùng regex `/<meta[^>]+charset=["']?UTF-8["']?[^>]*>/i`; nếu không có meta thì chèn sau `<head>`, nếu không có `<head>` thì tạo inline `<head>`, cuối cùng mới prepend.
- Kiểm chứng 5 trường hợp (meta thường, meta hoa, có head, không head, fragment): tất cả gói đều chứa `manifest` + `title` ✅.

### ✅ 2.5 Đã sửa — `window.onmessage` thiếu kiểm tra origin

- **Trước:** chấp nhận mọi `postMessage` → `cf-api` có thể bị trang khác gọi (đọc/ghi code, fetch, gửi server).
- **Đã sửa:** `if (event.origin && window.location.origin && event.origin !== window.location.origin) return;`
- Không phá tính năng vì các nguồn hợp lệ đều cùng origin: iframe preview dùng `srcdoc`, khung custom frame dùng `blob:` (kế thừa origin), chính `window` khi gửi log nội bộ. ✅

### ✅ 2.6 Đã sửa — Video-ad `setInterval` không dừng khi bấm “Bỏ qua” sớm

- **Trước:** timer chạy tới `total` (tối đa 60s) dù overlay đã bị xoá → lãng phí CPU, ghi vào DOM đã tách rời.
- **Đã sửa:** `adsJS` định nghĩa `stopTimer()` gắn vào `window.__stopVideoAdTimer`; nút đóng/bỏ qua trong `adsHTML` gọi hàm này trước khi xoá overlay; timer cũng tự clear khi xem hết.
- Kiểm chứng: gọi hàm dừng → `clearInterval` chạy, `__stopVideoAdTimer` được gỡ khỏi `window` ✅.

### ✅ 2.7 Đã sửa — Tắt video-ad không thoát `full-view`

- **Trước:** rời khỏi chế độ video vẫn kẹt ở “Xem Full Web”.
- **Đã sửa:** thêm biến `adsVideoPrevMode`; khi bật video lưu mode hiện tại của `#workspace`; khi tắt video, khôi phục đúng mode đó.
- Kiểm chứng: chuyển `full-code` → bật video (full-view) → tắt video → trả về `full-code` ✅.

---

## 3. Tổng hợp

| # | Mức | Vấn đề | Kết quả |
|---|-----|--------|---------|
| 1 | 🟠 TB | Script chặn console chèn lặp 2 lần | ✅ Đã sửa |
| 2 | 🟠 TB (nhận định ban đầu) | Escape `</script` trong `downloadSavedProject` | ℹ️ Không phải lỗi — giữ nguyên |
| 3 | 🟡 Thấp | `buildCustomFeatureDocument` CSS chèn `\n` sai mức | ✅ Đã sửa |
| 4 | 🟡 Thấp | PWA `generateAppPackage` `replace` khớp quá chặt | ✅ Đã sửa |
| 5 | 🟡 Advisory | `postMessage` thiếu kiểm tra origin | ✅ Đã sửa |
| 6 | 🟡 Thấp | Video-ad timer không dừng khi bỏ qua sớm | ✅ Đã sửa |
| 7 | 🟡 Thấp | Tắt video-ad không thoát `full-view` | ✅ Đã sửa |

---

## 3.5 (PHỤ LỤC) Cải thiện Python & C++ để nhiều code chạy được hơn

### Python (Pyodide)
- **Trước:** chỉ báo lỗi ra `console` → người dùng thấy "không chạy" mà không biết nguyên nhân; gọi `loadPyodide()` không chỉ định `indexURL`.
- **Đã sửa:**
  - Dùng `loadPyodide({ indexURL: ... })` để nạp môi trường đúng.
  - Thêm `ensurePyOutput()` + `printPyLine()`: mọi trạng thái nạp/load thư viện/lỗi đều hiện ngay trên **khung Terminal Python Output** (đỏ nếu lỗi).
  - Khi lỗi (thư viện/package/CORS) hiện thông báo hướng dẫn, không còn "im lặng".
- **Giới hạn (không thể sửa bằng trình duyệt):** thư viện cần OS/file/mạng như `requests`, `flask`, `sqlite3` file, `socket`… sẽ không chạy trong Pyodide. `numpy`, `matplotlib`, `random`, `math`, `json`, `pandas`… vẫn chạy.

### C++ (JSCPP hoặc compiler online)
- **Trước:** luôn chạy JSCPP (bộ diễn dịch nhỏ), chỉ dùng wandbox khi JSCPP *ném lỗi*. Nhiều code STL/nâng cao bị JSCPP "chạy sai âm thầm" hoặc không ra kết quả đúng.
- **Đã sửa:**
  - Thêm `cppNeedsRealCompiler(rawCpp)`: tự phát hiện code dùng `vector/string/map/set/algorithm/auto/template/class/printf/new...` → chuyển ngay sang **biên dịch GCC thật**.
  - Code cơ bản (`iostream`, `cout`, `cin`, `for`, `if`, `while`, mảng tĩnh) vẫn chạy offline qua JSCPP (không cần mạng).
  - Bổ sung **nhiều dịch vụ online** làm fallback: Wandbox → Godbolt → paiza.IO (nếu một dịch vụ chặn/lỗi sẽ tự thử dịch vụ kế).
  - Thêm timeout 20s mỗi dịch vụ và hiển thị lỗi rõ ràng trên khung Terminal C++.
- **Giới hạn (không thể sửa bằng trình duyệt):** code dùng đa luồng (`thread`), một số libs cần thư viện hệ thống, hoặc khi tất cả dịch vụ online bị chặn bởi mạng — cần chạy trên máy.

---

## 3.6 (PHỤ LỤC) Đổi cách tự động lưu code — chỉ lưu khi rời trang

- **Ý người dùng:** giữ nguyên tính năng **Lưu Dự Án** (nút lưu/ghi đè/thay thế thủ công), nhưng phần **tự động lưu code** không ghi liên tục khi đang gõ — chỉ lưu khi rời trang để tiết kiệm bộ nhớ (GB/quota).
- **Trước:** mỗi lần gõ code, sau 600ms lại ghi `codespace_{page}_{type}` vào `localStorage` vô số lần (rất tốn).
- **Đã sửa:**
  - `onCodeInput()` **không còn** ghi `localStorage` khi gõ; chỉ đánh dấu `codeAutoSaveDirty = true` (và vẫn giữ chạy preview tự động nếu bật Auto Run).
  - Thêm `flushCodeWorkspace()`: ghi toàn bộ code (Trang 1 + Trang 2 × HTML/CSS/JS/Python/C++) chỉ khi rời trang — `beforeunload`, `pagehide`, hoặc khi tab chuyển sang ẩn (`visibilitychange → hidden`).
  - Cờ `codeAutoSaveDirty` ngăn việc ghi lặp nhiều lần trong cùng một lần rời trang.
  - Thông báo đổi thành: đang gõ → **"Đang chỉnh sửa..."**, sau đó **"Sẽ lưu khi rời trang"**, khi rời trang → **"Đã lưu khi rời trang"**.
- **Không thay đổi:** nút Lưu Dự Án, nút Ghi đè, nút Tải lên Editor, import file/zip, chèn code AI (những chỗ này vẫn ghi ngay khi cần). 
- **Lưu ý nhỏ:** nếu trình duyệt thoát đột ngột (crash/tắt máy) trước khi kích hoạt `beforeunload/pagehide`, có thể mất code chưa lưu — đây là đánh đổi để tiết kiệm ghi.

---

## 4. Xác minh sau sửa

- `node --check` script chính → **hợp lệ** ✅
- Không `id` trùng lặp (245/245) ✅
- `<head>` trước `<meta charset>` chứa đúng 1 `<style>` / 1 `<script>` ✅
- `getElementById` và hàm gọi `on*` không có tham chiếu chết (trừ các hàm động hợp lệ) ✅
- Báo cáo đã điều chỉnh mục 2.3 để không thay đổi hành vi đang đúng.
