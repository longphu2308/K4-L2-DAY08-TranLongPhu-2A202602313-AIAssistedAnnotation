# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu chỉ có ngân sách rà soát 5 ảnh từ top 50 của `outputs/selection_round1.csv`, 5 frame tối ưu được chọn nhằm tối đa hóa độ bất định và phân bổ đa dạng theo trục thời gian (tránh lãng phí công rà trên các ảnh gần trùng) là:

1. **`frame_0182.jpg`** (Thứ tự: rank 1 | Điểm score: 0.9591 | Thời điểm: t = 72.8s | U = 0.9182, A = 1.0, D = 1.0, 28 box, 18 ambiguous):
   - *Lý do:* Đứng đầu toàn bộ pool về điểm số tổng hợp. Tỷ lệ box mập mờ đạt cực đại ($A = 1.0$ với 18 box ambiguous), phản ánh vùng mật độ xe cao mà mô hình phân vân nhiều nhất. Đại diện cho khoảng thời gian giữa video (t ~ 73s).
2. **`frame_0369.jpg`** (Thứ tự: rank 2 | Điểm score: 0.9324 | Thời điểm: t = 147.6s | U = 0.9315, A = 0.8889, D = 1.0, 43 box, 16 ambiguous):
   - *Lý do:* Điểm cao thứ hai trong pool, đại diện cho đoạn cuối video với mật độ phương tiện đông đúc nhất (43 box) và độ bất định rất cao ($U = 0.9315$).
3. **`frame_0326.jpg`** (Thứ tự: rank 4 | Điểm score: 0.9155 | Thời điểm: t = 130.4s | U = 0.9310, A = 0.8333, D = 1.0, 39 box, 15 ambiguous):
   - *Lý do:* Bổ sung mẫu ở phân đoạn t ~ 130s với 39 box và $U = 0.9310$. **Xét ảnh gần trùng:** Chọn `frame_0326.jpg` và chủ động loại bỏ `frame_0331.jpg` (rank 5, t = 132.4s, cách 2.0s), `frame_0330.jpg` (rank 12, t = 132.0s, cách 1.6s) và `frame_0372.jpg` (rank 6, t = 148.8s, cách `frame_0369.jpg` chỉ 1.2s < MIN_GAP_S). Vì camera tĩnh gắn trên cầu vượt, các xe di chuyển rất ít trong 1-2s, nếu chọn các ảnh sát nhau sẽ lãng phí ngân sách 5 ảnh vào các khung cảnh gần như trùng lặp (near-duplicates) mà không mang lại tri thức mới.
4. **`frame_0099.jpg`** (Thứ tự: rank 8 | Điểm score: 0.9063 | Thời điểm: t = 39.6s | U = 0.9460, A = 0.7778, D = 1.0, 29 box, 14 ambiguous):
   - *Lý do:* Có độ bất định top 5 box cao nhất trong top 10 ($U = 0.9460$), đại diện cho phân đoạn đầu video (t ~ 40s). Đây là frame chứa nhiều xe nhỏ ở xa và xe chìm trong bóng tối mép trái mà mô hình cold start bỏ sót nhiều (thực tế kiểm tra độc lập thấy 26 xe nhưng model chỉ đề xuất 13 box).
5. **`frame_0270.jpg`** (Thứ tự: rank 13 | Điểm score: 0.8878 | Thời điểm: t = 108.0s | U = 0.9089, A = 0.7778, D = 1.0, 35 box, 14 ambiguous):
   - *Lý do:* Lấp khoảng trống thời gian lớn ở mốc t ~ 108s giữa `frame_0182.jpg` (72.8s) và `frame_0326.jpg` (130.4s). Frame này có độ bất định cao ($U > 0.90$) và 35 box, giúp bộ dữ liệu 5 ảnh trải đều toàn bộ video (từ 39.6s đến 147.6s) mà không bị thiên lệch cục bộ.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **`frame_0182.jpg`** (rank 1, t = 72.8s):
   - *Bằng chứng CSV (`selection_round1.csv`):* `score = 0.9591` (cao nhất pool), `U = 0.9182`, `A = 1.0` (số box ambiguous đạt max 18 box trên 28 box đề xuất), `D = 1.0`.
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 1 cột 3. Hình ảnh cho thấy nhiều xe ở cự ly trung bình và xa, đốm đèn xe giao nhau và xe ở làn ngược chiều khiến mô hình có nhiều box conf lưng chừng (màu vàng/cam).
2. **`frame_0369.jpg`** (rank 2, t = 147.6s):
   - *Bằng chứng CSV (`selection_round1.csv`):* `score = 0.9324`, `U = 0.9315`, `A = 0.8889` (16 box ambiguous), tổng số 43 box đề xuất.
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 2 cột 4. Mật độ xe dày đặc ở các làn bên trái, các xe ở hậu cảnh xa sát nhau gây ra hiện tượng che khuất liên hoàn (occlusion) trong điều kiện ánh sáng yếu.
3. **`frame_0099.jpg`** (rank 8, t = 39.6s):
   - *Bằng chứng CSV (`selection_round1.csv`):* `score = 0.9063`, `U = 0.9460` (độ bất định top 5 box cao nhất trong lô), `A = 0.7778` (14 box ambiguous).
   - *Bằng chứng contact sheet (`selection_round1.jpg`):* Nằm ở hàng 1 cột 1. Xe ở tiền cảnh có đèn pha sáng rực rọi xuống mặt đường, trong khi ở hậu cảnh xa và góc mép trái có nhiều xe tối chỉ lộ 1-2 vệt đèn yếu ớt. Thực tế kiểm tra độc lập (`BLIND_SCAN.md`) ghi nhận 26 xe nhưng model chỉ đề xuất 13 box conf >= 0.25 (thiếu một nửa số xe).

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

