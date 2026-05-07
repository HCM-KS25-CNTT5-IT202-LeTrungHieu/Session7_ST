# [Phân tích] Thảm họa "Black Friday"

---

# 1. Phân tích sự cố

## Hiện trường (query gây thảm họa)

```sql
SELECT
    *
FROM
    courses
WHERE
    id NOT IN (
        SELECT
            course_id
        FROM
            enrollments
    );
```

---

## Triệu chứng

```text
Query trả về 0 dòng dữ liệu
```

Trong khi thực tế:

- vẫn còn rất nhiều khóa học chưa ai đăng ký.

---

## Nguyên nhân bí ẩn

Trong bảng:

```text
enrollments
```

xuất hiện dữ liệu rác:

| course_id |
| --------- |
| 1         |
| 2         |
| NULL      |

---

# 2. Bản chất nguy hiểm của `NOT IN`

---

## Điều mà Dev nghĩ

Dev tưởng query sẽ hiểu như:

```sql
id NOT IN (1, 2)
```

---

## Nhưng thực tế hệ thống đang xử lý

```sql
id NOT IN (1, 2, NULL)
```

---

# 3. Logic toán học phía sau `NOT IN`

---

## SQL sẽ biến đổi như sau

Ví dụ:

```sql
id NOT IN (1, 2, NULL)
```

Database sẽ hiểu thành:

```sql
id <> 1
AND id <> 2
AND id <> NULL
```

---

# 4. Đây là nơi thảm họa xảy ra

---

## Trong SQL

```sql
NULL không phải là giá trị
```

Nó mang nghĩa:

```text
UNKNOWN / Không biết
```

---

## Vì vậy

```sql
5 <> NULL
```

KHÔNG trả về:

```text
TRUE
```

mà trả về:

```text
UNKNOWN
```

---

# 5. Boolean Logic của SQL (3-valued logic)

SQL không chỉ có:

```text
TRUE
FALSE
```

mà có tới:

```text
TRUE
FALSE
UNKNOWN
```

---

# 6. Phân tích từng bước

Ví dụ với:

```sql
id = 5
```

---

SQL sẽ kiểm tra:

```sql
5 <> 1
AND 5 <> 2
AND 5 <> NULL
```

---

## Kết quả

```text
TRUE
AND TRUE
AND UNKNOWN
```

---

## Trong Boolean Logic của SQL

```text
TRUE AND UNKNOWN
= UNKNOWN
```

---

# 7. Điều chết người

Trong mệnh đề:

```sql
WHERE
```

Database chỉ giữ lại dòng khi:

```text
TRUE
```

---

Còn nếu:

```text
FALSE
hoặc
UNKNOWN
```

→ đều bị loại bỏ.

---

# 8. Hệ quả cuối cùng

Mọi dòng đều thành:

```text
UNKNOWN
```

→ toàn bộ kết quả:

```text
EMPTY SET
```

---

# 9. Đây là bug NULL kinh điển nhất của `NOT IN`

---

## Chỉ cần

```text
1 NULL duy nhất
```

xuất hiện trong subquery:

```sql
NOT IN (...)
```

có thể:

```text
phá hủy toàn bộ kết quả
```

---

# 10. Giải pháp kiến trúc

---

## Cách vá tối thiểu

Phải loại bỏ NULL ngay bên trong subquery.

---

## Thêm điều kiện

```sql
WHERE course_id IS NOT NULL
```

---

# 11. Query chống lỗi hoàn chỉnh

```sql
SELECT
    *
FROM
    courses
WHERE
    id NOT IN (
        SELECT
            course_id
        FROM
            enrollments
        WHERE
            course_id IS NOT NULL
    );
```

---

# 12. Vì sao cách này hoạt động?

Subquery giờ chỉ trả về:

```text
(1, 2)
```

thay vì:

```text
(1, 2, NULL)
```

---

→ SQL có thể đánh giá logic bình thường.

---

# 13. Nhưng đây chưa phải giải pháp tốt nhất

---

## Tech Lead thường sẽ chọn

```sql
NOT EXISTS
```

---

# 14. Vì sao `NOT EXISTS` an toàn hơn?

`EXISTS` chỉ kiểm tra:

```text
"Có dòng tồn tại hay không?"
```

---

Nó:

- không quan tâm giá trị NULL
- không bị ảnh hưởng bởi 3-valued logic
- không dính bug NULL kinh điển của `NOT IN`

---

# 15. Version Production-Ready (Khuyên dùng)

```sql
SELECT
    c.*
FROM
    courses c
WHERE
    NOT EXISTS (
        SELECT
            1
        FROM
            enrollments e
        WHERE
            e.course_id = c.id
    );
```

---

# 16. Logic hoạt động của `NOT EXISTS`

---

## Với mỗi khóa học

Database kiểm tra:

```text
"Có enrollment nào trỏ tới course này không?"
```

---

### Nếu có

```text
EXISTS = TRUE
NOT EXISTS = FALSE
```

→ loại khóa học đó.

---

### Nếu không có

```text
EXISTS = FALSE
NOT EXISTS = TRUE
```

→ giữ lại khóa học.

---

# 17. Tại sao `NOT EXISTS` gần như luôn tốt hơn `NOT IN`?

| NOT IN                | NOT EXISTS           |
| --------------------- | -------------------- |
| Dễ chết vì NULL       | An toàn với NULL     |
| Logic phức tạp hơn    | Logic rõ ràng        |
| Dễ bug dữ liệu rác    | Robust hơn           |
| Có thể trả về UNKNOWN | Chỉ quan tâm tồn tại |

---

# 18. Tổng kết

## Sai lầm kinh điển

```sql
NOT IN (subquery)
```

khi subquery có thể chứa:

```text
NULL
```

---

# Ghi nhớ cực quan trọng

```text
NULL làm hỏng NOT IN
```

---

## Muốn vá nhanh

```sql
WHERE ... IS NOT NULL
```

trong subquery.

---

## Muốn an toàn production

Dùng:

```sql
NOT EXISTS
```

gần như luôn là lựa chọn tốt hơn.
