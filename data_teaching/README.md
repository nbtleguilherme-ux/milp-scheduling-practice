# Lập Lịch Khóa Học — Tập Dữ Liệu Giảng Dạy

Một bài toán lập lịch nhỏ, khép kín được thiết kế cho buổi thực hành MILP lập lịch khóa học đầu tiên.

## Phát Biểu Bài Toán

Gán mỗi khóa học đúng **một khung giờ** và **một phòng học** sao cho tất cả các ràng buộc cứng
được thỏa mãn và tổng chi phí lập lịch được tối thiểu hóa.

---

## Các Tệp Dữ Liệu

| Tệp | Số dòng | Các cột chính |
|------|------|-------------|
| `courses.csv` | 10 | `course_id, name, lecturer_id, n_students, capacity_needed, preferred_period` |
| `rooms.csv` | 5 | `room_id, name, capacity, cost_per_slot` |
| `timeslots.csv` | 15 | `timeslot_id, day, period, start_time, cost` (3 ngày × 5 tiết) |
| `lecturers.csv` | 8 | `lecturer_id, name` |
| `course_conflicts.csv` | 8 | `course_a, course_b, shared_students` |
| `course_period_options.csv` | 6 | `course_id, period` (chỉ các tiết được phép cho khóa học bị hạn chế) |

---

## Tập Hợp & Tham Số

| Ký hiệu | Ý nghĩa | Nguồn |
|--------|---------|--------|
| **C** | tập hợp các khóa học | `courses.csv` |
| **R** | tập hợp các phòng học | `rooms.csv` |
| **T** | tập hợp các khung giờ | `timeslots.csv` |
| `cap[r]` | sức chứa của phòng *r* | `rooms.capacity` |
| `size[c]` | số sinh viên cần chỗ ngồi | `courses.capacity_needed` |
| `lec[c]` | giảng viên của khóa học *c* | `courses.lecturer_id` |
| `rc[r]` | chi phí phòng mỗi tiết | `rooms.cost_per_slot` |
| `tc[t]` | chi phí khung giờ | `timeslots.cost` |
| `E` | tập hợp các cặp khóa học xung đột (có sinh viên chung) | `course_conflicts.csv` |
| `A[c]` | các ID khung giờ được phép cho khóa học *c* (không hạn chế nếu vắng mặt) | `course_period_options.csv` |

---

## Biến Quyết Định

```
x[c, t, r] ∈ {0, 1}     cho tất cả c ∈ C, t ∈ T, r ∈ R

x[c,t,r] = 1  khi và chỉ khi khóa học c được xếp vào khung giờ t và phòng r
```

Với |C|=10, |T|=15, |R|=5 ta có **750 biến nhị phân** — nhỏ đủ để
kiểm tra bằng tay và giải trong vài giây với bất kỳ bộ giải MILP nào.

---

## Ràng Buộc Cứng

### C1 — Mỗi khóa học được phân công đúng một lần
```
∑_{t,r} x[c,t,r] = 1     ∀ c ∈ C
```

### C2 — Không có hai khóa học nào dùng chung (khung giờ, phòng)
```
∑_c x[c,t,r] ≤ 1     ∀ t ∈ T, r ∈ R
```

### C3 — Sức chứa phòng không được vượt quá
```
x[c,t,r] = 0     nếu cap[r] < size[c]
```
*(Tương đương: thêm ràng buộc này hoặc lọc trước các cặp (c,r) khả thi.)*

### C4 — Xung đột sinh viên: hai khóa học có sinh viên chung không thể trùng giờ
```
∑_r x[c1,t,r] + ∑_r x[c2,t,r] ≤ 1     ∀ (c1,c2) ∈ E, ∀ t ∈ T
```

### C5 — Xung đột giảng viên: một giảng viên dạy tối đa một khóa học mỗi khung giờ
```
∑_r x[c1,t,r] + ∑_r x[c2,t,r] ≤ 1     ∀ t ∈ T,
    ∀ c1 ≠ c2 sao cho lec[c1] = lec[c2]
```

*Các giảng viên dạy hai khóa học trong tập dữ liệu này:*
- **Prof. Nguyen (id=1)**: Math Analysis (1) + Statistics (9)
- **Prof. Le (id=3)**: Programming 101 (3) + Databases (6)

