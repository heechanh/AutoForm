# TỔNG QUAN DỰ ÁN: AUTOFORM DWH (DOCUGEN)
**Hệ thống Tự động Điền Biểu mẫu Tùy biến từ Dữ liệu Data Warehouse**

---

## 1. GIỚI THIỆU DỰ ÁN (PROJECT INTRODUCTION)

* **Tên tiếng Việt:** Hệ thống Tự động Điền Biểu mẫu DWH
* **Tên tiếng Anh:** DWH Form Filler (*DWH-DocGen*)
* **Bản chất dự án:** Nền tảng tự phục vụ (Self-service Document Automation Platform) cho phép người dùng tự thiết kế mẫu văn bản nghiệp vụ và tự động điền dữ liệu động từ Data Warehouse (DWH) để xuất văn bản hoàn chỉnh (.docx / .pdf).

Dự án hướng đến việc xóa bỏ rào cản phụ thuộc giữa người dùng nghiệp vụ (HR, Kế toán, Hành chính, Pháp chế) và đội ngũ kỹ thuật phần mềm (IT/Data Engineer) trong việc tạo và thay đổi các mẫu biểu hành chính.

---

## 2. BỐI CẢNH VÀ VẤN ĐỀ BÀI TOÁN (PROBLEM STATEMENT)

### 2.1. Thực trạng tại doanh nghiệp
* Doanh nghiệp đã đầu tư xây dựng **Data Warehouse (DWH)** lưu trữ tập trung và đầy đủ dữ liệu thực thể (hồ sơ nhân sự, lịch sử công tác, khách hàng, hợp đồng, phòng ban...).
* Các công việc hành chính như soạn thảo: **tờ trình, quyết định bổ nhiệm, thông báo tăng lương, thanh lý hợp đồng, biên bản đánh giá** vẫn đang được nhân viên xử lý thủ công qua Microsoft Word:
  1. Mở mẫu biểu trống có sẵn.
  2. Tra cứu thủ công mã nhân sự/đối tượng trên phần mềm hoặc DWH.
  3. Copy - Paste từng trường dữ liệu vào các vị trí tương ứng trong file Word.
  4. Rà soát, căn chỉnh định dạng và xuất file.

### 2.2. Điểm nghẽn và rủi ro (Pain Points)
1. **Lãng phí thời gian và nguồn lực:** Thao tác thủ công lặp đi lặp lại hàng trăm lần mỗi tháng với khối lượng văn bản lớn.
2. **Sai sót dữ liệu người dùng (Human Error):** Copy nhầm dòng, sót thông tin, định dạng ngày tháng/số tiền không chuẩn hóa, dẫn đến rủi ro pháp lý và hành chính.
3. **Phụ thuộc lập trình viên ("Bottleneck" IT):** Khi có quy định mới hoặc biểu mẫu thay đổi (thêm điều khoản, thay đổi font chữ, đổi vị trí hiển thị), người dùng không thể tự sửa nếu hệ thống xuất file bị "hardcode" logic trong code.
4. **Dữ liệu phân mảnh:** Dữ liệu chuẩn đã nằm trong DWH nhưng không được tái sử dụng tự động vào đầu ra cuối cùng của chuỗi nghiệp vụ (tài liệu/văn bản).

---

## 3. CÁCH GIẢI QUYẾT DỰ KIẾN (PROPOSED SOLUTION)

Hệ thống giải quyết bài toán theo triết lý **"Zero-Code & Self-Service"** — người dùng toàn quyền kiểm soát biểu mẫu mà không cần can thiệp mã nguồn:

### 3.1. Quy ước cú pháp Placeholder chuẩn hóa
* Cho phép người dùng nghiệp vụ tự soạn thảo trực tiếp trên file Word `.docx` quen thuộc.
* Đánh dấu các vị trí cần chèn dữ liệu bằng cú pháp biểu thức template (chuẩn Jinja2):
  * **Biến đơn:** `{{ ho_ten }}`, `{{ chuc_vu }}`, `{{ ma_nhan_vien }}`
  * **Bộ lọc định dạng:** `{{ ngay_vao_lam | format_date }}`, `{{ muc_luong | format_currency }}`
  * **Khối điều kiện & lặp:** Tự động ẩn hiện đoạn văn hoặc lặp dòng bảng khi có danh sách con (khen thưởng, kỷ luật, quá trình đào tạo).

### 3.2. Cơ chế Parser thông minh (Template Parsing Engine)
* Hệ thống tự động phân tích Abstract Syntax Tree (AST) của file Word khi người dùng tải lên.
* Tự động quét và lập danh sách tất cả placeholder có trong tài liệu mà **không cần khai báo trước trong code backend**.

### 3.3. Cấu hình Mapping động (Dynamic Schema Mapping)
* Cung cấp giao diện trực quan cho phép người dùng/quản trị viên ghép nối (map) giữa:
  $$\text{Placeholder trên biểu mẫu} \longleftrightarrow \text{Cột dữ liệu (Column/View) trong DWH}$$
* Cấu hình mapping được lưu trữ dưới dạng Metadata động (JSON/Database Config), cho phép cập nhật, tạo mới bất kỳ lúc nào mà không cần restart hệ thống hay sửa code.

### 3.4. Pipeline Truy vấn và Tạo văn bản tự động (Rendering Pipeline)
* Khi cần tạo tài liệu: Người dùng chỉ cần nhập/chọn **Mã định danh (Entity ID/Employee Code)**.
* Hệ thống:
  1. Đọc cấu hình mapping tương ứng của biểu mẫu.
  2. Tạo câu truy vấn DWH an toàn (chống SQL Injection) theo đúng các trường cần thiết.
  3. Lấy dữ liệu và thực hiện tiền xử lý (xử lý `NULL`, định dạng số, ngày tháng).
  4. Trộn dữ liệu vào template Word thông qua thư viện chuyên dụng (`docxtpl`).
  5. Xuất văn bản cuối cùng dưới dạng `.docx` hoặc convert sang `.pdf`.

---

## 4. KẾT QUẢ VÀ GIÁ TRỊ MANG LẠI (EXPECTED OUTCOMES)

| Tiêu chí | Trước khi có hệ thống | Sau khi áp dụng hệ thống |
| :--- | :--- | :--- |
| **Thời gian tạo 1 văn bản** | 5 – 15 phút / tài liệu | 1 – 3 giây / tài liệu |
| **Độ chính xác dữ liệu** | Phụ thuộc sự cẩn thận của con người | Chính xác tuyệt đối theo DWH |
| **Thay đổi biểu mẫu** | Chờ IT chỉnh sửa, deploy lại bản cập nhật | Người dùng tự sửa file Word và upload lại ngay lập tức |
| **Quy mô mở rộng** | Khó đáp ứng khi số lượng mẫu biểu tăng cao | Không giới hạn số lượng mẫu biểu và đối tượng xử lý |
