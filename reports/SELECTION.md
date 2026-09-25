# Vì sao chọn lô này?

Trong 50 dòng đứng đầu [outputs/selection_round1.csv](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/selection_round1.csv), nếu chỉ có ngân sách rà năm ảnh, tôi sẽ ưu tiên chọn 5 frame theo bảng sau:

| Thứ tự | Tên Frame | Thời điểm | Điểm tổng hợp | U (Bất định) | A (Mơ hồ) | D (Đa dạng) | Box mơ hồ | Lý do lựa chọn |
| :---: | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **1** | `frame_0395.jpg` | 158.0s | **0.9150** | 0.8799 | 0.9167 | 1.0 | 11 box | Điểm tổng hợp cao nhất toàn pool, 11 box mơ hồ, đa dạng bối cảnh cuối video |
| **2** | `frame_0333.jpg` | 133.2s | **0.9045** | 0.9091 | 0.8333 | 1.0 | 10 box | Độ bất định U rất cao (0.9091), 27 box dự đoán, mật độ giao thông đông |
| **3** | `frame_0184.jpg` | 73.6s | **0.8937** | 0.8374 | 0.9167 | 1.0 | 11 box | Đoạn giữa video, 11 box mơ hồ ở khoảng cách cự ly trung bình |
| **4** | `frame_0384.jpg` | 153.6s | **0.8872** | 0.7744 | 1.0000 | 1.0 | 12 box | Chỉ số A đạt tối đa 1.0 (12 box mơ hồ cao nhất), cách frame 0395 hơn 4.4s |
| **5** | `frame_0167.jpg` | 66.8s | **0.8826** | 0.9153 | 0.7500 | 1.0 | 9 box | Độ bất định U rất cao (0.9153), tăng cường tính đa dạng thời gian D |

*Ghi chú về quyết định loại bỏ ảnh gần trùng*: Quyết định chọn Top 5 trên đã chủ động bỏ qua `frame_0185.jpg` (thứ tự 4 toàn pool, điểm 0.8930) và `frame_0332.jpg` (thứ tự 5, điểm 0.8913). Hai ảnh này vi phạm quy tắc khoảng cách thời gian tối thiểu (`MIN_GAP_S = 2.0s`), lần lượt chỉ cách `frame_0184.jpg` và `frame_0333.jpg` đúng 0.4 giây.

---

### Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV / ảnh contact sheet:
1. **frame_0395.jpg** (Rank 1 trong CSV, score = 0.9150): Trên ảnh contact sheet [outputs/selection_round1.jpg](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/selection_round1.jpg), frame này có nhiều vệt xe buýt lớn và ô tô ở làn giữa chuyển động nhanh, nơi mô hình phân vân giữa xe tải/buýt và quầng sáng lóa.
2. **frame_0333.jpg** (Rank 2 trong CSV, score = 0.9045, 27 box): Trên contact sheet hiển thị cụm xe đông đúc ở làn trung tâm, nhiều xe bị nhòe chuyển động dẫn đến độ bất định $U = 0.9091$.
3. **frame_0184.jpg** (Rank 3 trong CSV, score = 0.8937, 23 box): Thể hiện rõ các xe con và xe tải ở khoảng cách xa gần đường chân trời với 11 box nằm trong dải bất định ($0.15 \le conf < 0.5$).

---

### Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- **frame_0185.jpg** (Rank 4, Điểm rất cao: 0.8930, $U = 0.9360$): Frame này có điểm bất định U cao vượt trội (0.9360) và điểm tổng đứng thứ 4 toàn pool, nhưng **bị loại bỏ (selected = False)**. 
  - *Lý do*: Thời điểm của nó (74.0s) chỉ cách `frame_0184.jpg` (73.6s) đúng 0.4 giây. Vì camera đặt cố định trên cầu vượt, hai ảnh cách nhau 0.4s gần như hoàn toàn trùng lặp về vị trí các xe và bối cảnh. Nếu chọn cả hai sẽ lãng phí chi phí gán nhãn mà không mang lại tri thức mới cho mô hình (redundancy). Việc áp dụng ràng buộc `MIN_GAP_S = 2.0s` đã ngăn chặn hiện tượng lãng phí ngân sách này.

---

### Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Điểm bất định (Uncertainty score) và số box mơ hồ chỉ phản ánh trạng thái "do dự" của mô hình tại thời điểm hiện tại (các box có độ tin cậy quanh mức 0.5), **hoàn toàn không chứng minh hoặc bảo đảm rằng việc gán nhãn ảnh đó sẽ làm tăng AP50 trên tập kiểm thử**.
- Nếu một ảnh có điểm bất định cao do nhiễu quang học (vệt đèn pha quá chói, lóa camera, phản chiếu mặt đường phức tạp), việc đưa ảnh đó vào train có thể đưa thêm nhiễu vào mô hình thay vì cải thiện độ chính xác.
- Ngoài ra, phép chọn mẫu theo công thức kết hợp tuyến tính `score = W_U·U + W_A·A + W_D·D` chỉ là phương pháp phỏng đoán (heuristic), chưa tính đến độ bao phủ phân phối không gian đặc trưng (feature space coverage) thực tế của dữ liệu.