### C6 — Hạn chế tiết học (ràng buộc cứng tùy chọn)
Một số khóa học chỉ được dùng một số tiết nhất định.
`course_period_options.csv` liệt kê các **tiết được phép** cho các khóa học bị hạn chế:

| Khóa học | Các tiết được phép | Lý do |
|--------|----------------|-----------|
| Databases (6) | P3, P4, P5 | Phòng máy tính chỉ có sẵn từ 09:30 |
| ML Basics (10) | P1, P2, P3 | Chỉ dùng tiết giảng buổi sáng |

Các khóa học **không** có trong tệp đó có thể dùng **bất kỳ** tiết nào.

```
∑_r x[c,t,r] = 0     ∀ c có hạn chế, ∀ t có period ∉ A[c]
```

---

## Hàm Mục Tiêu

Tối thiểu hóa tổng chi phí lập lịch, trong đó mỗi lần phân công phải trả **chi phí phòng** và
**chi phí khung giờ**:

```
Tối thiểu hóa  ∑_{c,t,r}  (rc[r] + tc[t]) × x[c,t,r]
```

### Cấu Trúc Chi Phí

**Chi phí phòng** (`rc[r]`) phản ánh kích thước — phòng lớn hơn tốn kém hơn:

| Phòng | Sức chứa | Chi phí/tiết |
|------|----------|-----------|
| Hall A | 45 | 5 |
| Hall B | 40 | 4 |
| Room C | 30 | 3 |
| Room D | 25 | 2 |
| Room E | 20 | 1 |

**Chi phí khung giờ** (`tc[t]`) phản ánh mức độ ưu tiên theo thời điểm trong ngày:

| Tiết | Giờ bắt đầu | Chi phí | Diễn giải |
|--------|-------|------|----------------|
| P1 | 07:30 | 2 | Quá sớm |
| P2 | 08:30 | 1 | Lý tưởng |
| P3 | 09:30 | 1 | Lý tưởng |
| P4 | 10:30 | 2 | Muộn buổi sáng |
| P5 | 13:30 | 3 | Buổi chiều (nóng / mệt mỏi) |

Cấu trúc chi phí tạo ra một **mâu thuẫn tự nhiên**: khóa học lớn (nhiều sinh viên) *phải*
dùng phòng đắt tiền (C3), trong khi bộ tối ưu hóa cố gắng tránh lãng phí sức chứa dư thừa
và tránh các khung giờ có chi phí cao bất cứ khi nào ràng buộc cho phép.

---

## Bài Tập Mở Rộng — Hình Phạt Ưu Tiên Mềm

`courses.preferred_period` ghi lại tiết lý tưởng của mỗi khóa học.  
Thêm hình phạt mềm vào hàm mục tiêu:

```
Tối thiểu hóa  ∑_{c,t,r} (rc[r] + tc[t]) × x[c,t,r]
             + λ × ∑_{c,t,r} penalty[c,t] × x[c,t,r]

trong đó  penalty[c,t] = 1  nếu period(t) ≠ preferred_period[c]
                       = 0  ngược lại
```

Thử nghiệm với các giá trị λ khác nhau để xem bộ giải lệch khỏi
nghiệm tối ưu ràng buộc cứng bao nhiêu để tôn trọng các ưu tiên.

---

## Đồ Thị Xung Đột (Tham Khảo Nhanh)

```
Xung đột sinh viên (C4):
  1 — 2   (15 sinh viên)
  1 — 9   (10 sinh viên)
  2 — 5   (12 sinh viên)
  3 — 4   (18 sinh viên)
  3 — 6   (14 sinh viên)   ← cũng là xung đột giảng viên (Prof. Le)
  4 — 10  ( 8 sinh viên)
  5 — 10  (20 sinh viên)
  7 — 8   (16 sinh viên)

Xung đột giảng viên (C5, chưa có trong C4):
  1 — 9   (Prof. Nguyen)
```

---

## Ghi Chú Về Tính Khả Thi

- 10 khóa học, 15 khung giờ, 5 phòng → 75 ô (khung giờ, phòng) khả dụng, nhiều hơn nhiều so với cần thiết.
- Các ràng buộc ràng buộc nhất là sức chứa (các khóa học 1, 3, 10 đều cần Hall A hoặc B) và
  đồ thị xung đột giảng viên/sinh viên.
- Tập dữ liệu **được đảm bảo khả thi** khi cả sáu ràng buộc đều hoạt động.
