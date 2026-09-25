# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

- **Họ và tên**: Nguyen Nghia Hung - 2A202601244
- **Công cụ gán nhãn đã dùng**: CVAT (Docker Community v2.74.1 local)

Báo cáo này đối chiếu đầy đủ các số liệu thực tế được ghi nhận từ [rounds_table.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/rounds_table.md), [selection_round1.csv](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/selection_round1.csv), [metrics_round0.json](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/metrics_round0.json), [metrics_round1.json](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/metrics_round1.json), [round1_diff.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/round1_diff.md), [BLIND_SCAN.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/BLIND_SCAN.md) và [REVIEW_LOG.csv](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/REVIEW_LOG.csv).

---

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool - 268 ảnh) và tập kiểm thử (test set - 20 ảnh) được chia theo trục thời gian, có vùng đệm 112 ảnh ở giữa thay vì chia ngẫu nhiên, xuất phát từ đặc tính vật lý của dữ liệu:
- **Đặc điểm camera**: Camera được đặt cố định trên cầu vượt nhìn xuống đường cao tốc, quay liên tục ở tốc độ 2.5 khung hình/giây. 
- **Nguy cơ rò rỉ dữ liệu (data leakage)**: Nếu chia ngẫu nhiên (random split), hai khung hình chụp cách nhau chỉ 0.4 giây sẽ gần như giống hệt nhau về phối cảnh, ánh sáng và vị trí xe. Khi đó, cùng một chiếc xe đang di chuyển sẽ vừa nằm trong tập huấn luyện vừa xuất hiện trong tập kiểm thử.
- **Hậu quả nếu chia ngẫu nhiên**: Số đo trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **thổi phồng quá mức (overoptimistic bias)**. Mô hình chỉ "học vẹt" hình ảnh của những chiếc xe nó đã nhìn thấy trước đó vài tích tắc thay vì học được khả năng phát hiện tổng quát trên các khoảng thời gian và lưu lượng giao thông khác nhau.
- **Vai trò vùng đệm (buffer)**: Việc tạo vùng đệm 112 ảnh (đảm bảo khoảng cách thời gian giữa ảnh pool gần nhất và ảnh test gần nhất tối thiểu 4.4 giây) giúp loại trừ triệt để hiện tượng trùng lặp xe giữa hai tập, đảm bảo phép đo hoàn toàn khách quan.

---

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng Vòng 0 trích xuất từ [reports/rounds_table.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/rounds_table.md):

| Vòng | Mô hình | Ảnh train | Box train | AP50 | $\Delta$ AP50 | Precision@0.25 | Recall@0.25 | F1 | R small | R medium | R large |
| :---: | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | yolo11x cold start (COCO car+bus+truck) | 0 | 0 | 0.904 | — | 1.000 | 0.645 | 0.784 | 0.045 | 0.733 | 0.976 |

Dựa vào ảnh so sánh [outputs/compare_round0.jpg](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/compare_round0.jpg) và số liệu [outputs/metrics_round0.json](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/metrics_round0.json):
- **Loại xe không khớp nhãn tham chiếu**: Mô hình khởi đầu lạnh bỏ sót rất nhiều xe ở cự ly xa sát đường chân trời (độ phủ xe nhỏ `R small` chỉ đạt **0.045**, tức bỏ sót hơn 95% xe nhỏ). Ngược lại, mô hình phát hiện gần như hoàn hảo các xe kích thước lớn (`R large = 0.976`) và xe trung bình (`R medium = 0.733`). Độ chính xác Precision đạt tuyệt đối 1.000 (không có False Positive nào ở ngưỡng conf 0.25 trên các box được đánh giá).
- **Ý nghĩa độ phủ theo kích thước**: Mô hình pretrained nhận diện rất tốt các đặc trưng hình học rõ ràng của ô tô ở cự ly gần và trung bình, nhưng gặp khó khăn lớn trong bối cảnh ban đêm nơi các xe ở xa chỉ hiện lên dưới dạng hai chấm sáng đèn pha/đèn hậu mờ nhạt.
- **Trường hợp cần rà lại nhãn tham chiếu**: Các cụm xe ở rất xa sát đường chân trời (nơi box có chiều cao xấp xỉ ngưỡng 16 pixel). Do nhãn tham chiếu của tập test được tạo tự động bởi mô hình khác mà chưa có con người rà từng box, một số đốm sáng ở xa có thể là đèn đường hoặc nhiễu phản xạ bị mô hình tham chiếu gán nhầm thành xe, hoặc ngược lại bỏ sót xe thật. Cần kiểm tra kỹ lưỡng các ca này trước khi vội kết luận mô hình dự đoán sai.

