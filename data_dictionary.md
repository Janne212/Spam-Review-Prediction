# DS102 – Phát hiện spam trong đánh giá sản phẩm

Bài toán: nhận diện bình luận rác (spam). Đầu vào là một câu review, đầu ra là một trong 4 nhãn (0: no-spam, 1: fake review, 2: review on brand only, 3: non-review).

Kaggle: https://www.kaggle.com/competitions/ds-102-r-11-cnvn/overview

## Tổng quan các file

| File                           | Số dòng | Mô tả                                                             |
| ------------------------------ | --------- | ------------------------------------------------------------------- |
| `ds102_train.csv`            | 14.306    | Tập huấn luyện có nhãn (`spam_label`)                        |
| `ds102_dev.csv`              | 1.590     | Tập có nhãn (`spam_label`), dùng để kiểm định            |
| `ds102_test.csv`             | 3.974     | Tập không có nhãn, cần dự đoán                              |
| `ds102_samplesubmission.csv` | 3.974     | Mẫu file nộp bài, gồm `ID` và `spam_label` (mặc định 0) |

Không cột nào bị thiếu giá trị ở train, dev và test. Train và dev có cùng cấu trúc cột; test thêm cột `id` và không có `spam_label`.

## Mô tả

Miền giá trị tính trên cả 3 file.

| Thuộc tính     | Kiểu dữ liệu                         | Có trong        | Miền giá trị                                                                                                                                                 | Ý nghĩa                                                                                                                           |
| ---------------- | --------------------------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `id`           | `str`                                 | test             | `ds102_0` … `ds102_3973`, duy nhất                                                                                                                        | Mã định danh từng dòng test. Khớp với cột `ID` của file nộp bài.                                                       |
| `rating`       | `int64`                               | train, dev, test | Số nguyên 1–5, rất lệch về 5 sao                                                                                                                          | Số sao người mua chấm trong đánh giá.                                                                                        |
| `comment`      | `str`                                 | train, dev, test | Văn bản tiếng Việt, dài 1–1.089 ký tự. Có emoji, xuống dòng `\r\n`, chuỗi lặp                                                                    | Nội dung đánh giá của người mua. Đây là đầu vào chính để phân loại spam.                                          |
| `categories`   | `str` (chuỗi biểu diễn danh sách) | train, dev, test | Danh sách 2–5 cấp, ví dụ `['Làm Đẹp - Sức Khỏe', 'Trang điểm ', 'Dụng cụ trang điểm', 'Nhíp, dao cạo']`. Train có 698 giá trị khác nhau | Cây danh mục chi tiết của sản phẩm, từ rộng đến hẹp. Cần parse bằng `ast.literal_eval`.                              |
| `category`     | `str`                                 | train, dev, test | 12 giá trị (xem danh sách bên dưới)                                                                                                                       | Nhóm ngành hàng đã gộp lại từ `categories`.                                                                               |
| `product_name` | `str`                                 | train, dev, test | Văn bản dài 14–232 ký tự. Train có 1.673 sản phẩm khác nhau trên 14.306 dòng                                                                        | Tên sản phẩm được đánh giá. Một sản phẩm có nhiều đánh giá.                                                        |
| `description`  | `str`                                 | train, dev, test | Văn bản dài 14–17.084 ký tự. Train có 1.638 mô tả khác nhau                                                                                           | Mô tả sản phẩm do người bán viết.                                                                                           |
| `num_sold`     | `int64`                               | train, dev, test | Số nguyên 0–2.600.000, trung vị khoảng 5.000, lệch nặng (nên lấy log)                                                                                  | Tổng số lượng đã bán của sản phẩm.                                                                                        |
| `num_review`   | `int64`                               | train, dev, test | Số nguyên 0–377.500, trung vị khoảng 1.000, lệch nặng (nên lấy log)                                                                                    | Tổng số đánh giá của sản phẩm.                                                                                              |
| `spam_label`   | `int64`                               | train, dev       | {0, 1, 2, 3} (xem phân phối bên dưới)                                                                                                                      | Nhãn loại bình luận cần dự đoán: 0 no-spam, 1 fake review, 2 review on brand only, 3 non-review. Test không có cột này. |

## Ví dụ
 
Mỗi dòng dữ liệu là một đánh giá. Chuỗi `description` dưới đây đã được cắt ngắn cho gọn.
 
**Một dòng ở train/dev** (có `spam_label`, không có `id`), lấy từ `ds102_dev.csv`:
 
```jsonc
{
  "rating": 5,                            
  "comment": "Dùng tốt, cắt lông mũi phù hợp",  
  "categories": "['Làm Đẹp - Sức Khỏe', 'Trang điểm ', 'Dụng cụ trang điểm', 'Nhíp, dao cạo']", 
  "category": "Sắc Đẹp",                        
  "product_name": "Kéo Cắt Tỉa Lông Mũi Đầu Tròn Kềm Nghĩa KM 604- Thép Không Gỉ Cao Cấp - Thiết Kế Nhỏ - Gọn - Nhẹ - An toàn",
  "description": "Sản phẩm Kéo cá nhân KM-604 thuộc dòng sản phẩm chăm sóc cá nhân của Kềm Nghĩa, thiết kế nhỏ- g…", // đã cắt ngắn
  "num_sold": 54,                               
  "num_review": 11,                            
  "spam_label": 0               
}
```
 
