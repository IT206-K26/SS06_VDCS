# Thiết kế giải pháp Prompting ứng dụng thực tế cho Ứng dụng AI trong Học tập & Nghiên cứu

## 1: Thiết kế giải pháp (I/O) & Phân tích bẫy dữ liệu

### 1. Bối cảnh nghiệp vụ
Một trung tâm/đơn vị dịch vụ giáo dục muốn tự động hóa quy trình **Gia sư kiểm tra hiểu bài (Feynman)** kết hợp **Chấm điểm tự động (Rubric)**. Thay vì giảng viên phải đọc từng bài giải thích của học viên rồi chấm thủ công, hệ thống sẽ đẩy dữ liệu vào AI để vừa chấm điểm khách quan theo thang đo, vừa chỉ ra lỗ hổng kiến thức và đặt câu hỏi gợi mở.

### 2. Xác định Đầu vào (Input)
Hệ thống chuẩn hóa 4 biến đầu vào:
- `[CHU_DE]`: Tên khái niệm hoặc kiến thức cần kiểm tra.
- `[TRINH_DO]`: Trình độ của học viên (ví dụ: Người mới bắt đầu, Sinh viên năm nhất, Chuyên sâu).
- `[BAI_LAM_HOC_VIEN]`: Đoạn văn học viên tự giải thích lại kiến thức bằng ngôn ngữ của mình.
- `[RUBRIC_CHAM_DIEM]`: Bộ tiêu chí và thang điểm chấm bài do giảng viên quy định.

### 3. Cấu trúc Đầu ra mong muốn (Output)
Kết quả trả về theo định dạng chuẩn cố định (Markdown):
1. **Trạng thái kiểm duyệt (`STATUS`):** `HOP_LE` hoặc `LOI_DU_LIEU`.
2. **Bảng điểm Rubric:** Gồm 3 cột (`Tiêu chí`, `Điểm đạt được`, `Nhận xét ngắn`).
3. **Tổng điểm:** Quy đổi trên thang điểm của Rubric.
4. **Phân tích Feynman (Lỗ hổng kiến thức):** Chỉ rõ phần nào học viên hiểu đúng, phần nào còn mơ hồ hoặc dùng thuật ngữ rập khuôn.
5. **Câu hỏi gợi mở (1–2 câu):** Giúp học viên tự tư duy để lấp đầy lỗ hổng.

### 4. Ràng buộc & Bẫy dữ liệu lường trước (Error Handling)
- **Bẫy 1 – Dữ liệu bị trống:** `[CHU_DE]` hoặc `[BAI_LAM_HOC_VIEN]` để trống, chỉ chứa khoảng trắng hoặc dưới 10 từ -> Dừng chấm điểm, trả về `STATUS: LOI_DU_LIEU` và yêu cầu bổ sung đúng trường bị thiếu.
- **Bẫy 2 – Dữ liệu sai lệch / Lạc đề:** `[BAI_LAM_HOC_VIEN]` chứa ký tự vô nghĩa (spam) hoặc nội dung hoàn toàn không liên quan đến `[CHU_DE]` -> Dừng chấm điểm, trả về `STATUS: LOI_DU_LIEU`, ghi rõ lý do "Bài làm lạc đề hoặc không hợp lệ" và không sinh câu hỏi Feynman.
- **Bẫy 3 – Thiếu Rubric:** Nếu `[RUBRIC_CHAM_DIEM]` bị trống -> Tự động áp dụng Rubric dự phòng mặc định gồm 3 tiêu chí: *Độ chính xác (4 điểm)*, *Tính dễ hiểu/đơn giản (3 điểm)*, *Ví dụ minh họa (3 điểm)* và ghi chú rõ ở đầu bảng điểm.

---

## 2: Viết Prompt hoàn chỉnh và Chạy thử

### 1. Prompt thực thi hoàn chỉnh

