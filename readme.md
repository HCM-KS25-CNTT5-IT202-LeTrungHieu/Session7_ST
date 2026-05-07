# [Sáng tạo] Báo cáo "Độ Lệch Học Phí"

---

# 1. Phân tích bài toán

## Yêu cầu của Giám đốc

Hệ thống cần xuất:

| title          | price  | price_difference |
| -------------- | ------ | ---------------- |
| SQL Basic      | 500000 | +100000          |
| Docker Mastery | 300000 | -100000          |

---

Trong đó:

```text
price_difference
=
Giá khóa học
-
Giá trung bình toàn hệ thống
```

---

# 2. Nỗi đau kỹ thuật

---

## Điều Dev thường làm sai

Nhiều người sẽ viết:

```sql
SELECT
    title,
    price,
    AVG(price)
FROM
    courses;
```

---

## Nhưng hệ thống sẽ báo lỗi

Hoặc nếu thêm:

```sql
GROUP BY title, price
```

thì:

```text
AVG(price)
```

lại chỉ tính theo từng nhóm nhỏ.

---

# 3. Vấn đề cốt lõi

Bài toán này yêu cầu đồng thời:

---

## Xem dữ liệu chi tiết

```text
Tên từng khóa học
Giá từng khóa học
```

---

## Và dữ liệu tổng quan

```text
Giá trung bình của TOÀN BỘ hệ thống
```

---

# 4. Vì sao `GROUP BY` không phù hợp?

---

## `GROUP BY` mang bản chất

```text
Gộp nhiều dòng thành ít dòng hơn
```

---

Ví dụ:

```sql
GROUP BY category_id
```

→ tất cả khóa học cùng category bị gom lại.

---

Nhưng ở bài này:

```text
Giám đốc vẫn muốn nhìn từng khóa học riêng lẻ
```

→ không được phép làm mất row detail.

---

# 5. Giải pháp kiến trúc: Scalar Subquery trong SELECT

---

## Ý tưởng cực mạnh

Ta đặt:

```sql
(
    SELECT AVG(price)
    FROM courses
)
```

ngay bên trong:

```sql
SELECT
```

---

# 6. Scalar Subquery là gì?

---

## Khái niệm

Scalar Subquery là:

```text
Subquery trả về đúng 1 giá trị duy nhất
```

Ví dụ:

```sql
SELECT AVG(price) FROM courses
```

trả về:

```text
400000
```

---

→ Vì nó chỉ trả về:

- 1 dòng
- 1 cột

nên SQL có thể dùng nó như:

```text
một con số bình thường
```

---

# 7. Đây là điều kỳ diệu của kỹ thuật này

---

## Query ngoài vẫn giữ nguyên từng dòng

```sql
FROM courses
```

→ vẫn quét từng khóa học riêng biệt.

---

## Nhưng Scalar Subquery

```sql
(SELECT AVG(price) FROM courses)
```

sẽ cung cấp:

```text
Giá trị trung bình toàn hệ thống
```

cho từng dòng.

---

# 8. Tư duy kiến trúc phía sau

---

## Query ngoài

chịu trách nhiệm:

```text
Detail-level data
```

---

## Scalar Subquery

chịu trách nhiệm:

```text
Global-level data
```

---

→ Đây chính là kỹ thuật:

```text
"Xem chi tiết + xem tổng quan cùng lúc"
```

---

# 9. Câu lệnh SQL hoàn chỉnh

```sql
SELECT
    title,
    price,
    price - (
        SELECT
            AVG(price)
        FROM
            courses
    ) AS price_difference
FROM
    courses;
```

---

# 10. Logic hoạt động của query

---

## Bước 1

Subquery chạy:

```sql
SELECT AVG(price)
FROM courses
```

Ví dụ kết quả:

```text
400000
```

---

## Bước 2

Query ngoài scan từng khóa học:

| title     | price  |
| --------- | ------ |
| SQL Basic | 500000 |
| Docker    | 300000 |

---

## Bước 3

Database tính:

```text
500000 - 400000 = 100000
300000 - 400000 = -100000
```

---

# 11. Kết quả cuối cùng

| title     | price  | price_difference |
| --------- | ------ | ---------------- |
| SQL Basic | 500000 | 100000           |
| Docker    | 300000 | -100000          |

---

# 12. Ưu điểm cực lớn của Scalar Subquery

| Kỹ thuật        | Kết quả                       |
| --------------- | ----------------------------- |
| GROUP BY        | Mất chi tiết từng dòng        |
| Scalar Subquery | Giữ detail + có global metric |

---

# 13. Đây là pattern cực phổ biến trong BI / Dashboard

Dùng để tính:

- chênh lệch so với trung bình
- % so với tổng
- ranking
- deviation
- benchmark
- KPI comparison

---

# 14. Một tối ưu thực chiến

Trong hệ thống lớn:

```sql
(SELECT AVG(price) FROM courses)
```

có thể được optimizer cache.

---

Nhưng trên database cực lớn:

Tech Lead thường sẽ chuyển sang:

- CTE
- Window Function
- Materialized View

để tối ưu hơn.

---

# 15. Tổng kết

## Scalar Subquery trong SELECT cho phép

```text
Đưa dữ liệu tổng quan vào từng dòng chi tiết
```

---

# Ghi nhớ nhanh

```text
GROUP BY
→ gom dòng
```

```text
Scalar Subquery
→ giữ nguyên dòng
```

---

## Đây là một kỹ thuật cực mạnh trong SQL Analytics

Vì nó giúp:

```text
"Row-level data"
+
"System-level insight"
```

xuất hiện cùng lúc trong một báo cáo.
