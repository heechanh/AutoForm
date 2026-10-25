# TÀI LIỆU ĐẶC TẢ YÊU CẦU HỆ THỐNG (SRS)
## DỰ ÁN: HỆ THỐNG TỰ ĐỘNG ĐIỀN BIỂU MẪU TÙY BIẾN TỪ DATA WAREHOUSE (DWH)

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1. Bối cảnh
Doanh nghiệp sở hữu Data Warehouse (DWH) lưu trữ tập trung dữ liệu thực thể (nhân sự, phòng ban, đối tác, khách hàng...). Tuy nhiên, việc xuất bản các văn bản hành chính/nghiệp vụ (tờ trình, quyết định, hợp đồng, biên bản, thông báo) đang phụ thuộc vào việc sao chép thủ công. Việc này dẫn đến:
- Tốn nhiều thời gian và nhân lực.
- Nguy cơ sai sót dữ liệu thực tế so với DWH.
- Thiếu tính linh hoạt: Mỗi lần mẫu biểu thay đổi lại cần can thiệp lập trình viên (hardcode template).

### 1.2. Mục tiêu cốt lõi
Xây dựng giải pháp phần mềm tự phục vụ (Self-service Document Automation Platform):
- Cho phép người dùng nghiệp vụ tự tạo và chỉnh sửa mẫu tài liệu (file `.docx`) bằng cách cắm placeholder (biến giữ chỗ).
- Hệ thống tự động phân tích (parse) template, trích xuất placeholder, quản lý mapping với schema của DWH.
- Tự động query DWH theo mã định danh (Entity ID/Employee Code...) và kết xuất văn bản hoàn chỉnh định dạng `.docx` hoặc `.pdf`.

---

## 2. KIẾN TRÚC TỔNG THỂ & QUY ƯỚC CÚ PHÁP

### 2.1. Quy ước cú pháp Placeholder
Hệ thống sử dụng cú pháp tương thích Jinja2 để tận dụng các bộ thư viện như `docxtpl`:
- **Biến đơn (Scalar variables):** `{{ ho_ten }}`, `{{ chuc_vu }}`, `{{ ngay_vao_lam | date_format }}`
- **Khối điều kiện (Conditionals):**
  ```jinja2
  {% if gioi_tinh == 'Nam' %}Ông{% else %}Bà{% endif %}
  ```
- **Khối lặp (Tables / Lists):**
  ```jinja2
  {% tr for item in danh_sach_khen_thuong %}
  {{ item.so_quyet_dinh }} | {{ item.ngay_ky }} | {{ item.hinh_thuc }}
  {% tr endfor %}
  ```
  *(Lưu ý: Thẻ `{% tr %}` dùng để lặp nguyên dòng trong bảng Word với docxtpl).*

### 2.2. Luồng hoạt động chính (Workflow)
1. **Thiết kế & Tải lên:** User tải file `.docx` chứa placeholder lên hệ thống.
2. **Parser & Trích xuất:** Backend phân tích AST / regex của file template để lấy danh sách placeholder duy nhất.
3. **Cấu hình Mapping:** User gán từng placeholder tương ứng với trường/cột trong DWH hoặc biểu thức SQL/biến tính toán.
4. **Sinh tài liệu (Generation):** User chọn Template $\rightarrow$ nhập/chọn Entity ID $\rightarrow$ Hệ thống query DWH $\rightarrow$ Render dữ liệu $\rightarrow$ Xuất file Word/PDF.

---

## 3. ĐẶC TẢ CHI TIẾT CÁC CHỨC NĂNG (FUNCTIONAL REQUIREMENTS)