- **Frame có điểm cao nhưng KHÔNG chọn: `frame_0372.jpg`**
  - *Dữ liệu:* Rank 6, thời điểm `t = 148.8s`, `score = 0.9101`, `U = 0.9202`, `A = 0.8333`, 42 box đề xuất.
  - *Lý do không chọn:* Dù điểm score đứng thứ 6 trong toàn bộ 268 frame của pool, frame này không được chọn vào lô 12 ảnh vì vi phạm ràng buộc khoảng cách thời gian tối thiểu `MIN_GAP_S = 2.0s`. Cụ thể, frame `frame_0369.jpg` có score cao hơn (0.9324, rank 2) tại `t = 147.6s` đã được chọn trước đó. Khoảng cách thời gian giữa hai frame chỉ là $|148.8 - 147.6| = 1.2s < 2.0s$. Do camera cố định trên cầu vượt, trong 1.2 giây các xe trên đường chỉ nhích một đoạn ngắn, góc nhìn, điều kiện ánh sáng và trạng thái che khuất gần như giống hệt nhau (ảnh gần trùng - near-duplicate). Việc loại bỏ `frame_0372.jpg` là quyết định chính xác nhằm tiết kiệm công gán nhãn, nhường ngân sách cho các frame khác mang lại thông tin mới mẻ hơn.
- *(Hoặc xét frame điểm thấp vẫn NÊN xem: `frame_0195.jpg`)*
  - *Dữ liệu:* Rank 268 (thấp nhất pool), `t = 78.0s`, `score = 0.5721`, `U = 0.6442`, `A = 0.1667` (chỉ 3 box ambiguous), 20 box đề xuất.
  - *Lý do vẫn nên xem:* Điểm thấp không chứng minh mô hình đã phát hiện tốt. Mô hình có thể hoàn toàn tự tin (conf cao) vào một vài xe lớn ở tiền cảnh trong khi bỏ sót hoàn toàn các xe nhỏ ở xa (conf < 0.05). Khi không có box nào trong vùng ranh giới, chỉ số $U$ và $A$ bị kéo tụt xuống, tạo ra cảm giác "an toàn giả tạo" cho thuật toán chọn mẫu.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

Phép chọn mẫu của Active Learning (Uncertainty Sampling kết hợp Temporal Diversity) chỉ là một cơ chế heuristic để ưu tiên dữ liệu gán nhãn dựa trên sự phân vân nội tại của mô hình và khoảng cách thời gian. Nó hoàn toàn CHƯA chứng minh:
1. **Không chứng minh mô hình sẽ tự động cải thiện hoặc tăng AP50 sau khi fine-tune:** Việc chọn các mẫu mà mô hình phân vân nhất chỉ đảm bảo đưa vào huấn luyện các trường hợp khó (hard examples). Khi tập huấn luyện quá nhỏ (chỉ 12 ảnh), việc học các mẫu quá khó hoặc nhãn có độ nhiễu có thể dẫn đến hiện tượng quên thảm họa (catastrophic forgetting), co cụm ngưỡng tự tin, làm giảm Recall tổng thể hoặc tụt AP50 như thực tế đã ghi nhận ở vòng 1 (AP50 giảm từ 0.771 xuống 0.458).
2. **Điểm bất định cao không đồng nghĩa với nhãn sai, và điểm bất định thấp không đồng nghĩa với nhãn đúng:** Mô hình có thể cực kỳ tự tin (conf > 0.9, U thấp) vào một dự đoán sai (False Positive lớn, ví dụ vệt đèn đường phản chiếu), hoặc hoàn toàn bỏ sót đối tượng (False Negative với conf < 0.05) khiến U và A đều bằng 0. Ngược lại, điểm bất định cao đôi khi chỉ phản ánh nhiễu ngẫu nhiên vốn có của dữ liệu ban đêm (aleatoric uncertainty) như ánh đèn nhòe hay xe bị che khuất gần hết không đủ thông tin nhận dạng.
3. **Không chứng minh được khả năng tổng quát hóa trên tập kiểm thử độc lập:** Phép chọn diễn ra trên tập pool và chỉ phản ánh trạng thái nội tại của mô hình hiện tại đối với pool. Chất lượng thực sự của mô hình chỉ có thể được chứng minh thông qua đánh giá khách quan trên một tập kiểm thử độc lập (test set) với ground-truth chuẩn mực.
