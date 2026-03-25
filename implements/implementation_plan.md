# Kế hoạch Triển khai: VitaDairy O2C AI Extractor

## Mục tiêu
Xây dựng công cụ trích xuất đơn hàng (PO) tự động bằng AI, chạy hoàn toàn trên trình duyệt (Client-side). Hệ thống cho phép upload tối đa 10 file PDF/ảnh, xử lý song song bằng Gemini AI, và xuất kết quả trực tiếp ra Excel theo chuẩn SAP S/4HANA FMCG.

---

## Kiến trúc Hệ thống

```
Browser (index.html)
  ├── UI/UX Layer       → Tailwind CSS, Skeleton Loading, Empty States
  ├── Upload Module     → Drag & Drop, preview PDF/ảnh
  ├── File Queue        → Hàng đợi tối đa 10 file (Concurrency = 3)
  ├── AI Engine         → Gemini 2.5 Flash (OCR + Logic)
  │     ├── Dictionary  → Dynamic Prompt Injection
  │     ├── Fuzzy Match → AI-driven mapping fallback
  │     └── Data Guard  → Anomaly detection
  ├── Chat Assistant    → Floating AI context-aware widget
  ├── Data Grid         → Hiển thị + chỉnh sửa inline (Data Attributes ready)
  └── Excel Exporter    → SheetJS, lọc 38 cột SAP chuẩn
```

---

## Các Thay đổi đã Triển khai

### 1. AI "Vibe Coding" Enhancements (Siêu tích hợp AI)
Hệ thống không chỉ dừng lại ở OCR đơn thuần mà tích hợp AI sâu vào quy trình xử lý dữ liệu:

- **Dictionary Dynamic Injection**: Tự động tiêm từ khóa từ "Từ điển AI" vào System Prompt thông qua hàm `buildDynamicPrompt`, giúp AI nhận diện Vendor/Sloc chính xác mà không cần sửa code.
- **AI Fuzzy Matcher (`aiRefineMatches`)**: Khi mapping cứng bằng từ khóa thất bại, hệ thống tự động gọi một "Agent AI" thứ hai để thực hiện so khớp mờ (Fuzzy Matching) giữa dữ liệu thô và Master Data.
- **AI Data Guard (Nâng cao)**: Kiểm tra logic nghiệp vụ phức tạp như: số lượng bất thường, sai lệch Sloc, logic ngày tháng, và đưa ra cảnh báo thông minh.
- **AI Chat Assistant**: Widget chat nổi tích hợp sẵn, hiểu ngữ cảnh dữ liệu hiện tại để trả lời các câu hỏi về lỗi mapping, hướng dẫn xử lý, hoặc giải thích dữ liệu.

### 2. UI/UX & Frontend Architecture (Chuẩn Senior Architect)
Tối ưu hóa giao diện theo tiêu chuẩn B2B SaaS hiện đại:

- **Skeleton Loading**: Hiệu ứng tải giả lập (Skeleton) trong lúc AI đang trích xuất, giúp giảm cảm giác chờ đợi và tăng trải nghiệm người dùng.
- **Empty State UI**: Giao diện "Trạng thái trống" chuyên nghiệp với icon minh họa và hướng dẫn rõ ràng khi chưa có dữ liệu.
- **Vendor Selection Dropdown**: Thêm tùy chọn chọn Vendor thủ công (Override) bên cạnh cơ chế tự động nhận diện của AI.
- **DOM Metadata Readiness**: Sử dụng `data-row-index` và `data-column` trên toàn bộ bảng dữ liệu, tách biệt hoàn toàn giữa UI hiển thị và Logic xử lý dữ liệu ngầm.
- **Tailwind CSS Optimization**: Refactor code CSS, sử dụng các animation tùy chỉnh (`pulse`, `fade-in`) và cấu trúc utility-first sạch sẽ.

### 3. Cấu hình Model AI & Hiệu năng
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

#### Kiến trúc Logic Nghiệp vụ FMCG (JavaScript + AI Prompts)
| Khối Logic | Mô tả Cơ Chế |
|---|---|
| **VENDOR_RULESET Động** | Xóa bảng Markdown tĩnh, tiêm cấu hình ruleset đọc AI cho từng Vendor cụ thể qua Object JSON cấu trúc động. |
| **Token-Based Fuzzy Search** | Tìm Cột (Material) và Khách hàng (Customer) theo cơ chế chẻ nhỏ từ khóa (`split(/\\W+/)`), triệt tiêu lỗi format dị của AI (VD: `BigC / GO!` vẫn tự động tra thủng `BigC`). |
| **Luật Override Tự Động** | Nếu Vendor (Pharmacity, Bibomart...) không in pháp nhân Sold-To trên hóa đơn, hệ thống tự động gán dữ liệu `Công ty VitaDairy` qua `VENDOR_SOLDTO_DEFAULTS`. |
| **Fallback Trực Tiếp** | Nếu Master Lookup (mã Code) thất bại, hệ thống tự động chèn Raw Text Data nguyên bản từ quá trình OCR (Mã SP, Tên, Cửa hàng) thả thẳng vào Excel chống để khoanh trắng. |
| **Bảo Vệ GuardAI** | AiDataGuard check chéo để cảnh báo nếu lệch Sloc (`O1KM`/`O1B1`) hoặc số lượng đột biến (`>1000`). Bỏ rule check "cấm ngày PO quá hạn" do thực tiễn Vendor. |

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