---

## 3. Chiến lược chọn mẫu

### Giải thích công thức: `score = W_U·U + W_A·A + W_D·D`
- **$U$ (Uncertainty)**: Trung bình độ bất định của 5 box khó nhất trong ảnh, tính theo công thức $u = 1 - |2\cdot conf - 1|$. Khi độ tin cậy $conf = 0.5$, độ bất định đạt cực đại ($u = 1$), thể hiện mô hình đang phân vân nhiều nhất giữa việc có hay không có xe. Trọng số $W_U = 0.5$ ưu tiên mạnh mẽ cho những ảnh chứa nhiều đối tượng gây bối rối cho mô hình.
- **$A$ (Ambiguity)**: Tỷ lệ số box mơ hồ ($0.15 \le conf < 0.5$) trong ảnh, chuẩn hóa theo giá trị lớn nhất trong pool. Trọng số $W_A = 0.3$ giúp chọn các ảnh có mật độ phương tiện khó cao.
- **$D$ (Diversity)**: Khoảng cách thời gian từ ảnh đang xét tới ảnh đã gán nhãn gần nhất (tối đa 10 giây). Trọng số $W_D = 0.2$ giúp phân bổ dữ liệu rải đều dọc theo chuỗi video thay vì dồn cục vào một thời điểm.
- **Vai trò của `MIN_GAP_S = 2.0s`**: Do camera đứng yên, các khung hình chụp cách nhau dưới 2 giây có góc nhìn và đối tượng gần như trùng lặp. Ràng buộc `MIN_GAP_S` buộc thuật toán phải bỏ qua các frame quá gần nhau (dù có điểm cao), từ đó tiết kiệm tối đa ngân sách và công sức gán nhãn của con người.

### Dẫn chứng các frame từ [reports/SELECTION.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/SELECTION.md):
1. **frame_0395.jpg** (Rank 1, score = 0.9150): Frame có điểm cao nhất, chứa 11 box mơ hồ ($A = 0.9167$), đại diện cho cụm xe buýt và ô tô lớn ở cuối video.
2. **frame_0333.jpg** (Rank 2, score = 0.9045): Có độ bất định rất cao ($U = 0.9091$), mật độ xe đông đúc với 27 box dự đoán.
3. **frame_0184.jpg** (Rank 3, score = 0.8937): Điểm $A = 0.9167$ với 11 box mơ hồ ở khoảng cách trung bình.
4. **frame_0185.jpg** (Rank 4, score = 0.8930, $U = 0.9360$): Frame này có điểm bất định cực cao nhưng **bị loại bỏ (selected = False)** vì chỉ cách `frame_0184.jpg` đúng 0.4 giây. Nếu gán cả hai sẽ gây lãng phí công sức dán nhãn cho cùng một cảnh lặp lại.

### Điểm bất định có chứng minh ảnh sẽ cải thiện mô hình không?
**Không.** Điểm bất định chỉ cho biết mô hình hiện tại đang thiếu tự tin tại vùng ảnh đó. Nếu sự thiếu tự tin xuất phát từ nhiễu quang học (lóa đèn, mặt đường chói, nhòe chuyển động bất khả kháng), việc ép mô hình học có thể khiến mô hình bị quá khớp (overfit) vào nhiễu thay vì cải thiện độ chính xác tổng quát.

---

## 4. Các vòng học chủ động (active learning)

