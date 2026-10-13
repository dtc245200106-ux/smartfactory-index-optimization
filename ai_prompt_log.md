# Nhật ký Sử dụng AI
## Dự án: Tối ưu hóa Index - SmartFactory

### 🎯 Prompt 1: Covering Index là gì?
**Prompt:** "Covering Index trong MySQL là gì? Tại sao nó tăng tốc SELECT lên mức tối đa?"

**Trả lời:**
- **Covering Index** = Index chứa **tất cả các cột** cần thiết cho câu truy vấn (SELECT + WHERE + ORDER BY).
- MySQL **không cần lookup Table** — chỉ đọc từ Index.
- **Extra = 'Using index'** trong EXPLAIN.
- **Ví dụ:** `SELECT a, b FROM T WHERE c = ?` — nếu Index có `(c, a, b)` → Covering.

**Nhược điểm:** Index phình to → tốn Storage + chậm INSERT/UPDATE.

**Áp dụng:** Index cũ `idx_fat_covering` là Covering Index → Read nhanh nhưng Write chậm.

---

### 🎯 Prompt 2: Write Penalty của Index
**Prompt:** "Tại sao thêm 1 cột vào Index lại làm INSERT chậm đi?"

**Trả lời:**
- Mỗi INSERT phải cập nhật **tất cả các Index** trên bảng.
- Mỗi Index là 1 **cây B-Tree riêng** — phải sắp xếp lại vị trí.
- Càng nhiều cột → Node trong B-Tree càng lớn → **nhiều I/O hơn**.
- Càng nhiều Index → **nhiều cây B-Tree** phải cập nhật.

**Công thức:** `Write Penalty = Số Index × Kích thước Index × Số dòng INSERT`.

**Áp dụng:** Bỏ 3 cột khỏi Index → giảm Write Penalty.

---

### 🎯 Prompt 3: Clustered vs Secondary Index
**Prompt:** "MySQL phân bổ dữ liệu trên Clustered Index và Secondary Index như thế nào?"

**Trả lời:**
- **Clustered Index** = Primary Key. Dữ liệu thực được **sắp xếp theo PK**.
- **Secondary Index** = Index phụ. Lưu **PK** trỏ đến Clustered Index.
- **InnoDB:** Chỉ có **1 Clustered Index** (PK). Tất cả Index khác là Secondary.

**Hệ quả:**
- SELECT qua Secondary Index → cần **2 bước**: (1) Tìm PK trong Secondary, (2) Tìm dữ liệu trong Clustered.
- Covering Index giúp **bỏ qua bước 2** → nhanh hơn.

**Áp dụng:** Fat Index là Secondary nhưng Covering → bỏ qua bước 2.

---

### 🎯 Prompt 4: Tính dung lượng Index
**Prompt:** "Làm sao tính dung lượng Index trong MySQL theo bytes?"

**Trả lời:**
| Kiểu dữ liệu | Kích thước |
|---|---|
| `INT` | 4 bytes |
| `BIGINT` | 8 bytes |
| `DATETIME` | 5 bytes |
| `DECIMAL(5,2)` | 3 bytes |
| `VARCHAR(n)` | n + 1 byte (UTF-8) |
| `TINYINT` | 1 byte |

**Ví dụ:** Index `(sensor_id INT, recorded_at DATETIME, temperature DECIMAL(5,2))`:
- `4 + 5 + 3 = 12 bytes` + overhead B-Tree (~10 bytes) = **~22 bytes/dòng**.

**Áp dụng:** Tính dung lượng Fat Index vs Lean Index.

---

### 🎯 Prompt 5: Khi nào Covering Index KHÔNG phải tội ác?
**Prompt:** "Nếu bảng là Danh mục quốc gia (Countries), Covering Index có tốt không?"

**Trả lời:**
- **Bảng Read-heavy, ít Write** → Covering Index **RẤT TỐT**.
- Vì:
  - INSERT/UPDATE rất ít → Write Penalty không đáng kể.
  - SELECT nhiều → Covering Index tăng tốc đáng kể.
  - Dữ liệu ít thay đổi → Index không cần rebuild.
- **Ví dụ:** Countries, Categories, Products Catalog.

**Ngược lại:**
- **Bảng Write-heavy** (SensorLogs, Logs, Transactions) → Lean Index tốt hơn.

**Áp dụng:** Đánh giá đúng workload trước khi thiết kế.

---

### 🎯 Prompt 6: TINYINT thay VARCHAR
**Prompt:** "Nếu thay status VARCHAR(20) thành TINYINT, tác động thế nào?"

**Trả lời:**
- **VARCHAR(20)** → 20 bytes/dòng (UTF-8).
- **TINYINT** → 1 byte/dòng.
- **Tiết kiệm 19 bytes/dòng**.
- Nếu Index có 1M dòng → tiết kiệm **19 MB**.

**Nhược điểm:**
- Khó đọc hơn (phải mapping: 1=NORMAL, 2=WARNING, 3=CRITICAL).
- Khó debug.

**Áp dụng:** Nếu cần đưa status vào Index → dùng TINYINT.