## Hướng dẫn Sử dụng (User Guide)

### Bước 1: Cấu hình Ban đầu
1. Nhấn nút **"Cấu hình AI"** ở góc trên bên phải.
2. Nhập **Gemini API Key** (lấy từ [Google AI Studio](https://aistudio.google.com/)).
3. Tải lên file **Customer Master** và **Material Master** (dùng template có sẵn trên giao diện). Hệ thống sẽ tự động lưu vào bộ nhớ trình duyệt.
4. (Tùy chọn) Cập nhật **Từ điển AI** để tối ưu hóa nhận diện chuỗi cửa hàng.

### Bước 2: Xử lý Chứng từ
1. Kéo thả tối đa 10 file PDF/ảnh vào vùng **"Tải chứng từ"**.
2. Chọn **Vendor** cụ thể từ dropdown nếu muốn ép AI trích xuất theo chuỗi nhất định, hoặc để mặc định để AI tự nhận diện.
3. Nhấn **"Trích xuất AI"**. 
   - Hệ thống sẽ hiển thị **Skeleton Loading** trong lúc xử lý.
   - AI sẽ tự động chạy 2 giai đoạn: **OCR Gốc** → **AI Fuzzy Matching** để khớp Master Data.
4. Kiểm tra các dòng có trạng thái `UNMATCHED` (màu đỏ) hoặc `AMBIGUOUS` (màu vàng). 

### Bước 3: Tương tác & Hỗ trợ AI
1. Nếu có thắc mắc về dữ liệu, hãy sử dụng **Chat Assistant** ở góc dưới bên phải.
2. Chatbot có thể giải thích tại sao một sản phẩm không khớp hoặc hướng dẫn bạn cách chỉnh sửa.
3. Chỉnh sửa trực tiếp trên bảng dữ liệu nếu AI trích xuất chưa hoàn hảo.

### Bước 4: Xuất Dữ liệu
1. Nhấn **"Xuất Excel"** để tải file `.xlsx` chuẩn 38 cột SAP.
2. Lưu ý: File Excel chỉ chứa các dòng đã được chọn trên giao diện.

---

## Xử lý Lỗi thường gặp (Troubleshooting)

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| `API Error 429` | Quá tải hạn mức API Free | Chờ 1 phút hoặc đổi API Key khác. Dưới gầm hệ thống đã cài Auto-Fallback xuống `1.5-flash`. |
| Giao diện xuất Excel ra tên Thô (Chưa chuẩn SAP) | Do file Master Data không khớp mã | Hệ thống sẽ tự động gọi **AI Fuzzy Matcher** để cứu vãn. Nếu vẫn không được, hãy cập nhật "Từ điển AI". |
| `QTY_PARSE_ERROR` | AI không đọc được số lượng rõ ràng | Kiểm tra bằng **Chat Assistant** hoặc chỉnh sửa thủ công cột "Order Quantity". |
| Skeleton Loading chạy mãi không dừng | Lỗi kết nối mạng hoặc API treo | Reload trang và thử lại với 1 file duy nhất để debug. |
| Lỗi Load Giao Diện / Không hiển thị PDF | Trình duyệt chặn iframe hoặc file quá nặng | Dùng trình duyệt Chrome/Edge mới nhất. Dọn rác cache IndexedDB thường xuyên. |

---

## Bảo trì & Nâng cấp (Maintenance)

- **Cập nhật Logic AI**: Sửa đổi hàm `buildDynamicPrompt` trong `index.html` nếu muốn thay đổi cách AI tiếp cận dữ liệu.
- **Thêm chuỗi siêu thị mới**: Vào **"Từ điển AI"** để thêm từ khóa nhận diện cho chuỗi đó mà không cần sửa code.
- **Tối ưu UI**: Toàn bộ UI nằm trong khối `<style>` và các class Tailwind trong `index.html`.
- **Dọn dẹp bộ nhớ**: Nếu Master Data quá lớn làm chậm trình duyệt, hãy nhấn "Xóa tất cả" và nạp lại bản rút gọn.

---

## Kế hoạch Kiểm tra (Verification)

| Bước | Mô tả | Kết quả mong đợi |
|---|---|---|
| 1 | Mở `index.html` trên trình duyệt | Giao diện hiển thị **Empty State** chuyên nghiệp |
| 2 | Nhập API Key, lưu cấu hình | Toast "Đã lưu", reload vẫn còn key |
| 3 | Upload 3-5 file PDF cùng lúc | Queue hiển thị đủ, preview đúng file |
| 4 | Bấm Trích xuất AI | **Skeleton Loading** hiển thị, 3 file xử lý song song |
| 5 | AI Fuzzy Matching chạy | Các dòng không khớp cứng được AI cứu vãn tự động |
| 6 | Mở Chat Assistant | Chatbot trả lời đúng ngữ cảnh dữ liệu hiện tại |
| 7 | Xuất Excel | File `.xlsx` tải về, đủ 38 cột SAP chuẩn |
| 8 | Kiểm tra Console log | `[COST]` log xuất hiện với số token thực tế |
| 9 | Kiểm tra hóa đơn Google Cloud | Chỉ thấy Gemini Flash, không có Pro |
| 10 | Kiểm tra Responsive | Giao diện hoạt động tốt trên các kích thước màn hình |
