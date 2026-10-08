# 1. Giới thiẹu về đề tài nói chung
- Mình làm gì
+) Chắc gọi tên ba cái gạch đầu dòng thôi...đúng hơn là ba từ khóa: Food, culture, natural science
- Với danh nghĩa cá nhân...cảm ơn hai thầy xíu :))) Ko liên quan đến team --- có nên làm cái này ko?

# 2. Dẫn nhập
- Người châu Âu...lạnh...nền văn hóa bơ sữa phát triển? Maybe cần nguồn
- Nơi nào đó có khí hậu về nho => Rượu nho phát triển
- Theo food pairing theory, châu Á và châu Âu có sự khác biệt về kết hợp hương vị. Châu Á tương phản hơn, châu Âu thì theo hướng là hòa hợp
- Câu hỏi chung: Các bước trên là top-down approach
+ Nếu bottom-up approach thì sao. Tức từ những thứ ko liên quan tới văn hóa, chỉ có khoa học tự nhiên thì sao?
- Culture is not emerge from vaccum => Nhấn mạnh câu này

# 3. Methodology

## 3.1 Chèn hình tổng quát chung methodology là mình đã làm gì từ data ra figure của kết quả cuối cùng
- rồi các phần sau mới trình bày chi tiết

## 3.2 Làm sạch dữ liệu
- Dataset Zenodo
- Trước tiên là làm sạch dữ liệu với LLM: Cho cái bảng table 1, example of data processing vào đây
- Sau đó nói luôn performance của LLM

## 3.3 Các công thức tính từng loại vector cho từng món ăn
### 3.3.1 Địa lý
- Chèn công thức latex
- Giải thích intuition: Với món ăn này, mỗi quốc gia dc bao nhiêu điểm dựa vào phân bố đa dạng sinh học của chúng?
- Văn bản trong paper: To model ecological availability invariant to recipe length

### 3.3.2 Nguyên liệu
- Cách kết hợp nguyên liệu có thể tạo nên đặc trưng của vùng văn hóa, như trong paper nói y nguyên: The scale-invariant proportion pc(gk) yields a structural culinary footprint
- Chèn latex
- Giải thích intuition

### 3.3.3 Mùi
- Justification về tại sao phải có mục mùi:  Chemical Odor Profile Formulation: Thermal reactions transform nutrient precursors into volatile aroma compounds.
- Chèn công thức latex của mục mùi
- Sau đó justification về food pairing theory, cần đo sự đa dạng, cần Shannon entropy
- Chèn latex của Shannon entropy

# 4. Evaluation
## 4.1 Giới thiệu sơ qua về hai metrics đánh giá?

# ...kết
- System prompt, output của LLM dc công bố trên github release