# Buổi 8 — Lập Lịch Khóa Học với MILP

Buổi thực hành về **Lập Trình Tuyến Tính Nguyên Hỗn Hợp (MILP)** ứng dụng vào lập lịch khóa học tại trường đại học.  
Bạn sẽ mô hình hóa và giải một bài toán lập lịch từ đầu sử dụng [Pyomo](http://www.pyomo.org/) và bộ giải HiGHS.

---

## Mục Tiêu

Sau buổi học này bạn sẽ có khả năng:

- Chuyển đổi một bài toán lập lịch thực tế thành biến quyết định, ràng buộc và hàm mục tiêu
- Triển khai mô hình MILP trong Pyomo (`ConcreteModel`, `Var`, `Constraint`, `Objective`)
- Sử dụng ràng buộc dạng quy tắc (rule-based) và `ConstraintList` cho các tập không đều
- Giải mô hình bằng bộ giải HiGHS và trích xuất nghiệm
- Mở rộng mô hình ràng buộc cứng bằng hình phạt ưu tiên mềm và phân tích sự đánh đổi

---

## Cấu Trúc Repository

```
milp_scheduling_practice.ipynb   ← notebook thực hành của bạn (hoàn thành các TODO)
eda_teaching_dataset.ipynb       ← phân tích khám phá tập dữ liệu
data_teaching/                   ← tập dữ liệu (xem bên dưới)
```

> Nhánh **`main`** chứa notebook thực hành.  
> Nhánh **`solution`** bổ sung lời giải tham khảo đầy đủ — được chia sẻ vào cuối buổi học.

---

## Tập Dữ Liệu — `data_teaching/`

Một bài toán lập lịch nhỏ, khép kín: **10 khóa học**, **5 phòng**, **15 khung giờ** (3 ngày × 5 tiết).

| Tệp | Số dòng | Mô tả |
|------|------|-------------|
| `courses.csv` | 10 | Mã khóa học, tên, giảng viên, số sinh viên, sức chứa cần thiết, tiết ưu tiên |
| `rooms.csv` | 5 | Mã phòng, tên, sức chứa, chi phí mỗi tiết |
| `timeslots.csv` | 15 | Mã khung giờ, ngày (Mon/Tue/Wed), tiết (P1–P5), giờ bắt đầu, chi phí |
| `lecturers.csv` | 8 | Mã giảng viên, tên |
| `course_conflicts.csv` | 8 | Các cặp khóa học có chung sinh viên (không thể trùng giờ) |
| `course_period_options.csv` | 6 | Các tiết được phép cho các khóa học bị hạn chế |

### Cấu Trúc Chi Phí

**Chi phí phòng** (phòng lớn hơn = chi phí cao hơn):

| Phòng | Sức chứa | Chi phí/tiết |
|------|----------|-----------|
| Hall A | 45 | 5 |
| Hall B | 40 | 4 |
| Room C | 30 | 3 |
| Room D | 25 | 2 |
| Room E | 20 | 1 |

**Chi phí khung giờ** (phản ánh mức độ ưu tiên theo thời điểm trong ngày):

| Tiết | Giờ bắt đầu | Chi phí |
|--------|-------|------|
| P1 | 07:30 | 2 — quá sớm |
| P2 | 08:30 | 1 — lý tưởng |
| P3 | 09:30 | 1 — lý tưởng |
| P4 | 10:30 | 2 — muộn buổi sáng |
| P5 | 13:30 | 3 — buổi chiều |

---

## Tóm Tắt Bài Toán

Gán mỗi khóa học vào đúng một cặp (khung giờ, phòng) để **tối thiểu hóa tổng chi phí**, với các ràng buộc:

| Ràng buộc | Quy tắc |
|------------|------|
| C1 — Phân công | Mỗi khóa học được lập lịch đúng một lần |
| C2 — Xung đột phòng | Mỗi (khung giờ, phòng) chỉ có tối đa một khóa học |
| C3 — Sức chứa | Sức chứa phòng ≥ quy mô lớp học |
| C4 — Xung đột sinh viên | Các khóa học có chung sinh viên không thể trùng giờ |
| C5 — Xung đột giảng viên | Một giảng viên không thể dạy hai khóa học cùng lúc |
| C6 — Hạn chế tiết *(tùy chọn)* | Một số khóa học chỉ được dùng một số tiết nhất định |

**Mở rộng (Phần 5):** thêm hình phạt mềm cho các khóa học được xếp ngoài tiết ưu tiên và điều chỉnh trọng số hình phạt λ.

---

## Cài Đặt

Khuyến nghị Python 3.9+. Cài đặt tất cả các phụ thuộc bằng:

```bash
pip install pyomo highspy pandas numpy matplotlib seaborn
```

| Gói | Mục đích |
|---------|---------|
| `pyomo` | Framework mô hình hóa MILP |
| `highspy` | Bộ giải HiGHS (gọi qua `appsi_highs` trong Pyomo) |
| `pandas` | Tải dữ liệu và bảng kết quả |
| `numpy` | Tiện ích số học |
| `matplotlib` | Lưới lịch học và biểu đồ mức sử dụng |
| `seaborn` | Định dạng đồ thị |

### Kiểm Tra Cài Đặt

Chạy hai ô đầu tiên của `milp_scheduling_practice.ipynb`. Bạn sẽ thấy:

```
Pyomo sẵn sàng.
Dữ liệu đã tải.
  courses      (10, 6)
  rooms        (5, 4)
  timeslots    (15, 5)
  ...
```

---

## Hướng Dẫn Làm Notebook

1. Đọc từng tiêu đề `## Phần N` và công thức toán học trong ô markdown phía trên.
2. Hoàn thành mọi ô có đánh dấu **`# TODO`** — gợi ý và mẫu Pyomo đã được cung cấp dưới dạng chú thích.
3. Chạy Phần 4 để giải và kiểm tra kết quả (chi phí tối ưu phải là **30**).
4. Hoàn thành Phần 5 (ràng buộc mềm) và thử nghiệm với các giá trị λ khác nhau.
5. Hoàn thành Phần 6 Bước 1 để tạo các biểu đồ trực quan.