| Mã CN | Tên chức năng | Mô tả chi tiết |
| :--- | :--- | :--- |
| **FN-01** | **Quản lý & Tải lên Biểu mẫu (Template Management)** | - Cho phép Upload file `.docx`.<br>- Quản lý danh sách template, phiên bản (versioning), trạng thái (Active/Draft/Archived). |
| **FN-02** | **Bộ bóc tách Placeholder (Template Parser)** | - Tự động đọc file `.docx` (cả thân bài viết, Header, Footer, Bảng).<br>- Trích xuất tất cả các biến dạng `{{ ... }}` và các block lặp `{% for ... %}`.<br>- Kiểm tra lỗi cú pháp (Syntax validation) trước khi lưu. |
| **FN-03** | **Cấu hình Ánh xạ Động (Dynamic Mapping Engine)** | - Giao diện trực quan cho phép map: `Placeholder` $\leftrightarrow$ `DWH Field`.<br>- Hỗ trợ map bảng chi tiết (1-N) cho các danh sách lặp.<br>- Hỗ trợ hàm biến đổi (Transformations): Định dạng ngày tháng, viết hoa, định dạng số/tiền tệ ($1,000,000 \text{ VND}$).<br>- Cấu hình giá trị mặc định khi dữ liệu DWH bị `NULL` / rỗng. |
| **FN-04** | **Truy vấn DWH (Data Extraction Engine)** | - Kết nối an toàn đến DWH (PostgreSQL, ClickHouse, Snowflake, SQL Server...).<br>- Thực thi query theo Entity ID (ví dụ: `employee_id = ?`).<br>- Hỗ trợ cơ chế cache metadata schema để người dùng dễ chọn cột từ DWH. |
| **FN-05** | **Sinh & Kết xuất Tài liệu (Document Generation)** | - Merge data từ DWH vào Template.<br>- Xử lý các placeholder chưa map (cảnh báo hoặc điền chuỗi trống).<br>- Chuyển đổi linh hoạt sang PDF (sử dụng LibreOffice headless hoặc Gotenberg).<br>- Cho phép sinh hàng loạt (Batch Generation) theo danh sách ID. |
| **FN-06** | **Lịch sử & Nhật ký (Audit & Generation Log)** | - Lưu vết ai đã sinh tài liệu, thời gian, Entity ID nào, template nào.<br>- Tải lại tài liệu đã sinh trong vòng $N$ ngày. |

---

## 4. CHI TIẾT INPUT - OUTPUT THEO TỪNG CÔNG ĐOẠN

### 4.1. Công đoạn Tải lên & Phân tích Template (FN-01, FN-02)
- **Input:** 
  - File tài liệu mẫu: Định dạng `.docx`, kích thước $\le 20\text{MB}$.
  - Metadata: Tên biểu mẫu, Mã nhóm văn bản (Nhân sự, Pháp chế...), Mô tả.
- **Output:**
  - Bản ghi Template trong CSDL.
  - Danh sách trích xuất các placeholder:
    ```json
    {
      "template_id": "TPL_HR_001",
      "placeholders": [
        {"name": "ho_ten", "type": "scalar"},
        {"name": "chuc_vu", "type": "scalar"},
        {"name": "danh_sach_khen_thuong", "type": "iterable", "fields": ["so_qd", "ngay_qd", "noi_dung"]}
      ]
    }
    ```

### 4.2. Công đoạn Cấu hình Mapping (FN-03)
- **Input:**
  - `template_id`.
  - Cấu hình mapping chi tiết từ người dùng:
    ```json
    {
      "template_id": "TPL_HR_001",
      "source_dataset": "dim_employee",
      "identity_key": "employee_code",
      "mappings": [
        {"placeholder": "ho_ten", "dwh_column": "full_name", "fallback": "N/A"},
        {"placeholder": "ngay_sinh", "dwh_column": "dob", "format": "DD/MM/YYYY"},
        {
          "placeholder": "danh_sach_khen_thuong",
          "dwh_table": "fact_rewards",
          "join_key": "employee_id",
          "sub_mappings": {
            "so_qd": "reward_decision_no",
            "ngay_qd": "decision_date"
          }
        }
      ]
    }
    ```
- **Output:**
  - File/Record JSON lưu trong database hoặc file cấu hình YAML được kích hoạt và kiểm tra tính hợp lệ (Validation status: `VALID`).

### 4.3. Công đoạn Sinh tài liệu (FN-04, FN-05)
- **Input:**
  - `template_id`.
  - `entity_id` (ví dụ: `EMP20261025`).
  - Tùy chọn định dạng đầu ra: `DOCX` hoặc `PDF`.