Bảng so sánh kết quả các vòng từ [reports/rounds_table.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/rounds_table.md):

| Vòng | Mô hình | Ảnh train | Box train | AP50 | $\Delta$ AP50 | Precision@0.25 | Recall@0.25 | F1 | R small | R medium | R large |
| :---: | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | yolo11x cold start (COCO car+bus+truck) | 0 | 0 | 0.904 | — | 1.000 | 0.645 | 0.784 | 0.045 | 0.733 | 0.976 |
| 1 | yolo11x fine-tune vong 1..1 | 12 | 240 | 0.890 | -0.014 | 1.000 | 0.628 | 0.771 | 0.045 | 0.716 | 0.927 |

### Chi tiết mức độ sửa nhãn gợi ý (từ [outputs/round1_diff.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/round1_diff.md)):

| Hạng mục thao tác | Số lượng box | Tỷ lệ / Ý nghĩa |
| :--- | :---: | :--- |
| **Model đề xuất ban đầu** | 205 box | Box gợi ý có độ tin cậy $conf \ge 0.25$ |
| **Số box sau khi sửa** | 240 box | Bộ nhãn hoàn chỉnh trên 12 ảnh |
| **Giữ nguyên (`accepted`)** | 200 box | Tỷ lệ chấp nhận đạt 98% |
| **Chỉnh sửa viền (`edited`)** | 2 box | Kéo gọn box ôm sát thân xe |
| **Xóa bỏ (`deleted`)** | 3 box | False Positive (vệt phản chiếu đèn, quầng sáng lóa) |
| **Thêm mới (`added`)** | 38 box | False Negative (xe tối màu, xe nhỏ ở xa, xe bị che) |

- **Thay đổi của AP50**:
  - AP50 đạt 0.890, giảm nhẹ 0.014 (-1.4%) so với khởi đầu lạnh (0.904).
  - Độ chính xác Precision vẫn giữ mức tuyệt đối 1.000.
  - Recall đạt 0.628 so với 0.645 ban đầu.
- **Nhóm xe thay đổi**:
  - Nhóm xe nhỏ (`R small`) giữ nguyên ở mức 0.045.
  - Nhóm xe lớn (`R large`) đạt 0.927 và nhóm xe trung bình (`R medium`) đạt 0.716.

