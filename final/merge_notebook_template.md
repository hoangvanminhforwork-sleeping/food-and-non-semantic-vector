# 0. Data and Configuration setup
## a. Download libraries
- Tự tham khảo nhé các libraries cần thiết cho colab nhé:
+) Code mới: .devcontainer
+) Code cũ: old_code/.devcontainer

## b. Verify data existence
- Vì dự án có dùng LLM để làm sạch data
- Nên là các ô trong này dùng để check xem đã upload data hết chưa
- Nếu ko thì quá trình ko thể tiếp tục, các nhà khoa học khác muốn reproduce lại thì mất tiền
- Các cell code trong ô này chỉ có mục đích là...check xem có file ko thôi
- Đổi tên tệp thành {folder}_output01/02/03 nhé...tức có file json ở folder content là dc
- Tức là tui sẽ đăng những tệp data này trên Github release luôn nhé, và sau đó ai khác muốn chạy lại code của tui, thì tải những cái tệp này xuống, và cho vào bên ngoài cùng của Google colab là dc

- Các tệp data cần check ở folder content trong Google colab:

+ Danh sách từ những ba lần chạy: old_code/03_ingredient_extraction/data/output1.jsonl => tức check xem có những tệp như ingredient_extraction_output1.json, output2.json, output3.json ko... (note, cần nói chuyện với AI về cách diễn đạt format đặt tên này)
+ old_code/04_geography_extraction/data/output1.jsonl...tự hiểu quy luật nha. Là geograpgy_extraction_output1.json /output2.json /output3.json á
+ old_code/05_scientific_extraction/data/output1.jsonl
+ old_code/06_reactions/data2/output1.jsonl

## c. Download data
- Tự đổi tên tương ứng như ở container trong local nha
- Đổi tên file zip đã giải nén thành tên như trong container này á
### Download Recipe data from zenodo

Đây là code đã chạy dc ở Colab tải dataset từ Zenodo về
```
import os
import requests
from tqdm import tqdm

RECORD_ID = "15169428"
FILE_NAME = "dataset.csv" 

# Tạo URL tải trực tiếp
download_url = f"https://zenodo.org/records/{RECORD_ID}/files/{FILE_NAME}?download=1"
# https://zenodo.org/records/15169428/files/dataset.csv?download=1
save_path = f"/content/{FILE_NAME}"
```
Mục tiêu là....tạo ra một file csv ở bên ngoài lề khi chạy Google colab

### Download FooDB
Link tải: https://foodb.ca/public/system/downloads/foodb_2020_4_7_csv.tar.gz
Nhớ thêm code để mở Zip ra nhé
Tui chưa có tải bằng code thử, toàn click tải thủ công...chắc cũng giống như download Zenodo ở kia
Thì tui dc một folder, sau khi giải nén thì nó chứa những file csv như thế này: data/foodb_2020_04_07
Tự ls -la mà đọc bố cục nhé...nói chung cứ hiểu là tạo một notebook lớn

### Download FoodUSDA
- Có hai phiên bản: latest và legacy
- Link download tương ứng:
+) latest: https://fdc.nal.usda.gov/fdc-datasets/FoodData_Central_foundation_food_csv_2026-04-30.zip...nó sẽ giống cái folder này: /workspace/data/FoodUSDA_latest
+) legacy: https://fdc.nal.usda.gov/fdc-datasets/FoodData_Central_sr_legacy_food_csv_2018-04.zip

# 1. Verify data để xuất ra gửi LLM + upload FooDB và FoodUSDA + LLM performance...hm....có hơi nhiều việc, cần nói chuyện với AI để phân bố cục cho gọn
- Tức ban đầu ấy, thì từ data có sẵn => xuất ra thành từng dòng như user prompt nè: old_code/03_ingredient_extraction/data/user_prompt.txt
- Có hai cột trong cái csv này dc trích ra để gửi LLM: cuisine và ingredient:
+ File csv gốc: data/RecipeZenodo.csv (cái này nó bẩn lắm)
+ Pipeline xử lý: tham khảo 0_upload_data_to_DuckDB và 1_silver_create
+ Nên nhớ là thẳng vào cuối cùng luôn. Chỉ cần một bảng duy nhất là Zenodo
+ tức là upload thẳng cái csv dạng final vào zenodo luôn. tức là gen ra code để có final rồi upload cái đó vào DuckDB luôn, cái file /workspace/data/RecipeZenodo_final.csv này nè. Ko cần tạo bảng trung gian làm gì cả, RAM Colab nhiều hơn máy tui, ko phải lo, ko cần artifact, t

- Sau đó là hai cột cuisine và ingredient ấy
- Tách từng cái unique value ở cấp độ element ra, tham khảo hai tệp này
+ ingredient: old_code/03_ingredient_extraction/ingredient_extraction.ipynb
+ cuisine: old_code/04_geography_extraction/geography_extraction.ipynb

+ tra xem ingredient và cuisine trong DuckDB của cái table zenodo final, lệch bao nhiêu so với output của LLM

- Sau đó upload các output của LLM
+ Viết công thức check self consistency và các công thức khác trong LLLM performance, mấy cái tệp output ở phần 0b á
+ mỗi table có tên là cái tên của folder luôn
+ Hình dạng mong muốn: input, output1, output2, output3, final_output
+ Tức từ ba cái json, phải viết SQL để có bảng này trong DuckDB
+ Rồi với mỗi mục thì liệt kê con số % mỗi tiêu chí luôn nhé, tham khảo 6_confirm_LLM_performance_number và investigate_data_mismatch

+ tham khảo: 3_add_AI_standardized_data

- Upload final FooDB và FoodUSDA:
+ Tạo schema fooDB, FoodUSDA_latest và FoodUSDA_legacy luôn
+ Upload chỉ những table cuối cùng, đẹp nhất vào DuckDB, những table ko liên quan có thể bỏ
+ Folder tham khảo: 0_upload_data_to_DuckDB

# 2. Query GBIF
- File tham khảo: old_code/10_geo_vector/scientific_name_country_query.ipynb và old_code/10_geo_vector/GBIF_queryable_rate.ipynb
- Tức là sau khi mà đã có final danh sách của những tên khoa học từ merge 3 cái file này lại: old_code/05_scientific_extraction/data/output{1, 2, 3}.jsonl thành file trong DuckDB
- Thì nguyên một phần chỉ để query GBIF có thành công bao nhiêu. Chỉ vậy thôi
- Chưa build vector ở giai đoạn này
- Nhưng mà nên có...gì đó...một chút liên quan tới country cho mỗi ingrdient chưa chuẩn hóa...kiểu như cái file này /workspace/old_code/10_geo_vector/data/dirty_ingredient_country_mapping.json

# 3. Tạo vector
## 3.1 Tạo vector chính của paper mình
- tạo ra ba vector geo, category, odor: tham khảo 7_create_npy_files
### 3.2 Tạo vector mà FoodBERT encode dc
- tham khảo old_code/FoodBERT_encoding_npy.ipynb

# 4. Tạo battle...tức phân rõ đánh nhãn hay chưa mà thôi
- Tham khảo:
+ Code mới: 8_create_battle
+ Code cũ: old_code/15_KMean_battle_field

# 5. Cho đánh nhau
- Tham khảo những file chỉ nhận old_code/15_KMean_battle_field làm cha, tức những file notebook đứng ngoài, ko phải file sâu hơn

# 6. Vẽ biểu đồ
- Tham khảo: old_code/17_result_and_discussion_/Redraw_all_figures.ipynb
- Theo đúng style này nhé