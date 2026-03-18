# Kế hoạch Triển khai: VitaDairy O2C AI Extractor

## Mục tiêu
Xây dựng công cụ trích xuất đơn hàng (PO) tự động bằng AI, chạy hoàn toàn trên trình duyệt (Client-side). Hệ thống cho phép upload tối đa 10 file PDF/ảnh, xử lý song song bằng Gemini AI, và xuất kết quả trực tiếp ra Excel theo chuẩn SAP S/4HANA FMCG.

---

## Kiến trúc Hệ thống

```
Browser (index.html)
  ├── Upload Module     → Drag & Drop, preview PDF/ảnh
  ├── File Queue        → Hàng đợi tối đa 10 file
  ├── Gemini API Client → Concurrent Queue (3 luồng song song)
  ├── Data Grid         → Hiển thị + chỉnh sửa inline
  └── Excel Exporter    → SheetJS, lọc 38 cột SAP chuẩn
```

---

## Các Thay đổi đã Triển khai

### `index.html` — File duy nhất (Single-page App)

#### Cấu hình Model AI
- Model: **`gemini-2.5-flash`** (hardcode, không auto-detect tránh chọn nhầm Pro)
- Fallback: `gemini-1.5-flash` (kích hoạt khi nhận lỗi 404 hoặc 429)
- Pricing: Input `$0.075/1M token`, Output `$0.30/1M token`, tỉ giá `25,500 VNĐ/USD`

#### Bảo mật API Key
- Key được lưu vào `sessionStorage` (tự xóa khi đóng tab, không persist lâu dài)
- Tự điền lại input khi reload trong cùng phiên

#### Hiệu năng: Concurrent Queue
- Trước: `for...await` tuần tự — 10 file mất 8-10 phút
- Sau: `Promise.allSettled` với `CONCURRENCY_LIMIT = 3` — 10 file còn 3-4 phút
- Queue tự quản lý: mỗi worker tự kéo file tiếp theo khi xong

#### Tối ưu Prompt (Token Cost)
- Xóa JSON template 38 trường rỗng (~800 token thừa/lần gọi)  
- Thay bằng danh sách key names ngắn gọn (~200 token)
- Tiết kiệm ước tính: **~₫6,000-8,000 VNĐ/tháng** với 8,000 PO

#### Logic Nghiệp vụ FMCG (System Prompt)
| Luật | Mô tả |
|---|---|
| Chuẩn hóa ngày | `DD/MM/YYYY`, bỏ giờ phút |
| Chống số mũ | Thêm `'` trước SaleOrder ID, BarCode |
| Làm sạch OCR | Sửa lỗi quang học Tên SKU |
| Tách dòng KM | Sloc=`O1KM`, nối ` - Hàng KM` vào Tên SKU |
| Zero Hallucination | Chỉ trả về JSON Array, không giải thích |

#### Đo lường Chi phí Thực tế
- `performance.now()` bao quanh toàn bộ lệnh `fetch` → đo đúng thời gian API
- Đọc `usageMetadata.promptTokenCount` + `candidatesTokenCount` từ response
- Hiển thị realtime trên UI: **Tổng thời gian** và **API Phí (Ước tính)**
- Log chi tiết trong Console: `[COST] file.pdf: Input=9000tkn, Output=350tkn`

---

## Cấu trúc Thư mục

```
Automate PO/
├── index.html          ← File chính, toàn bộ UI + Logic
├── .env                ← Ghi chú API Key (không đọc tự động, chỉ lưu tham khảo)
└── implements/
    ├── implementation_plan.md  ← Tài liệu này
    └── task.md                 ← Danh sách công việc
```

---

## Deploy lên Vercel

### Cách 1: Vercel CLI
```bash
npm i -g vercel
cd "Automate PO"
vercel --prod
```

### Cách 2: Vercel Dashboard (Kéo thả)
1. Vào [vercel.com](https://vercel.com) → New Project
2. Kéo thả thư mục `Automate PO` vào vùng upload
3. Framework: **Other** (Static HTML)
4. Nhấn Deploy

> [!IMPORTANT]
> File `.env` KHÔNG được đọc tự động bởi trang web tĩnh. API Key phải nhập thủ công qua nút "Cấu hình AI" trên giao diện. KHÔNG commit API Key vào file `.env` khi push lên GitHub public.

---

## Kế hoạch Kiểm tra (Verification)

| Bước | Mô tả | Kết quả mong đợi |
|---|---|---|
| 1 | Mở `index.html` trên trình duyệt | Giao diện hiển thị, không bị vỡ layout |
| 2 | Nhập API Key, lưu cấu hình | Toast "Đã lưu", reload vẫn còn key |
| 3 | Upload 3-5 file PDF cùng lúc | Queue hiển thị đủ, preview đúng file |
| 4 | Bấm Trích xuất AI | 3 file xử lý song song, UI cập nhật realtime |
| 5 | Kiểm tra Console log | `[COST]` log xuất hiện với số token thực tế |
| 6 | Xuất Excel | File `.xlsx` tải về, đủ 38 cột SAP chuẩn |
| 7 | Kiểm tra hóa đơn Google Cloud | Chỉ thấy Gemini Flash, không có Pro |
