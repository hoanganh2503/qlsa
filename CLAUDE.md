# CLAUDE.md

Tài liệu hướng dẫn cho Claude Code khi làm việc với mã nguồn này.

## Tổng quan

**QLSA** (Quản lý Suất ăn Quân đội) là ứng dụng web một trang (single-page app) quản lý
bếp ăn đơn vị quân đội: cấu hình tiêu chuẩn ăn, chấm cơm theo quân số, lên thực đơn,
đi chợ, nhập–xuất–tồn kho, tính tiền ăn và xuất các báo cáo (TBHC, công khai tiêu chuẩn…).

Đây là bản port từ một file Excel sang HTML/JS thuần. Mọi công thức làm tròn được giữ
nguyên đúng như Excel — **không được tự ý đổi cách làm tròn**.

## Kiến trúc

- **Toàn bộ ứng dụng nằm trong 1 file: `index.html`** (~5.200 dòng, ~250 KB).
  CSS, HTML và JavaScript đều inline trong file này. Không có bước build.
- Không có backend. Dữ liệu lưu hoàn toàn trong `localStorage` trình duyệt
  (key `qlsa_csv_v1`).
- Thư viện ngoài tải qua CDN: Bootstrap 5.3, `xlsx@0.18.5`, `exceljs@4.4.0`
  (dùng cho xuất Excel). Font Inter + JetBrains Mono qua Google Fonts.
- `csv_mau/` chứa CSV mẫu (`mat_hang.csv`, `nguoi_an.csv`, `tieu_chuan.csv`) để
  tham chiếu cấu trúc dữ liệu khi import/export.

### Chạy & kiểm thử
- Mở trực tiếp `index.html` trong trình duyệt — không cần server, không cần cài đặt.
- Không có test tự động, không có toolchain. Kiểm thử bằng cách thao tác trên UI.
- Để xóa dữ liệu/khởi tạo lại: xóa key `qlsa_csv_v1` trong localStorage (hoặc dùng
  trang **Sao lưu CSV** trong app).

## Mô hình dữ liệu

Toàn bộ state nằm trong biến toàn cục `DATA` (xem `DEFAULT_DATA()` ~dòng 609).
Các nhánh chính:

- `cauHinh` — thông tin đơn vị, cán bộ, quân số, kỳ hiện tại, giá công khai.
- `tieuChuan` — 4 mức tiêu chuẩn ăn (TC1–TC4), tiền sáng/trưa/chiều.
- `matHang` — danh mục mặt hàng. `isTC:true` = tự xuất theo định lượng tiêu chuẩn;
  `isTC:false` = phụ thuộc thực đơn. `decimal` = số chữ số làm tròn khi xuất kho.
- `monAn` — list món, mỗi món gắn `mhId` (mặt hàng) + định lượng theo bữa S/T/C.
- `nguoiAn` — danh sách người ăn, mỗi người gắn `tcId` (mức tiêu chuẩn).
- `chamCom` — key `${nguoiId}_${ngay}_${bua}`.
- `thucDon` — key `"YYYY-MM-DD"` → `{ S/T/C: [{monId, dl}] }`.
- `diCho` — key `"YYYY-MM-DD"` → `{ S/T/C: { qs, anThem, items:[{mhId,sl}] } }`.
- `nhapTrongNgay`, `phanHo`, `anThem` — nhập kho, phân hộ, ăn thêm.

Hằng số định lượng tiêu chuẩn (port từ sheet "Tiêu chuẩn" của Excel) nằm trên cùng
trong khối `<script>`: `TC_DINH_LUONG`, `TC_DINH_LUONG_NGAY`, `TC_BANG_DEFAULT`,
`TC_NHIET_LUONG`.

### Lưu & migrate
- `loadData()` đọc từ localStorage, fallback về `DEFAULT_DATA()`.
- `saveData()` ghi lại sau mỗi thay đổi — **nhớ gọi `saveData()` sau khi sửa `DATA`**.
- Có đoạn migrate cho snapshot cũ ngay sau `loadData()` (~dòng 716) — khi thêm
  field/mặt hàng mới vào model, cân nhắc bổ sung migrate để không vỡ dữ liệu cũ.

## Tổ chức UI

- **Điều hướng**: sidebar trái với các `.nav-item[data-target]`. Object `SECTIONS`
  (~dòng 1410) map `data-target` → `{ title, renderer, group }`.
- **Render**: `showSection(name)` dựng khung trang rồi gọi `sec.renderer()`. Mỗi trang
  có một hàm `renderXxx()` ghi HTML vào `#page-body`. Có ~28 hàm `render*`.
- Badge `auto` trên menu = trang tính toán tự động (chỉ đọc/derived); badge khác
  (`tham khảo`, `I/O`) là ghi chú loại trang.
- Helper UI: `setSubtitle()`, `setActions()`.

## Quy ước & lưu ý quan trọng

- **Làm tròn**: dùng `roundExcel()`, `roundToUnit()`, `roundMoney()`, `roundThousand()`
  (~dòng 526). Giữ đúng số `decimal` của từng mặt hàng. Đây là phần dễ sai nhất —
  thay đổi sẽ làm lệch số liệu so với Excel gốc.
- **Ngôn ngữ**: UI và comment đều bằng tiếng Việt. Viết code/nội dung mới bằng
  tiếng Việt cho nhất quán.
- **CSV I/O**: tự cài đặt `csvParse`/`csvStringify`/`csvEscape`/`downloadCsv`
  (~dòng 922) — không dùng thư viện. Lưu ý escape dấu phẩy và ký tự tiếng Việt (UTF-8).
- Khi sửa `index.html`, lưu ý file rất lớn — định vị bằng số dòng / tên hàm
  `renderXxx` thay vì đọc toàn bộ.