**Một dòng ở test** (có `id`, không có `spam_label`), lấy từ `ds102_test.csv`:
 
```jsonc
{
  "rating": 5,
  "comment": "Shop giao hàng khá nhanh. Chất liệu cũng được dày dặn che sáng tốt, gia công hơi ẩu vạt dài vạt ngắn làm mình phải mất công dùng kéo chỉnh lại thành ra lại ngắn mất đi. Thks shop",
  "categories": "['Nhà Cửa & Đời Sống', 'Trang trí nhà cửa', 'Rèm & Màn sáo']",
  "category": "Nhà Cửa - Đời Sống",
  "product_name": "Rèm cửa dán keo không cần thanh treo (nhiều mẫu và size)",
  "description": "RÈM DÁN KEO TRỰC TIẾP TRÊN THÀNH CỬA ĐANG ĐƯỢC LỰA CHỌN NHIỀU HIỆN NAY, TIẾT KIỆM CHI PHÍ THANH TREO, KHÔNG MẮC CÔNG LẮP…", // đã cắt ngắn
  "num_sold": 40000,
  "num_review": 11400,
  "id": "ds102_0"                   
}
```

## Giá trị của `category` (12 nhóm)

Thời Trang, Điện Tử, Điện Thoai - Máy Tính (nguyên văn, thiếu dấu), Khác, Nhà Cửa - Đời Sống, Sắc Đẹp, Đồ Chơi, Thể Thao, Máy Ảnh, Phương Tiện, Giày Dép, Sách.

## Phân phối nhãn `spam_label`

| Nhãn | Train          | Dev           | Ý nghĩa                                                 |
| ----- | -------------- | ------------- | --------------------------------------------------------- |
| 0     | 10.512 (73,5%) | 1.183 (74,4%) | No-spam: bình luận bình thường                       |
| 1     | 211 (1,5%)     | 17 (1,1%)     | Fake review: đánh giá giả                             |
| 2     | 1.103 (7,7%)   | 120 (7,5%)    | Review on brand only: chỉ nhận xét về thương hiệu  |
| 3     | 2.480 (17,3%)  | 270 (17,0%)   | Non-review: nội dung không phải đánh giá sản phẩm |

## Phân phối `rating`

| Sao | Train          | Dev           | Test          |
| --- | -------------- | ------------- | ------------- |
| 1   | 251 (1,8%)     | 31 (1,9%)     | 71 (1,8%)     |
| 2   | 110 (0,8%)     | 13 (0,8%)     | 20 (0,5%)     |
| 3   | 217 (1,5%)     | 30 (1,9%)     | 59 (1,5%)     |
| 4   | 387 (2,7%)     | 43 (2,7%)     | 95 (2,4%)     |
| 5   | 13.341 (93,3%) | 1.473 (92,6%) | 3.729 (93,8%) |

## Lưu ý

- **Định nghĩa nhãn:** theo đề bài đồ án. Nhãn 1, 2, 3 đều là spam; chỉ có tên loại, chưa có mô tả chi tiết hay tiêu chí gán nhãn.
- **Cột lặp theo sản phẩm:** `categories`, `category`, `product_name`, `description`, `num_sold`, `num_review` là thuộc tính sản phẩm, nên lặp lại ở mọi đánh giá của cùng sản phẩm. Chỉ `rating` và `comment` là thông tin riêng của từng đánh giá.
- **Tên cột khóa:** file test dùng `id` (chữ thường), file nộp bài dùng `ID`.
- **Mất cân bằng:** nhãn 0 chiếm khoảng 74%, nhãn 1 chỉ khoảng 1–1,5% (211 mẫu ở train, 17 ở dev). Tỉ lệ nhãn của train và dev gần như giống nhau.
- **`num_review` có thể lớn hơn `num_sold`:** xảy ra ở 1.573 dòng train (11%), 188 dòng dev và 414 dòng test. Hai cột này không có quan hệ bao hàm, nên đừng dùng `num_review/num_sold` như một tỉ lệ ≤ 1.
- **`category` không phải phần tử đầu của `categories`:** mỗi nhóm `category` ứng với 2–8 giá trị đầu khác nhau của `categories` (ví dụ `Thời Trang` có 8). `category` là nhóm đã gộp lại, không suy ra trực tiếp được.
- **Trùng lặp:** train có 1 dòng trùng toàn bộ các cột; dev và test không có. Không có comment rỗng.
- **Tính nhất quán:** kiểu cột giống nhau ở cả 3 file (`int64`: `rating`, `num_sold`, `num_review`, `spam_label`; `str`: các cột còn lại), `rating` chỉ nhận 1–5, không có giá trị âm, không có `category` lạ ở dev và test so với train, `categories` parse được 100%.
