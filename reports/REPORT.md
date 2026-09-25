# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Trần Long Phú

Công cụ gán nhãn đã dùng: CVAT

Báo cáo được hoàn thiện theo đúng cấu trúc hướng dẫn. Mọi con số được truy xuất chính xác từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json`, `outputs/round1_diff.md`, `reports/BLIND_SCAN.md` và `reports/REVIEW_LOG.csv`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

### Lý do chia theo trục thời gian và thiết lập vùng đệm

Dữ liệu của bài thực hành được trích xuất từ một video dài 160 giây (400 frame từ `frame_0000.jpg` đến `frame_0399.jpg` với tốc độ lấy mẫu 2.5 fps), ghi hình từ một camera giám sát cố định đặt trên cầu vượt nhìn xuống đường cao tốc ban đêm. Toàn bộ video là một cảnh quay liên tục duy nhất, không có bất kỳ pha chuyển cảnh (scene cut) nào; bối cảnh mặt đường và hậu cảnh hoàn toàn tĩnh.

Trong điều kiện camera cố định, các khung hình liên tiếp cách nhau chỉ 0.4 giây hầu như giống hệt nhau. Một phương tiện giao thông khi đi qua góc máy thường lưu lại trong khung hình từ vài giây đến hơn chục giây. 

Nếu chia ngẫu nhiên (random split), các khung hình cách nhau chỉ 0.4s hoặc 0.8s sẽ bị phân tán vào cả tập huấn luyện (pool/train) lẫn tập kiểm thử (test set). Khi đó, cùng một chiếc xe (hoặc cùng một luồng xe cụ thể) sẽ đồng thời xuất hiện ở cả hai tập, dẫn đến hiện tượng **rò rỉ dữ liệu nghiêm trọng (data leakage / temporal correlation leakage)**.

Để đảm bảo tính độc lập nghiêm ngặt, dữ liệu được chia theo trục thời gian (temporal split):
- **Tập kiểm thử (test set):** Gồm đúng 20 ảnh, chia thành 4 cụm thời gian (mỗi cụm 5 ảnh cách nhau 1.2s) đặt tại các tâm thời gian: giây 20, 60, 100 và 140.
- **Vùng đệm (buffer zone):** Gồm 112 ảnh trong khoảng ít nhất 4.0 giây trước và sau mỗi đoạn kiểm thử (cùng các ảnh xen giữa). Ảnh pool gần ảnh test nhất vẫn cách nhau ít nhất 4.4 giây (theo `data/DATA.md` và `data/frames.csv`). Khoảng đệm thời gian này đảm bảo mọi chiếc xe trong tập test đã hoàn toàn rời khỏi góc máy hoặc chưa từng xuất hiện trong tập pool, triệt tiêu sự tương quan dữ liệu giữa hai tập.
- **Tập chưa gán nhãn (pool):** Gồm 268 ảnh còn lại dùng cho quy trình học chủ động (active learning).

### Hướng lệch của số đo nếu chia ngẫu nhiên và giải thích nguyên nhân

Nếu chia ngẫu nhiên, số đo hiệu năng trên tập kiểm thử (như AP50, Precision, Recall, F1) sẽ bị **lệch theo hướng lạc quan giả tạo (overly optimistic / cao hơn thực tế rất nhiều)**.

Nguyên nhân là do mô hình không cần học cách khái quát hóa (generalization) các đặc trưng trừu tượng của phương tiện ban đêm (như hình khối thân xe mờ, cụm đèn pha, đèn hậu, bóng phản xạ trên mặt đường), mà chỉ cần "học vẹt" (ghi nhớ - memorization) vị trí, màu sắc, hình dạng vệt đèn của những chiếc xe cụ thể đã nhìn thấy trong tập train ở các frame cách đó chỉ vài phần mười giây. Khi đem mô hình này triển khai thực tế trên một đoạn đường khác hoặc một khoảng thời gian khác, hiệu năng phát hiện sẽ sụt giảm nghiêm trọng.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

### Dòng vòng 0 từ `reports/rounds_table.md`

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

*(Chi tiết từ `outputs/metrics_round0.json`: AP50 = 0.7714, Precision = 0.9249, Recall = 0.4888, F1 = 0.6396; TP = 197, FP = 16, FN = 206 trên tổng số 403 box tham chiếu sau khi đã lọc bỏ 14 box nhỏ dưới 16 px).*

### Phân tích các loại xe không khớp nhãn tham chiếu

Dựa vào ảnh so sánh `outputs/compare_round0.jpg` trên 4 frame kiểm thử đại diện (`frame_0050`, `frame_0150`, `frame_0250`, `frame_0350`), mô hình khởi đầu lạnh (YOLOv8n pretrained trên COCO) không khớp với nhãn tham chiếu (tạo ra nhiều box đỏ False Negative - FN) ở các loại xe sau:
1. **Xe nhỏ ở xa sát đường chân trời (small/distant vehicles):** Ở dải xa của đường cao tốc (phía trên khung hình), xe chỉ xuất hiện dưới dạng hai đốm sáng nhỏ li ti của đèn pha hoặc vệt sáng đỏ của đèn hậu, phần thân xe tối đen chìm hoàn toàn vào hậu cảnh. Mô hình COCO bỏ sót phần lớn các xe này (ví dụ ở `frame_0050` có 7 box FN đỏ ở phía xa, `frame_0350` có tới 14 box FN đỏ).
2. **Xe bị che khuất một phần (occluded vehicles):** Các xe đi ở làn giữa hoặc làn trong bị xe phía trước che khuất một phần cụm đèn hoặc thân xe.
3. **Xe đi ở làn ngược chiều hoặc sát mép phải đường:** Các xe này chỉ lộ ánh đèn hậu mờ nhạt hoặc di chuyển trong vùng tối ngoài tầm chiếu sáng của đèn đường.
4. **Xe có kích thước hoặc kết cấu đặc thù (xe tải, xe buýt dài):** Mô hình COCO đôi khi chỉ bắt được đầu xe có đèn pha mà bỏ sót phần thùng dài phía sau, dẫn đến việc kích thước box không khớp với nhãn tham chiếu.

### Ý nghĩa của độ phủ (Recall) theo kích thước xe

Bảng số đo phân rã theo kích thước (`metrics_round0.json`):
- **Xe nhỏ (`R small`):** `0.1818` (18.18% — chỉ phát hiện được 12 trên 66 box tham chiếu xe nhỏ).
- **Xe vừa (`R medium`):** `0.5473` (54.73% — phát hiện được 162 trên 296 box tham chiếu xe vừa).
- **Xe lớn (`R large`):** `0.5610` (56.10% — phát hiện được 23 trên 41 box tham chiếu xe lớn).

*Nhận xét:* Độ phủ tỉ lệ thuận rất rõ rệt với kích thước hiển thị của phương tiện. Mô hình COCO pretrained đạt độ phủ chấp nhận được ở nhóm xe vừa và lớn ở cự ly gần đến trung bình (recall ~55–56%), nhưng thất bại nặng nề ở nhóm xe nhỏ ở xa (bỏ sót tới hơn 81% số xe nhỏ). Điều này phản ánh rõ hạn chế của tập dữ liệu COCO: chủ yếu gồm các ảnh ban ngày với vật thể rõ nét, không được tối ưu để nhận biết các phương tiện chỉ hiển thị qua cụm đèn pha/đèn hậu ban đêm ở độ phân giải thấp.

### Trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai

Theo `data/DATA.md`, nhãn tham chiếu trong `data/test/labels/` được tạo tự động bởi một mô hình phát hiện đối tượng khác (pseudo-ground-truth) và **chưa từng được con người rà soát thủ công từng box**. Do đó, không thể coi nhãn tham chiếu là chân lý tuyệt đối.

Trường hợp cụ thể cần con người rà soát lại:
- **Đốm sáng phản chiếu, biển báo giao thông hoặc đèn đường ở xa:** Ở cự ly xa sát đường chân trời, mô hình tạo nhãn tham chiếu có thể đã gán nhầm đèn cao áp ven đường hoặc biển báo phản quang thành box xe (False Positive của bộ tham chiếu). Khi mô hình cold start không dự đoán box vào vị trí này, nó bị hệ thống tính oan là một lỗi bỏ sót (False Negative).
- **Hai xe đi sát nhau:** Nhãn tham chiếu có thể đã gộp hai xe đi cạnh nhau thành một box lớn duy nhất. Khi mô hình phát hiện tách thành hai box độc lập chính xác theo thực tế, nó lại bị phạt cả lỗi FP lẫn FN do độ trùng khớp IoU không đạt ngưỡng 0.5. Cần kiểm tra đối chiếu trực tiếp trên ảnh gốc trước khi khẳng định mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

### Giải thích công thức tính điểm và vai trò của `MIN_GAP_S`

Thuật toán chọn mẫu (cài đặt trong `tools/al_select.py`) kết hợp giữa chiến lược lấy mẫu bất định (Uncertainty Sampling) và đa dạng hóa thời gian (Temporal Diversity):

1. **Độ bất định của từng box $u(c) = 1 - |2c - 1|$:** Với mỗi box có độ tin cậy $c \ge 0.05$, hàm đạt giá trị cực đại $1.0$ khi $c = 0.5$ (thời điểm mô hình phân vân nhất giữa hai khả năng có xe hoặc không có xe), và giảm dần về $0$ khi $c \to 1.0$ (rất chắc chắn là xe) hoặc $c \to 0$ (rất chắc chắn không phải xe).
2. **$U$ (Top-5 Box Uncertainty):** Trung bình cộng của 5 giá trị $u(c)$ lớn nhất trong frame. Chỉ số này đại diện cho độ khó và độ phân vân của những đối tượng ranh giới tiêu biểu nhất trong ảnh.
3. **$A$ (Ambiguous Box Ratio):** Tỷ lệ số box "mập mờ" có confidence nằm trong khoảng ranh giới $[0.15, 0.50)$ chia cho số lượng box mập mờ tối đa ghi nhận trong toàn bộ pool (`n_ambiguous / max_amb`). Chỉ số này đo lường mật độ các ca khó trong frame.
4. **$D$ (Temporal Diversity):** Khoảng cách thời gian ngắn nhất từ frame đang xét tới các frame đã được gán nhãn ở các vòng trước, chuẩn hóa bằng cách chia cho `DIVERSITY_CAP_S = 10.0s` (chặn trên tại 1.0). Ở vòng 1, do chưa có frame nào được gán nhãn nên $D = 1.0$ cho mọi frame trong pool.
5. **Điểm tổng hợp:** $\text{score} = W_U \cdot U + W_A \cdot A + W_D \cdot D$ với trọng số mặc định $W_U = 0.5, W_A = 0.3, W_D = 0.2$. Với frame không có box nào vượt ngưỡng (`empty = True`), thuật toán cộng thêm `EMPTY_BONUS = 0.5` vì trong bối cảnh cao tốc luôn có xe, frame trống chứng tỏ mô hình đã bỏ sót hoàn toàn (FN nghiêm trọng).
6. **Vai trò của `MIN_GAP_S` (2.0 giây):** Là khoảng cách thời gian tối thiểu bắt buộc giữa hai frame bất kỳ được chọn trong cùng một lô (batch). Vì camera cố định trên cầu vượt, hai khung hình cách nhau dưới 2 giây có cảnh tượng và trạng thái xe hầu như giống hệt nhau (ảnh gần trùng - near-duplicates). Nếu không có `MIN_GAP_S`, thuật toán chọn tham lam sẽ gom một loạt frame liên tiếp có điểm cao, dẫn đến lãng phí công sức gán nhãn cho các dữ liệu trùng lặp. Ràng buộc `MIN_GAP_S` ép việc chọn mẫu phải rải đều theo trục thời gian, tối đa hóa tính đa dạng của tập huấn luyện.

### Dẫn chứng 3 frame trong `reports/SELECTION.md` và 1 frame khác

1. **`frame_0182.jpg` (rank 1, t = 72.8s, score = 0.9591):** Đạt điểm cao nhất pool nhờ số box ambiguous đạt tối đa ($A = 1.0$ với 18 box) và $U = 0.9182$. Đây là frame có mật độ xe ranh giới cao nhất, mô hình phân vân cực độ. Dù có tới 28 box pre-label cần rà soát, chi phí công sức bỏ ra là hoàn toàn xứng đáng với lượng thông tin thu về.
2. **`frame_0369.jpg` (rank 2, t = 147.6s, score = 0.9324):** Có mật độ xe dày đặc (43 box đề xuất, 16 box ambiguous, $U = 0.9315$). Công gán nhãn cho frame này rất lớn (thực tế người gán đã phải bổ sung thêm tới 26 box xe bị bỏ sót theo `round1_diff.md`), nhưng cung cấp một lượng lớn trường hợp xe bị che khuất và đi nối đuôi nhau ở phân đoạn cuối video.
3. **`frame_0099.jpg` (rank 8, t = 39.6s, score = 0.9063):** Có độ bất định top 5 box cao nhất trong top 10 ($U = 0.9460$), đại diện cho giai đoạn đầu video (t = 39.6s). Kiểm tra thực tế độc lập (`BLIND_SCAN.md`) ghi nhận có 26 xe nhưng model chỉ đề xuất 13 box, chứng minh điểm U cao phản ánh chính xác các ca xe nhỏ bị bỏ sót nghiêm trọng.
4. **Frame khác cân nhắc: `frame_0372.jpg` (rank 6, t = 148.8s, score = 0.9101):** Dù đứng thứ 6 toàn pool với điểm số rất cao ($U = 0.9202$, 42 box đề xuất), frame này đã bị thuật toán loại bỏ vì nó cách `frame_0369.jpg` (t = 147.6s) chỉ 1.2 giây ($< MIN\_GAP\_S = 2.0s$). Việc loại bỏ frame này là minh chứng rõ nét cho việc cân nhắc công gán nhãn và khử trùng lặp: việc bỏ công rà soát 42 box trên một khung cảnh gần như y hệt frame 0369 chỉ mang lại giá trị gia tăng tối thiểu cho mô hình.

### Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?

**KHÔNG.** Điểm bất định cao chỉ phản ánh trạng thái phân vân nội tại của mô hình hiện tại, hoàn toàn không chứng minh được rằng việc đưa ảnh đó vào huấn luyện sẽ làm mô hình tốt lên hay tăng điểm AP50.

Nguyên nhân:
- **Bất định do dữ liệu nhiễu (Aleatoric uncertainty):** Điểm bất định cao có thể sinh ra từ các yếu tố nhiễu không thể học được, chẳng hạn như ánh đèn pha rọi thẳng gây chói lóa ống kính, vệt đèn phản chiếu nhòe trên mặt đường ướt, hoặc vật thể ở quá xa bị cắt mép không đủ pixel để nhận dạng. Đưa các mẫu này vào có thể làm mô hình bị nhiễu thêm thay vì học được đặc trưng tốt.
- **Hiện tượng quên thảm họa (Catastrophic forgetting) và Overfitting:** Khi huấn luyện trên một lô ảnh quá nhỏ (chỉ 12 ảnh), việc dồn dập đưa vào các mẫu quá khó hoặc nhãn có độ nhiễu cao có thể khiến mạng nơ-ron bị co cụm trọng số, quên đi các đặc trưng nhận dạng tổng quát ban đầu của COCO, làm suy giảm hiệu năng trên tập test tổng thể (như thực tế vòng 1 đã chứng minh).
- **Mô hình tự tin sai (Overconfident False Positives) hoặc bỏ sót hoàn toàn (Extreme False Negatives):** Nếu mô hình cực kỳ tự tin vào một dự đoán sai (conf > 0.9) hoặc hoàn toàn không phát hiện đối tượng (conf < 0.05), giá trị bất định $u(c)$ đều xấp xỉ 0. Những ảnh có lỗi sai nghiêm trọng này lại có thể bị điểm score thấp và bị bỏ qua.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

### Bảng tổng hợp các vòng từ `reports/rounds_table.md`

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 359 | 0.458 | -0.314 | 1.000 | 0.107 | 0.193 | 0.000 | 0.098 | 0.342 |

### Trình bày chi tiết Vòng 1

- **Mức độ sửa nhãn gợi ý (truy xuất từ `outputs/round1_diff.md`):**
  Trên 12 ảnh được chọn cho vòng 1, mô hình đề xuất 169 box pre-label. Sau khi thực hiện rà soát và chỉnh sửa cẩn trọng trên CVAT theo đúng guideline, số lượng box huấn luyện đạt 359 box:
  - **Accepted (giữ nguyên):** 124 box (tỷ lệ chấp nhận pre-label là 73%).
  - **Edited (chỉnh sửa tọa độ/kích thước):** 26 box (chủ yếu mở rộng box để bao trọn thân xe thay vì chỉ ôm cụm đèn pha).
  - **Deleted (xóa box False Positive của model):** 19 box (loại bỏ các box bắt nhầm vệt đèn phản chiếu trên mặt đường hoặc biển báo giao thông phản quang).
  - **Added (thêm mới box False Negative bị model bỏ sót):** 209 box (số box thêm mới gấp hơn 1.2 lần toàn bộ số box AI đề xuất ban đầu, chủ yếu là các xe ở làn xa và xe tối ở mép đường).
- **Biến thiên AP50:**
  - AP50 đạt **0.458**, giảm **0.314** so với khởi đầu lạnh (0.771).
- **Nhóm xe tốt lên hoặc xấu đi theo số đo trên cùng tập test:**
  - **Xấu đi nghiêm trọng về Recall:**
    - Xe nhỏ (`R small`): giảm từ 0.182 xuống **0.000** (bỏ sót 100% xe nhỏ ở xa).
    - Xe vừa (`R medium`): giảm từ 0.547 xuống **0.098** (chỉ còn nhận diện được dưới 10%).
    - Xe lớn (`R large`): giảm từ 0.561 xuống **0.342**.
    - Tổng Recall giảm mạnh từ 0.489 xuống 0.107 (chỉ bắt được 43 trên 403 box tham chiếu so với 197 box ở cold start).
  - **Tốt lên về Precision:** Precision tăng từ 0.925 lên tuyệt đối **1.000** (False Positives giảm từ 16 về đúng 0). Mô hình không còn dự đoán sai bất kỳ box nào ở ngưỡng conf 0.25.
  - *Bản chất kỹ thuật:* Việc fine-tune 50 epochs trên một tập dữ liệu nhỏ (12 ảnh) với số lượng lớn xe tối được gán thêm (209 box added) đã làm thay đổi mạnh phân phối confidence calibration của mạng. Mô hình trở nên cực kỳ khắt khe và thận trọng; hầu hết các dự đoán bị hạ confidence score xuống dưới ngưỡng 0.25. Do đó, ở ngưỡng đánh giá cố định 0.25, mô hình đạt độ chính xác tuyệt đối nhưng đánh mất phần lớn độ phủ.

### Phân tích ca kết quả đổi sau fine-tune trong `outputs/compare_round1.jpg`

Quan sát hàng thứ nhất trên ảnh `outputs/compare_round1.jpg` (`frame_0050`):
- Ở cold start: TP = 11, FP = 2, FN = 7 (mô hình phát hiện hầu hết các xe ở cự ly gần và trung bình, hiển thị box xanh lá).
- Ở round 1: TP giảm xuống chỉ còn 1, FP = 0, FN tăng vọt lên 17.
- **Ca cụ thể:** Chiếc xe con màu tối đi ở làn giữa tiền cảnh (có đèn pha rọi sáng xuống mặt đường). Ở cold start, xe này được nhận diện chính xác là True Positive (box xanh lá). Tuy nhiên ở round 1, chiếc xe này biến mất khỏi danh sách TP (trở thành FN) hoặc confidence bị tụt dưới 0.25; mô hình sau fine-tune chỉ còn giữ lại đúng 1 xe lớn ở góc dưới bên phải.
- **Lý do có thể kiểm chứng:** Do kích thước batch huấn luyện nhỏ (12 ảnh), mạng nơ-ron bị dịch chuyển phân phối xác suất và co dãn ngưỡng tự tin. Các xe ở cự ly trung bình vẫn có thể được mạng nhận diện nhưng điểm confidence bị rơi vào khoảng 0.10 – 0.20, dẫn đến việc bị bộ lọc `conf_thr = 0.25` loại bỏ hoàn toàn trong bước đánh giá.

### Phân biệt quan sát độc lập, lỗi pre-label và kết quả mô hình sau train

- **Quan sát độc lập (`reports/BLIND_SCAN.md` trên `frame_0099.jpg`):** Người quan sát nhìn trực tiếp ảnh gốc khi chưa bật nhãn AI, đếm được 26 xe và chỉ ra hai vị trí nguy cơ: *"Ở góc khuất thiếu đèn, xa camera. Xe chỉ thấy 1-2 đèn xe trước, hoặc chỉ thấy đèn sau xe, phần thân xe tối đen không thấy rõ."*
- **Lỗi pre-label đã sửa (`outputs/round1_diff.md` & `reports/REVIEW_LOG.csv`):** Pre-label của YOLO cold start trên `frame_0099.jpg` chỉ đưa ra 13 box (bỏ sót đúng 13 xe). Người gán nhãn đã giữ 6, sửa 6, xóa 1 và bổ sung thêm 14 xe bị bỏ sót để đạt đúng 26 box (khớp tuyệt đối với số xe nhìn thấy trong blind scan). Trong `REVIEW_LOG.csv`, ca cụ thể được ghi nhận: `"1, frame_0099.jpg, xe tối gần mép trái, added, Thấy thân xe và đủ ranh giới theo GUIDELINE_LABEL.md"`.
- **Kết quả mô hình sau train:** Sau khi fine-tune với 12 ảnh, mô hình chưa thể tổng quát hóa ngay để bắt được toàn bộ các xe khó này trên 20 ảnh test, mà ngược lại bị co cụm confidence do số lượng mẫu huấn luyện còn quá ít.
- *Ý nghĩa phân biệt:* Cho thấy rõ ba giai đoạn: (1) Nhận định khách quan của con người (nhận diện đầy đủ đối tượng thực tế), (2) Sự can thiệp sửa nhãn (chuẩn hóa dữ liệu đầu vào cho AI), và (3) Khả năng hấp thu tri thức của mô hình học sâu (cần đủ số lượng mẫu và điều chỉnh siêu tham số phù hợp mới phát huy hiệu quả).

### Mô tả một ca khó theo guideline

- **Tình huống:** Xe chạy ở làn ngược chiều hoặc xe đi phía xa chỉ nhìn thấy hai đốm đèn đỏ của đèn hậu, phần thân xe tối đen chìm trong bóng đêm (tương ứng với ca ghi nhận trong `reports/REVIEW_LOG.csv` trên `frame_0392.jpg`: `"xe buýt trắng, edited, Chỉ bao quanh đèn xe nên sửa lại bao cả thân xe theo GUIDELINE_LABEL.md"`).
- **Quy tắc theo [GUIDELINE_LABEL.md](GUIDELINE_LABEL.md):** *"Chỉ thấy đèn, thân xe tối nhưng vẫn đoán được đường viền -> Vẽ box theo phần thân xe đoán được quanh cụm đèn, không chỉ khoanh hai chấm đèn. Không tính vệt sáng của đèn pha chiếu xuống mặt đường."*
- **Khó khăn thực tế:** AI ban đầu chỉ tạo box nhỏ xíu bao quanh hai chấm đèn (vì chỉ có đốm sáng mới kích hoạt đặc trưng biên độ cao của mạng), hoặc ngược lại kéo dài box bao trùm cả vệt đèn phản chiếu loang trên mặt đường. Người gán nhãn phải quan sát vệt mờ của nóc xe, kính sau hoặc vệt bánh xe để ước lượng đường viền vật lý của xe, điều chỉnh box vuông vức bao trọn thân xe mà không lấn ra ngoài.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

### Đánh giá kết quả vòng 1 so với cold start và quyết định dừng hay tiếp tục

So với mô hình khởi đầu lạnh, mô hình sau vòng 1 có AP50 giảm từ 0.771 xuống 0.458 ($\Delta = -0.314$), Recall giảm từ 0.489 xuống 0.107, mặc dù Precision tăng lên mức tuyệt đối 1.000. Mô hình trở nên quá dè dặt và bỏ sót toàn bộ xe nhỏ ở cự ly xa ($R_{small} = 0.000$).

**Quyết định: Cần tiếp tục thực hiện Vòng 2** (không dừng lại). Sự sụt giảm AP50 ở vòng 1 là hiện tượng thường gặp khi fine-tune mạng nơ-ron trên một lô dữ liệu quá nhỏ (chỉ 12 ảnh), khiến mô hình bị sốc phân phối nhãn và co cụm confidence. Dừng lại ở vòng 1 đồng nghĩa với việc chấp nhận một mô hình có độ phủ rất kém. Việc tiếp tục vòng 2 với các ảnh được chọn thông minh sẽ bổ sung thêm dữ liệu đa dạng để hiệu chỉnh lại trọng số mạng, giúp mô hình phục hồi Recall và cải thiện AP50.

### Đề xuất hai ca còn yếu hoặc bất định cho vòng sau

1. **Ca 1: Nhóm xe nhỏ ở cự ly xa sát đường chân trời (Distant/Small vehicles):**
   - *Ví dụ mẫu:* `frame_0114.jpg` (t = 45.6s, score = 0.8625, có mật độ xe nhỏ ở dải xa rất cao).
   - *Chi phí rà nhãn:* Trung bình đến cao (khoảng 30 box, đòi hỏi phóng to màn hình để căn chỉnh tỉ mỉ từng box nhỏ quanh cụm đèn yếu).
   - *Nguy cơ ảnh gần trùng:* Cần lưu ý khoảng cách với `frame_0107.jpg` (t = 42.8s) đã chọn ở vòng 1; khoảng cách là 2.8s ($> MIN\_GAP\_S = 2.0s$), an toàn về mặt kỹ thuật nhưng do xe ở xa di chuyển chậm, cần rà soát cẩn thận để tránh lặp lại cùng một nhóm xe.
2. **Ca 2: Xe lớn / xe tải và xe bị che khuất ở dải phân cách mép phải (Large / Occluded vehicles):**
   - *Ví dụ mẫu:* `frame_0218.jpg` (t = 87.2s, score = 0.8669, 33 box đề xuất).
   - *Chi phí rà nhãn:* Cao (mật độ xe đan xen phức tạp, cần phân định ranh giới giữa thân xe và bóng phản chiếu trên dải phân cách, điều chỉnh các box xe bị che khuất một phần).
   - *Nguy cơ ảnh gần trùng:* Cách xa frame 0187 (74.8s) hơn 12s và frame 0227 (90.8s) 3.6s, tính đa dạng thời gian rất cao ($D \approx 0.36$), nguy cơ gần trùng rất thấp.

### Ảnh hưởng của các giới hạn tập kiểm thử đến kết luận

1. **Tập kiểm thử chỉ có 20 ảnh:** Quy mô mẫu 20 ảnh là tương đối nhỏ, dẫn đến phương sai thống kê (statistical variance) cao. Việc phát hiện thêm hay bỏ sót một vài box trên 1–2 ảnh có thể làm biến động chỉ số AP50 từ 0.05 đến 0.10. Do đó, không nên kết luận vội vã chỉ dựa trên sự thay đổi nhỏ của số đo.
2. **Luật bỏ qua xe quá nhỏ (< 16 px):** Giúp loại bỏ các tranh cãi về các đốm sáng li ti không thể khẳng định chắc chắn bằng mắt, nhưng cũng đồng nghĩa với việc số đo không phản ánh năng lực phát hiện của mô hình ở vùng biên cực xa.
3. **Nhãn tham chiếu do mô hình AI tạo chưa rà thủ công (pseudo-labels):** Đây là giới hạn quan trọng nhất. Nhãn test bản thân nó đã chứa các lỗi chủ quan của mô hình tạo nhãn gốc (bỏ sót xe tối, bắt nhầm đèn đường). Khi người học viên gán nhãn rất kỹ ở tập train (bổ sung 209 xe thật), mô hình học theo phong cách gán nhãn chuẩn của con người, nhưng khi đánh giá lại bị so khớp với nhãn máy cũ, dẫn đến việc AP50 bị giảm sút giả tạo. Cần đối chiếu trực quan bằng ảnh `compare_round*.jpg` song song với việc đọc số đo số học.

### Quy trình kiểm tra nếu AP50 giảm trước khi train thêm

Nếu AP50 giảm, trước khi tiếp tục train vòng mới cần thực hiện kiểm tra có hệ thống theo 4 bước:
1. **Kiểm tra tính nhất quán của nhãn (Label Consistency Check):** Mở các file trong `labels/round1/` và rà soát lại xem có vi phạm guideline không: có vẽ box bao trùm cả vệt đèn pha rọi xuống mặt đường không? Có gán nhầm bóng đèn đường không? Kích thước box quanh các xe tối có đồng nhất giữa các frame không?
2. **Kiểm tra phân phối Confidence Score (Calibration Check):** Kiểm tra xem mô hình có thực sự dự đoán sai vị trí hay chỉ do confidence score bị tụt xuống dưới ngưỡng 0.25 (ví dụ nằm ở mức 0.15 – 0.24). Nếu do ngưỡng confidence, cần xem xét điều chỉnh lại ngưỡng đánh giá hoặc tinh chỉnh hàm loss.
3. **Kiểm tra cấu hình huấn luyện (Hyperparameters & Overfitting):** Huấn luyện 50 epochs trên 12 ảnh có thể gây hiện tượng quá khớp (overfitting) nặng. Cần cân nhắc: (a) giảm số epoch hoặc giảm learning rate ban đầu (`lr0`); (b) đóng băng các tầng trích xuất đặc trưng của backbone (`freeze = 10`) để bảo toàn tri thức COCO; (c) tăng cường độ data augmentation để tạo thêm độ biến thiên cho dữ liệu.
4. **Phân tích định tính trên ảnh `compare_round*.jpg`:** Quan sát trực tiếp xem mô hình đang bị mất điểm do False Negative (quá thận trọng bỏ sót xe) hay False Positive (đoán bừa vệt sáng), từ đó đưa ra hướng khắc phục chính xác thay vì đoán mò.
