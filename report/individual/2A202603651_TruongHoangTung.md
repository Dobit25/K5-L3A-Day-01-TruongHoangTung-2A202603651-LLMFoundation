# Báo cáo cá nhân — Day 01: Lọc tin nhắn rác bằng ML, DL, Embedding và LLM

- **MSSV:** 2A202603651
- **Họ và tên:** Trương Hoàng Tùng
- **SEED:** 2342

## 1. Bằng chứng

| Tệp | Có? |
| --- | --- |
| [`day01_lab.ipynb`](day01_lab.ipynb) — notebook đã chạy, còn output | ✓ |
| [`lab1_report.json`](lab1_report.json) — do ô báo cáo cuối sinh ra | ✓ |
| [`hand_calc.jpg`](hand_calc.jpg) — ảnh tính tay có các bước | ✓ |

Notebook có 15/15 bộ kiểm tra đạt, đủ 10 dự đoán trước thí nghiệm và không chứa API key.

## 2. Kết quả chính

| Cách tiếp cận | F1 test | F1 SUB 40 | p95 ms/tin | USD / 1k tin |
| --- | ---: | ---: | ---: | ---: |
| BoW + LogReg | 0.865 | 0.919 | 0.026 | 0 |
| BoW + MLP | 0.930 | 0.974 | 0.220 | 0 |
| MiniLM + kNN | 0.900 | 1.000 | 27.909 | 0 |
| MiniLM + LogReg | 0.951 | 1.000 | 11.371 | 0 |
| MiniLM + MLP head | 0.941 | 1.000 | 11.791 | 0 |
| LLM nhỏ zero-shot | 0.870 | 0.870 | 6590.981 | 0.019 |
| LLM lớn zero-shot | 0.895 | 0.895 | 6549.163 | 0.009 |
| LLM nhỏ few-shot | 0.950 | 0.950 | 2949.465 | 0.036 |

## 3. Dự đoán và thực tế

1. Em dự đoán model lớn đắt hơn model nhỏ. Kết quả: LLM lớn có F1 cao hơn (0.895 so với 0.870) nhưng chi phí ước tính thấp hơn (0.009 so với 0.019 USD/1k tin). Ước tính chi phí phụ thuộc số token prompt và output của lần chạy, không chỉ phụ thuộc giá niêm yết mỗi token.
2. Em dự đoán few-shot tăng nhẹ F1 và tăng chi phí 3–5 lần. Kết quả: F1 tăng từ 0.870 lên 0.950, còn chi phí tăng khoảng 1.9 lần. Bốn ví dụ làm prompt dài hơn nhưng không dài đến mức tăng 3–5 lần trong lần chạy này.
3. Em dự đoán MiniLM + MLP head nằm trên Pareto. Kết quả: MiniLM + LogReg nằm trên Pareto vì cùng F1 SUB = 1.000 nhưng p95 thấp hơn MLP head (11.371 ms so với 11.791 ms).

## 4. Phân tích 50 case khó

BoW + LogReg tụt nhiều nhất: F1 từ 0.919 trên SUB xuống 0.286 trên 50 case khó. Nhóm spam giọng tự nhiên là khó nhất với BoW (0%) vì các token quen thuộc của spam gần như không xuất hiện. Với LLM zero-shot, nhóm ham trông giống spam chỉ đúng 40%; prompt phòng thủ giúp tăng accuracy tổng từ 78% lên 84% nhưng làm nhóm nói về spam giảm từ 75% xuống 50%.

Ví dụ, case #15 là spam giọng tự nhiên nhưng LLM phòng thủ đoán sai vì nội dung giống một lời giới thiệu dịch vụ bình thường. Case #47 là tin ham kể lại một tin lừa đảo; LLM đoán sai vì các từ như “won”, “£1000” và số điện thoại giống tín hiệu spam. Các case này cho thấy điểm cao trên test cùng phân phối không đảm bảo độ bền trước ngữ cảnh mới.

## 5. Thử thách cá nhân

Đề #3: đổi hidden layer của MLP thành 4 và 512 nơ-ron. Với hidden = 4, mạng có 8,009 tham số, F1 test = 0.924 và train mất 12.305 giây. Với hidden = 512, mạng có 1,025,025 tham số, F1 test = 0.920 và train mất 21.326 giây.

Mạng nhỏ phù hợp khi cần ít bộ nhớ, huấn luyện nhanh và giảm nguy cơ overfitting. Trong lần chạy này, mạng 512 nơ-ron nhiều tham số hơn rất nhiều nhưng không cải thiện F1, nên hidden = 4 hợp lý hơn.

## 6. Phương án cho sản phẩm thật

Tất cả model local đạt p95 < 50 ms; MiniLM + LogReg đạt F1 test cao nhất (0.951), p95 = 11.371 ms và throughput khoảng 1,190 tin/s, nên phù hợp nhất cho luồng lọc chính. kNN đạt SLA nhưng chỉ khoảng 180 tin/s, ít dư địa hơn nhu cầu trung bình 116 tin/s. Các LLM mất 2.9–6.6 giây mỗi tin nên không đạt SLA. Nếu dùng LLM nhỏ zero-shot cho 10 triệu SMS/ngày trong 30 ngày, chi phí ước tính là 0.019 × 10,000 × 30 = **5,700 USD/tháng**.

Router đề xuất: MiniLM + LogReg xử lý ngay các tin có xác suất rất thấp hoặc rất cao; tin có xác suất lưỡng lự hoặc bất đồng với BoW + MLP được kiểm tra bằng MLP head/rule cục bộ. LLM chỉ chạy bất đồng bộ để phân tích mẫu mới và tạo dữ liệu gán nhãn. Thiết kế này giữ SLA cho toàn bộ luồng chính nhưng vẫn tận dụng được nhiều model.

Điều em dự đoán sai nhiều nhất là cho rằng F1 cao trên SUB đồng nghĩa model bền vững. MiniLM + LogReg có F1 SUB = 1.000 nhưng chỉ còn F1 = 0.542 trên bộ case khó, nên cần stress test ngoài tập test thông thường.

## 7. AI log và điều đã học

| Phần | Bạn hỏi AI gì | AI trả lời/viết gì | Bạn đã sửa / kiểm chứng gì |
| --- | --- | --- | --- |
| Phần 4 | Cách sửa `call_openai`, đổi model và chạy mock test | Gợi ý tạo client trong hàm, truyền `provider_kwargs` và dùng mock | Chạy check 4.1 + 4.4, đạt ✅ |
| Phần 4–5 | Cách đọc kết quả LLM, Pareto và stress test | Gợi ý cách tính F1, p95, chi phí và phân tích nhóm lỗi | Đối chiếu với output notebook và sửa số liệu theo lần chạy cuối |

Em hiểu rằng F1 trên tập test thông thường chỉ phản ánh dữ liệu gần với train; cần kiểm tra thêm dữ liệu ngoài phân phối, độ trễ và chi phí trước khi chọn model cho sản phẩm thật. Em cũng hiểu rằng LLM có thể linh hoạt với bài toán mới nhưng không phù hợp làm lớp quyết định trực tuyến khi SLA rất thấp.
