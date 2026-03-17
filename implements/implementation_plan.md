# Kế hoạch Triển khai: Tính năng Upload Hàng loạt (Batch Upload)

Mục tiêu: Cho phép người dùng chọn và tải lên cùng lúc tối đa 10 tài liệu (PDF, Hình ảnh). Hệ thống sẽ đưa vào hàng đợi chờ xử lý, cung cấp chức năng lướt xem từng file, và nút "Trích xuất AI" sẽ lặp logic xử lý tự động cho toàn bộ files trong hàng đợi.

## Thay đổi đề xuất

### `index.html` (Frontend & Logic)

#### [MODIFY] index.html
1.  **Cập nhật Input Element:**
    *   Thêm thuộc tính `multiple` vào thẻ `<input type="file" id="fileInput">`.
    *   Thêm thuộc tính `accept=".pdf,image/*,.xls,.xlsx"` để giới hạn loại file phù hợp.
2.  **Quản lý State State (Trạng thái):**
    *   Thay thế biến `selectedFile` và `currentBase64Data` bằng một mảng `fileQueueData = []`. Mỗi phần tử chứa: `{ id, file, base64, status: 'pending'|'success'|'error', extractedData: [] }`.
    *   Khai báo biến lưu chỉ số file đang được Preview: `currentPreviewIndex = 0`.
3.  **UI Hàng đợi (File Queue):**
    *   Viết lại hàm render khu vực `id="fileQueue"` dể lặp qua mảng `fileQueueData` và in ra danh sách các file.
    *   Hỗ trợ giới hạn tải lên: `if(fileQueueData.length + newFiles.length > 10) showToast('Chỉ hỗ trợ tối đa 10 file cùng lúc')`.
    *   Click vào mỗi dòng trong Hàng đợi sẽ kích hoạt hàm Review (Xem trước) nội dung file đó ở khung `previewContainer`.
    *   Thêm nút (Icon thùng rác) để xóa từng file khỏi hàng đợi.
4.  **UI Nút 'Trích xuất AI':**
    *   Cập nhật logic nút bấm: Khi ấn, kiểm tra Mảng `fileQueueData`. Nếu trống thì báo lỗi.
    *   Duyệt qua vòng lặp (For loop) dựa trên mảng `fileQueueData`.
    *   Gọi hàm `extractDataWithGemini` tuần tự hoặc song song (Nên dùng tuần tự để tránh chạm giới hạn Rate Limit quota của Gemini quá nhanh nếu up 10 files).
    *   Gom tổng (Concat) mảng kết quả JSON của từng file vào một mảng `window.currentExtractedData` duy nhất.
5.  **Render Dữ liệu Bảng (Table Grid):**
    *   Sau mỗi lần xử lý từng file xong (hoặc xử lý xong lô 10 file), gọi lại hàm `renderTable(window.currentExtractedData)` để lấp đầy bảng bằng tổng dữ liệu của MỌI file.
    *   Cột **"Source File"** trong lưới Grid sẽ phát huy mạnh mẽ tác dụng ở đây, giúp user phân biệt dòng nào thuộc chứng từ PDF số mấy.
6.  **Xóa thẻ Đợi (Xóa tất cả):**
    *   Cập nhật lại logic nút "Xóa tất cả" để reset mảng `fileQueueData`, dọn dẹp biến, reset Table và Preview UI về rỗng.

## Kế hoạch Kiểm tra (Verification Plan)

### Kiểm thử Thủ công (Manual Review)
1. Tải ứng dụng trên Browser bằng công cụ Live Server hoặc click mở trực tiếp file `index.html`.
2. Kéo thả hoặc click chọn NHIỀU file PDF/Hình (3-5 files) cùng 1 lúc.
3. Kiểm tra UI Hàng đợi xem có hiển thị đủ danh sách file không.
4. Click luân phiên vào từng file trong hàng đợi xem khung Preview bên phải có load đúng nội dung file tương ứng không.
5. Nhấn "Trích xuất" và quan sát tiến trình: Icon trạng thái của từng file trong list Queue sẽ đổi từ `đang đợi` -> `xoay tròn` -> `thành công tick xanh`.
6. Kiểm tra Grid data bên phải có nối (append) đủ dữ liệu của toàn bộ số PO vừa chạy không. Số dòng phải bằng tổng các dòng SLOC+Lines của mọi documents.
7. Nhấn Export Excel kiểm tra file đầu ra xuất trọn vẹn mọi dòng của 5 files đó.