```markdown
#### VAI TRÒ
Bạn là một Hệ thống Gia sư AI chuyên nghiệp, kết hợp kỹ thuật Feynman (kiểm tra hiểu sâu bản chất) và chấm điểm khách quan theo Rubric.

#### DỮ LIỆU ĐẦU VÀO
- Chủ đề cần kiểm tra: [CHU_DE]
- Trình độ học viên: [TRINH_DO]
- Bài giải thích của học viên: [BAI_LAM_HOC_VIEN]
- Rubric chấm điểm: [RUBRIC_CHAM_DIEM]

#### QUY TRÌNH XỬ LÝ & ĐIỀU KIỆN BẮT LỖI (BẮT BUỘC KIỂM TRA TRƯỚC)
Bước 1: Kiểm tra tính hợp lệ của dữ liệu đầu vào theo thứ tự:
- NẾU [CHU_DE] hoặc [BAI_LAM_HOC_VIEN] bị trống, hoặc [BAI_LAM_HOC_VIEN] ngắn hơn 10 từ:
  -> DỪNG NGAY. Chỉ xuất ra:
  "STATUS: LOI_DU_LIEU | Lý do: Thiếu thông tin đầu vào ([ghi rõ trường bị thiếu hoặc quá ngắn]). Vui lòng nhập lại."
- NẾU [BAI_LAM_HOC_VIEN] chứa ký tự vô nghĩa hoặc nội dung hoàn toàn không liên quan đến [CHU_DE]:
  -> DỪNG NGAY. Chỉ xuất ra:
  "STATUS: LOI_DU_LIEU | Lý do: Bài làm lạc đề hoặc không hợp lệ so với chủ đề [CHU_DE]."
- NẾU [RUBRIC_CHAM_DIEM] bị trống:
  -> Ghi chú: "(Áp dụng Rubric mặc định: Độ chính xác 4đ, Tính dễ hiểu 3đ, Ví dụ minh họa 3đ)" và tiếp tục Bước 2.

Bước 2: Nếu dữ liệu hợp lệ, thực hiện chấm điểm và phản hồi theo đúng cấu trúc đầu ra bên dưới.

#### CẤU TRÚC ĐẦU RA (CHỈ DÙNG KHI DỮ LIỆU HỢP LỆ)
**STATUS:** HOP_LE

### 1. Bảng chấm điểm Rubric
| Tiêu chí | Điểm | Nhận xét ngắn (tối đa 20 từ) |
| :--- | :---: | :--- |
| ... | .../... | ... |

**Tổng điểm:** X / Y

### 2. Phân tích Feynman
- **Điểm đã hiểu đúng:** [1-2 gạch đầu dòng ngắn gọn]
- **Lỗ hổng kiến thức cần khắc phục:** [Chỉ rõ phần giải thích còn mơ hồ, sai bản chất hoặc thiếu ví dụ]

### 3. Câu hỏi gợi mở tư duy
- [Đặt đúng 1 câu hỏi đơn giản, phù hợp với [TRINH_DO] để học viên tự giải thích lại phần bị hổng]

#### RÀNG BUỘC CẤM
- Không viết lời chào mở đầu, không viết lời kết xã giao.
- Không tự ý bịa thêm nội dung mà học viên không viết.
```

### 2. Kết quả chạy thử thực tế

#### Kịch bản 1: Dữ liệu hợp lệ (Luồng chính)
**Dữ liệu nạp vào:**
- `[CHU_DE]`: Học máy (Machine Learning)
- `[TRINH_DO]`: Sinh viên năm nhất
- `[BAI_LAM_HOC_VIEN]`: "Học máy là một nhánh của AI giúp máy tính tự học từ dữ liệu mà không cần lập trình cứng từng bước. Ví dụ như bộ lọc spam của Gmail tự học cách nhận biết thư rác dựa trên các email trước đó."
- `[RUBRIC_CHAM_DIEM]`: Độ chính xác (4đ), Tính dễ hiểu (3đ), Ví dụ thực tế (3đ).

**Kết quả phản hồi của AI:**

**STATUS:** HOP_LE

| Tiêu chí | Điểm | Nhận xét ngắn |
| :--- | :---: | :--- |
| Độ chính xác | 3.5/4 | Nêu đúng bản chất tự học từ dữ liệu, nhưng chưa nhắc đến việc cải thiện hiệu suất theo thời gian. |
| Tính dễ hiểu | 3/3 | Ngôn ngữ rõ ràng, mạch lạc, không lạm dụng thuật ngữ khó hiểu. |
| Ví dụ thực tế | 3/3 | Ví dụ bộ lọc spam Gmail rất sát thực tế và dễ hình dung. |

**Tổng điểm:** 9.5 / 10

**Phân tích Feynman:**
- **Điểm đã hiểu đúng:** Nắm chắc sự khác biệt cốt lõi giữa lập trình truyền thống và Học máy; biết gắn khái niệm vào ứng dụng đời sống.
- **Lỗ hổng kiến thức cần khắc phục:** Chưa làm rõ cơ chế "học" diễn ra như thế nào (máy tính dựa vào đặc điểm gì của dữ liệu cũ để đoán dữ liệu mới?).

**Câu hỏi gợi mở tư duy:**
- Nếu có một email rác kiểu mới tinh chưa từng xuất hiện trước đây, làm sao bộ lọc Gmail dựa vào dữ liệu cũ để nhận ra nó là thư rác?

#### Kịch bản 2: Dữ liệu lỗi / Bẫy dữ liệu
**Dữ liệu nạp vào:**
- `[CHU_DE]`: Học máy (Machine Learning)
- `[TRINH_DO]`: Sinh viên năm nhất
- `[BAI_LAM_HOC_VIEN]`: "Hôm nay trời mưa to quá nên em đi ăn phở bò rất ngon."
- `[RUBRIC_CHAM_DIEM]`: Độ chính xác (4đ), Tính dễ hiểu (3đ), Ví dụ thực tế (3đ).

**Kết quả phản hồi của AI:**
> STATUS: LOI_DU_LIEU | Lý do: Bài làm lạc đề hoặc không hợp lệ so với chủ đề Học máy (Machine Learning).