- **Output:**
  - File hoàn chỉnh `.docx` hoặc `.pdf` đã được điền đầy đủ dữ liệu từ DWH.
  - Mã trạng thái xử lý (Success / Warning nếu thiếu data / Error).

---

## 5. XỬ LÝ CÁC TRƯỜNG HỢP BIÊN & NGOẠI LỆ (EDGE CASES)

1. **Placeholder chưa được mapping:**
   - **Xử lý:** Hệ thống cảnh báo trước khi xuất file. Cho phép cấu hình hiển thị: Giữ nguyên `{{ placeholder }}`, thay bằng khoảng trống `____`, hoặc dừng tiến trình báo lỗi.
2. **Dữ liệu trong DWH là `NULL` hoặc rỗng:**
   - **Xử lý:** Áp dụng giá trị cấu hình `fallback` mặc định (ví dụ: để trống, ghi `"Không có"`, hoặc `"..."`).
3. **Template chứa lỗi cú pháp Jinja2 (ví dụ thiếu dấu đóng ngoặc `}}`):**
   - **Xử lý:** Từ chối upload ngay từ bước Parser, khoanh vùng dòng/đoạn văn bản bị lỗi để người dùng sửa.
4. **Văn bản chứa ký tự đặc biệt XML (`&`, `<`, `>`):**
   - **Xử lý:** Bộ sinh tài liệu tự động escape ký tự để không làm hỏng cấu trúc `document.xml` của file Word.

---

## 6. YÊU CẦU PHI CHỨC NĂNG & KỸ THUẬT (NON-FUNCTIONAL REQUIREMENTS)

### 6.1. Hiệu năng & Khả năng mở rộng
- **Thời gian xử lý:** Thời gian parse template $\le 2\text{ giây}$. Thời gian sinh 01 văn bản (kể cả query DWH) $\le 3\text{ giây}$ (DOCX) và $\le 5\text{ giây}$ (nếu chuyển PDF).
- **Mở rộng (Scalability):** Hệ thống không hardcode bất kỳ logic template nào; thêm 100 biểu mẫu mới không cần viết thêm dòng code nào hay restart hệ thống.

### 6.2. Stack Công nghệ khuyến nghị
- **Ngôn ngữ & Xử lý tài liệu:** Python 3.11+, `docxtpl` (kết hợp `python-docx` và `Jinja2`), `pydantic`.
- **Backend API:** FastAPI (nhẹ, nhanh, hỗ trợ async khi query DWH và I/O file).
- **Chuyển đổi PDF:** LibreOffice (chạy chế độ headless) hoặc container `Gotenberg`.
- **Lưu trữ Cấu hình & Quản lý:** PostgreSQL / MySQL (lưu metadata, config mapping), MinIO / S3 (lưu trữ file template và output).
- **Kết nối DWH:** SQLAlchemy / PySpark / DuckDB (tùy thuộc kiến trúc DWH hiện hữu).

---

## 7. KẾT QUẢ DỰ KIẾN (DELIVERABLES & EXPECTED OUTCOMES)

1. **Bộ mã nguồn (Source Code):**
   - Service Parser & Dynamic Template Engine.
   - API kết nối DWH và sinh file tự động.
   - Giao diện người dùng Web (UI): Upload template, UI kéo-thả/chọn cột DWH để map, giao diện nhập ID để sinh file.
2. **Tài liệu hướng dẫn (Documentation):**
   - Hướng dẫn cú pháp đặt placeholder cho người dùng văn phòng.
   - Hướng dẫn cấu hình kết nối DWH.
3. **Hiệu quả thực tế đạt được:**
   - Cắt giảm $90\%$ thời gian tạo văn bản hành chính so với điền thủ công.
   - Triệt tiêu hoàn toàn sai lệch số liệu giữa văn bản in ra và dữ liệu nguồn trong Data Warehouse.
   - Tính tự phục vụ (Self-service) đạt $100\%$: Người dùng phòng ban tự đổi mẫu văn bản trong Word mà không cần IT can thiệp.