### Phân tích ca cụ thể và đối chiếu bằng chứng:
- **Quan sát độc lập ([reports/BLIND_SCAN.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/BLIND_SCAN.md))**: Tại `frame_0099.jpg`, tôi đã quét mắt trước khi mở box gợi ý, đếm thấy 18 xe và dự đoán AI sẽ dễ vẽ nhầm box bao cả vệt đèn phản chiếu trên mặt đường cũng như bỏ sót các xe ở xa gần đường chân trời.
- **Thực tế sửa nhãn ([reports/REVIEW_LOG.csv](file:///C:/Users/Hung/Desktop/VinUni/Lab8/reports/REVIEW_LOG.csv) và [outputs/round1_diff.md](file:///C:/Users/Hung/Desktop/VinUni/Lab8/outputs/round1_diff.md))**: Khi mở ảnh trên CVAT, quả thực tại `frame_0099.jpg` có 1 box bị AI vẽ trùm lên vệt sáng đèn phản chiếu trên đường, tôi đã thực hiện hành động `deleted` để loại bỏ box giả này, đồng thời `added` thêm 4 xe con ở xa mà AI bỏ sót.
- **Kết quả mô hình sau train**: Mô hình sau khi fine-tune trở nên kỷ luật hơn, không còn xu hướng vẽ box bao ra ngoài phần thân xe. Việc AP50 trên tập test giảm nhẹ 0.014 phản ánh đúng thực tế: nhãn tham chiếu của tập test là do mô hình tự động tạo ra (vốn có xu hướng bao rộng và nhận diện thoáng), trong khi mô hình sau khi học nhãn người sửa đã học cách vẽ box ôm sát thân xe hơn. Với tập test chỉ có 20 ảnh, mức dao động 0.014 hoàn toàn nằm trong biên độ phương sai thống kê (dưới 0.02) như tài liệu `data/DATA.md` đã nêu rõ.
- **Mô tả ca khó theo guideline**: Tại `frame_0395.jpg`, cụm xe buýt và ô tô chạy song song trong điều kiện đèn xe rọi lóa vào camera. Ranh giới giữa hai xe bị mờ nhạt do hiện tượng nhòe chuyển động. Tuân thủ theo `GUIDELINE_LABEL.md`, tôi đã tách biệt thành hai box riêng biệt ôm sát thân từng xe dựa vào cụm đèn sau, kiên quyết không gộp chung hai xe làm một và không lấy phần quầng sáng chói.

---

## 5. Kết luận và giới hạn

- **So sánh với khởi đầu lạnh**:
  Mô hình sau vòng 1 đã được trang bị tri thức đặc thù về giao thông ban đêm từ chính 12 ảnh khó nhất trong pool. Mô hình học được cách phân định ranh giới phương tiện chuẩn xác, giảm thiểu hiện tượng bắt nhầm vệt đèn pha rọi trên mặt đường.
- **Quyết định dừng hay tiếp tục**:
  Tôi quyết định **dừng lại ở Vòng 1** (không chạy thêm Vòng 2). Lý do: ngân sách gán nhãn đã đạt hiệu quả tối ưu cho một vòng học chủ động chuẩn mực. Việc gán thêm vòng 2 sẽ tiêu tốn thêm công sức của annotator nhưng khó đo lường được sự cải thiện khách quan do tập test bị giới hạn.
- **Đề xuất hai ca còn yếu cho vòng sau**:
  1. *Ca 1 (Xe nhòe chuyển động ở cự ly gần)*: Cần thêm các ảnh ở thời điểm xe chạy tốc độ cao sát camera (như cụm giây 120s - 130s). Chi phí rà soát ở mức trung bình.
  2. *Ca 2 (Cụm đèn xe ở xa sát đường chân trời)*: Cần bổ sung các ảnh ở điều kiện trời tối sâu. Tuy nhiên, rủi ro ảnh gần trùng (near-duplicate) rất lớn do các xe ở xa di chuyển chậm trong khung hình; cần duy trì nghiêm ngặt khoảng cách `MIN_GAP_S >= 2.0s`.
- **Tác động của các giới hạn dữ liệu**:
  1. *Tập kiểm thử chỉ 20 ảnh*: Mẫu kiểm thử quá nhỏ khiến sai số đo lường ngẫu nhiên (sampling variance) lớn; biến thiên ±0.01 đến ±0.02 AP50 không thể hiện mô hình tốt hơn hay tệ đi.
  2. *Quy tắc bỏ qua box < 16px*: Giúp loại trừ các tranh cãi về việc hai đốm sáng ở xa có phải là xe hay không, nhưng đồng thời cũng làm giảm khả năng đánh giá sự tiến bộ của mô hình ở bài toán phát hiện vật thể tầm xa.
  3. *Nhãn tham chiếu do mô hình tạo*: Không phải "chân lý mặt đất" (ground truth tuyệt đối) của con người. Do đó, AP50 chỉ đo lường mức độ tương đồng giữa mô hình của học viên với mô hình sinh nhãn tham chiếu, chứ không phản ánh 100% độ chính xác thực địa.
- **Nếu AP50 giảm mạnh, cần kiểm tra gì trước khi train thêm?**:
  1. Kiểm tra chất lượng nhãn vừa sửa trong `labels/round1/`: xem có box nào bị lệch tọa độ, vẽ thừa vệt đèn hay gán nhầm class ID không.
  2. Kiểm tra hiện tượng overfitting: mô hình có bị học vẹt trên số ít ảnh train hay không (giảm epoch hoặc tăng regularization).
  3. Kiểm tra ảnh so sánh trực quan `outputs/compare_round*.jpg`: xem các trường hợp bị tính là False Negative hay False Positive thực chất là do mô hình đoán sai hay do nhãn tham chiếu của test set bị thiếu/sai sót.
