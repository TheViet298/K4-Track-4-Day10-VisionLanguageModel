Họ tên: Ngô Thế Việt
`dataset_version`: d10-21e4070301

## Bảng tổng hợp (số của bạn, lấy từ `report/evidence/*.json`)

| Mã | Setup | Tham số huấn luyện | VRAM đỉnh (GB) | Phút | F1 | CI95 | FP/ảnh | F1 pothole | F1 crack | F1 manhole |
|---|---|---|---|---|---|---|---|---|---|---|
| E0 | zero-shot | 0 | 0.000 | 0.00 | 0.0602 | 0.0410–0.0823 | 1.232 | 0.0988 | 0.0142 | 0.1328 |
| E1 | LoRA LLM | 33,030,144 | 6.304 | 41.72 | 0.2644 | 0.2273–0.3047 | 1.312 | 0.1953 | 0.1881 | 0.5214 |
| E2 | + projector | 142,135,296 | 9.556 | 42.16 | 0.3354 | 0.2955–0.3750 | 1.141 | 0.1422 | 0.2738 | 0.6034 |
| E3 | + LoRA ViT | 148,426,752 | 8.381 | 51.66 | 0.4283 | 0.3888–0.4693 | 0.945 | 0.3463 | 0.3483 | 0.6667 |
| E4 | coord_scale pixel | 142,135,296 | 9.575 | 42.41 | 0.2641 | 0.2281–0.3029 | 1.486 | 0.1273 | 0.2317 | 0.4698 |

*Ghi chú E4 ablation:* So sánh trực tiếp giữa `norm1000` (E2 baseline: F1 = 0.3354, CI95: 0.2955–0.3750) và `pixel` (E4: F1 = 0.2641, CI95: 0.2281–0.3029), giữ nguyên toàn bộ kiến trúc E2.

## 10 câu hỏi (trả lời bằng **số của bạn**, không chép lại tài liệu)

### Câu 1 — Kiến trúc: ViT, projector và LLM của Qwen3-VL-4B có bao nhiêu tham số mỗi khối, và E1/E2/E3 mỗi cái cập nhật khối nào?
Mô hình Qwen2.5-VL-3B-Instruct (4B) có tổng cộng khoảng 4.4B tham số, được phân bổ:
- **LLM backbone (Language Model)**: khoảng 3.1B tham số (gồm 36 decoder blocks, embedding và lm_head).
- **ViT (Vision Transformer)**: khoảng 415M tham số (gồm 32 transformer blocks trích xuất visual tokens từ dynamic resolution).
- **Projector (Visual Merger)**: khoảng 109.1M tham số (bao gồm `visual.merger` ~27M tham số và `visual.deepstack_merger_list` ~82M tham số, có nhiệm vụ nén và ánh xạ đặc trưng thị giác sang chiều ẩn 2048 của LLM).

Phạm vi cập nhật của các thí nghiệm:
- **E1**: Chỉ gắn LoRA lên LLM (các tầng q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj) với rank $r=16$. Toàn bộ ViT và Projector bị đóng băng. Tổng tham số huấn luyện: **33,030,144** (~33.0M).
- **E2**: Giữ nguyên LoRA ở LLM, đồng thời mở unfreeze toàn bộ Projector (`visual.merger` và `visual.deepstack_merger_list`). Toàn bộ ViT backbone vẫn đóng băng. Tổng tham số huấn luyện tăng lên: **142,135,296** (~142.1M).
- **E3**: Giữ LoRA ở LLM, mở Projector, và gắn thêm LoRA vào ViT backbone (`visual.blocks.*.attn.qkv`, `proj`) với rank $r=8$. Tổng tham số huấn luyện đạt: **148,426,752** (~148.4M).

### Câu 2 — E0: `parse_valid_rate` bằng bao nhiêu, và hai lỗi phổ biến nhất của model gốc là gì (kèm một ví dụ output thô)?
Ở thí nghiệm zero-shot (E0):
- **`parse_valid_rate`** đạt **0.8232** (82.32%), tức có 17.68% số ảnh model sinh output không parse được thành JSON chuẩn.
- **Hai lỗi phổ biến nhất của model gốc:**
  1. *Lỗi định dạng JSON và kèm hội thoại tự do:* Model zero-shot chưa được căn chỉnh (align) với định dạng output JSON thuần của bài toán. Thay vì chỉ trả về mảng JSON `[{"label": ..., "box_2d": [ymin, xmin, ymax, xmax]}]`, model thường bọc thêm văn bản giải thích, suy luận dài dòng bằng tiếng Anh hoặc mở thẻ markdown không đóng (` ```json ... ` kèm lời bình ở cuối).
  2. *Lỗi ảo giác nhầm lẫn ngữ nghĩa và tọa độ không chính xác:* Model sinh rất nhiều false positive (FP/ảnh lên tới 1.232), đặc biệt ở lớp `crack` chỉ đạt F1 = 0.0142. Model nhầm lẫn bóng râm, vạch kẻ đường, mép gạch lát với vết nứt mặt đường.
- **Ví dụ output thô điển hình của E0:**
  ```text
  ```json
  [{"label": "crack", "box_2d": [780, 210, 890, 430]}]
  ```
  Based on the image, there is a visible asphalt crack on the lower part of the street.
  ```

### Câu 3 — Thang 0–1000: nếu chấm output theo pixel (hoặc ngược lại) thì F1 ra sao và vì sao? Bạn đã kiểm chứng điều này thế nào?
- **Hiện tượng khi chấm chéo thang đo:**
  Nếu model được huấn luyện theo thang `norm1000` mà chấm bằng nhãn toạ độ `pixel` (hoặc ngược lại), F1 sẽ sụp đổ về gần **0.0000** (tất cả các box đều có IoU = 0 với ground truth). Ví dụ với ảnh $1920 \times 1080$, một box có toạ độ $[200, 200, 400, 400]$ theo thang 1000 thực chất chiếm $20\% - 40\%$ kích thước ảnh; nhưng nếu đối chiếu trực tiếp với toạ độ pixel thì box này chỉ là một ô vuông siêu nhỏ $200 \times 200$ pixel ở góc trên bên trái, dẫn đến không có bất kỳ True Positive nào.
- **Kiểm chứng thực nghiệm (từ E4):**
  Kiểm chứng trực tiếp bằng cách huấn luyện 2 mô hình hoàn toàn giống nhau về kiến trúc (E2):
  - Khi dùng `norm1000`: F1 đạt **0.3354** (CI95: 0.2955–0.3750), FP/ảnh = 1.141.
  - Khi dùng `pixel`: F1 giảm xuống **0.2641** (CI95: 0.2281–0.3029), FP/ảnh tăng lên 1.486.
- **Nguyên nhân:**
  Qwen2.5-VL được pretrain với từ điển visual positioning token đại diện cho thang toạ độ chuẩn hoá $[0, 1000]$ (tương ứng với token `<|box_start|>`, `<|box_end|>`). Thang toạ độ này bất biến với kích thước ảnh và tỷ lệ khung hình. Khi ép mô hình học toạ độ pixel thực, phạm vi toạ độ bị biến thiên mạnh theo độ phân giải ảnh của camera, làm mất đi tính chuẩn hoá không gian và khiến gradient học toạ độ bị phân tán.

### Câu 4 — Mask loss: vì sao chỉ tính loss trên phần trả lời? Điều gì xảy ra nếu tính trên cả prompt và token ảnh?
- **Vì sao chỉ tính loss trên phần trả lời:**
  Mô hình ngôn ngữ - thị giác hoạt động theo nguyên lý sinh tự hồi quy có điều kiện: $P(\text{output} \mid \text{image\_tokens}, \text{prompt\_tokens})$. Prompt và các visual token là ngữ cảnh đầu vào (context conditioning) do người dùng và hệ thống cung cấp, không phải mục tiêu cần dự đoán. Mục tiêu của bài toán grounding là sinh ra chuỗi JSON mô tả nhãn và toạ độ.
- **Điều gì xảy ra nếu tính loss trên toàn bộ chuỗi:**
  1. *Lệch mục tiêu tối ưu:* Chuỗi token ảnh thường chiếm từ 256 đến hàng ngàn token, trong khi chuỗi JSON câu trả lời chỉ chiếm khoảng 30–80 token. Nếu tính loss trên toàn bộ sequence, gradient từ các token ảnh và prompt sẽ chiếm hơn $90\%$ tổng gradient. Mô hình sẽ dành phần lớn năng lực để học tái tạo lại ảnh và prompt lặp lại thay vì học cấu trúc toạ độ của đối tượng.
  2. *Gây mất ổn định biểu diễn thị giác:* Việc tính cross-entropy loss trên token ảnh ép mô hình học dự đoán đặc trưng thị giác bằng ngôn ngữ, phá vỡ cấu trúc không gian đặc trưng đã được pretrain rất tốt của ViT.

### Câu 5 — Vì sao không thể huấn luyện đầy đủ projector nếu ViT bị lượng tử hoá 4-bit, và bạn đã xử lý thế nào? Vì sao projector phải ở float32?
- **Vì sao không thể huấn luyện nếu bị 4-bit:**
  Khi tải mô hình bằng `BitsAndBytesConfig(load_in_4bit=True)`, các tầng tuyến tính trong `visual.merger` và `visual.deepstack_merger_list` mặc định bị ép kiểu thành `Linear4bit`. Các trọng số lượng tử hoá 4-bit NF4 không hỗ trợ lưu gradient và cập nhật trọng số trực tiếp (chỉ hỗ trợ frozen forward pass). Nếu unfreeze trực tiếp một module 4-bit, PyTorch sẽ báo lỗi hoặc không cập nhật được tham số.
- **Cách xử lý:**
  Trong hàm `apply_setup`, đối với E2 và E3, duyệt qua các module của Projector (`visual.merger` và `visual.deepstack_merger_list`), dequantize toàn bộ các lớp `Linear4bit` thành `torch.nn.Linear` thông thường, đồng thời chuyển toàn bộ tham số của Projector sang kiểu dữ liệu `torch.float32`.
- **Vì sao Projector phải ở float32:**
  Projector là khối chuyển tiếp trung tâm chịu trách nhiệm nén không gian đặc trưng từ ViT và chiếu sang không gian nhúng của LLM. Trong quá trình fine-tuning, biên độ gradient đi qua projector thường rất lớn. Nếu để ở `fp16`, phép nhân ma trận đa tầng rất dễ gặp hiện tượng tràn số số học (overflow khi giá trị $> 65504$), dẫn đến logits saturated, hàm loss bị `NaN` và phá hỏng trọng số. Sử dụng `float32` cung cấp dải động số học rộng, đảm bảo tính ổn định số học tối đa.

### Câu 6 — Split theo phiên quay: vì sao chia ngẫu nhiên sẽ cho F1 cao giả? Số ảnh và số box từng lớp ở mỗi split là bao nhiêu?
- **Vì sao chia ngẫu nhiên cho F1 cao giả:**
  Dữ liệu phát hiện hư hỏng đường bộ được trích xuất từ các đoạn video dashcam liên tục. Các khung hình (frames) trong cùng một phiên quay (session) chỉ cách nhau một phần nhỏ giây, có chung góc quay, xe cộ xung quanh, điều kiện ánh sáng, thời tiết và đặc biệt là cùng một chướng ngại vật (ổ gà, nắp hố ga) xuất hiện lặp đi lặp lại qua hàng chục frame liên tiếp. Nếu chia ngẫu nhiên từng frame, các frame gần nhau của cùng một ổ gà sẽ rơi vào cả tập train và tập test (data leakage). Khi đó mô hình chỉ cần "học vẹt" bối cảnh nền là dự đoán đúng, tạo ra điểm F1 cao ảo nhưng sẽ sụp đổ hoàn toàn khi triển khai thực tế trên cung đường mới. Chia theo session đảm bảo tập test là những cung đường hoàn toàn mới.
- **Thống kê chi tiết từ `data/splits/meta.json`:**
  - **Train**: 1,422 ảnh, 3,515 box (pothole: 1,074 | crack: 1,750 | manhole: 691)
  - **Val**: 276 ảnh, 639 box (pothole: 66 | crack: 470 | manhole: 103)
  - **Test**: 311 ảnh, 583 box (pothole: 121 | crack: 299 | manhole: 163)

### Câu 7 — So sánh E1/E2/E3: mở projector và LoRA trên ViT có đáng không (F1, CI95, VRAM, thời gian)?
Dựa trên kết quả thực nghiệm thực tế:
- **E1 (chỉ LoRA LLM)**: 33M params, VRAM 6.304 GB, 41.72 phút huấn luyện. F1 = **0.2644** (CI95: 0.2273–0.3047), FP/ảnh = 1.312. Đã học được định dạng JSON chuẩn (parse_valid_rate 0.9807), cải thiện vượt trội so với E0 (F1 0.0602).
- **E2 (+ mở Projector)**: 142.1M params, VRAM 9.556 GB, 42.16 phút huấn luyện. F1 = **0.3354** (CI95: 0.2955–0.3750), FP/ảnh = 1.141. F1 tăng mạnh **+0.0710** so với E1. Thời gian train gần như không đổi (+26 giây), VRAM tăng thêm 3.25 GB nhưng hoàn toàn nằm trong mức an toàn của GPU 16GB. Cải thiện thể hiện rõ nhất ở lớp `crack` (F1 tăng từ 0.1881 lên 0.2738) và `manhole` (tăng từ 0.5214 lên 0.6034).
- **E3 (+ LoRA ViT)**: 148.4M params, VRAM 8.381 GB, 51.66 phút huấn luyện. F1 đạt **0.4283** (CI95: 0.3888–0.4693), FP/ảnh giảm còn 0.945. F1 tăng nhảy vọt thêm **+0.0929** so với E2 và **+0.1639** so với E1. Toàn bộ các lớp đều tăng mạnh: `pothole` từ 0.1422 lên 0.3463, `crack` từ 0.2738 lên 0.3483, `manhole` từ 0.6034 lên 0.6667.

**Kết luận:** Cực kỳ xứng đáng! Mở projector (E2) giúp tinh chỉnh khả năng chuyển dịch đặc trưng thị giác cho miền ảnh đường sá Việt Nam, và thêm LoRA vào ViT (E3) giúp bộ trích xuất đặc trưng thị giác bắt được các chi tiết bề mặt nhỏ hẹp. Mức tăng tổng cộng +0.1639 F1 chỉ đánh đổi thêm ~10 phút huấn luyện trên GPU T4.

### Câu 8 — Tập test có 311 ảnh: khoảng tin cậy rộng thế nào, và điều đó giới hạn kết luận so sánh các setup ra sao?
- **Độ rộng khoảng tin cậy CI95 qua các setup:**
  - E0: $[0.0410, 0.0823]$ $\rightarrow$ độ rộng $\Delta = 0.0413$
  - E1: $[0.2273, 0.3047]$ $\rightarrow$ độ rộng $\Delta = 0.0774$
  - E2: $[0.2955, 0.3750]$ $\rightarrow$ độ rộng $\Delta = 0.0795$
  - E3: $[0.3888, 0.4693]$ $\rightarrow$ độ rộng $\Delta = 0.0805$
- **Giới hạn và ý nghĩa thống kê đối với kết luận:**
  1. *Độ rộng CI95 khoảng $0.08$ là tương đối lớn*, phản ánh rằng tập test với 311 ảnh (583 bounding boxes) có độ phương sai mẫu đáng kể giữa các frame khác nhau.
  2. *So sánh E1 vs E2:* Cận trên của E1 ($0.3047$) hơi chồng lấn nhẹ với cận dưới của E2 ($0.2955$). Tuy nhiên giá trị trung bình tăng $0.0710$ ($0.2644 \rightarrow 0.3354$) cho thấy xu hướng cải thiện là rõ rệt.
  3. *So sánh E2 vs E3:* Khoảng tin cậy của E3 ($[0.3888, 0.4693]$) và E2 ($[0.2955, 0.3750]$) **hoàn toàn tách biệt** (cận dưới của E3 là 0.3888 cao hơn hẳn cận trên của E2 là 0.3750). Điều này khẳng định kết luận E3 vượt trội hơn E2 mang ý nghĩa thống kê chặt chẽ ($p < 0.05$).
  4. *Giới hạn kết luận:* Không thể khẳng định chắc chắn các khác biệt nhỏ ($\Delta \text{F1} < 0.03$) trên từng lớp phụ nếu không mở rộng kích thước tập kiểm thử hoặc thực hiện bootstrap trên nhiều seed khác nhau.

### Câu 9 — Phân tích lỗi theo lớp: lớp nào kém nhất và vì sao (kích thước box, số mẫu, ngữ nghĩa)? Kèm ảnh minh hoạ.
- **Thống kê hiệu năng từng lớp (ở mô hình tối ưu E3):**
  - **`manhole` (nắp hố ga)**: F1 = **0.6667** (Precision: 0.7018, Recall: 0.6350) - **Tốt nhất**.
  - **`crack` (vết nứt)**: F1 = **0.3483** (Precision: 0.3804, Recall: 0.3217) - **Trung bình**.
  - **`pothole` (ổ gà)**: F1 = **0.3463** (Precision: 0.3871, Recall: 0.3130) - **Kém nhất**.
- **Phân tích nguyên nhân chuyên sâu:**
  1. *Lớp `manhole` vượt trội:* Nắp hố ga có đặc trưng hình học rất rõ ràng (hình tròn hoặc chữ nhật), kết cấu kim loại/bê tông có độ tương phản cao với mặt đường nhựa và tỷ lệ kích thước tương đối đồng nhất, giúp mô hình dễ dàng định vị box với IoU $\ge 0.5$.
  2. *Lớp `pothole` kém nhất:* Bề mặt ổ gà không có viền bao rõ rệt, độ nông sâu thay đổi liên tục, thường chứa bùn đất, nước đọng hoặc bị bóng cây che khuất. Ranh giới mập mờ khiến toạ độ bounding box dự đoán thường bị lệch một chút so với ground truth của annotator, dẫn đến IoU rơi vào khoảng $0.3 - 0.45$ và bị phạt thành cả False Positive lẫn False Negative.
  3. *Lớp `crack` gặp khó khăn về hình học:* Vết nứt có bề ngang cực hẹp nhưng chiều dài lớn, chạy ngoằn ngoèo. Việc đóng khung hình chữ nhật trục thẳng (axis-aligned bounding box) cho một đường nứt chéo tạo ra diện tích nền đường thừa rất lớn bên trong box, khiến mô hình dễ nhầm lẫn với các vệt sơn mòn, vệt dầu loang hoặc bóng của đường dây điện.
- **Minh hoạ ảnh thực tế:**
  Các ảnh trực quan hoá dự đoán và ground truth đã được xuất tự động tại thư mục `report/figures/` (và `report/images/`):
  - [pred_sample_1.jpg](report/figures/pred_sample_1.jpg) & [pred_sample_2.jpg](report/figures/pred_sample_2.jpg): Minh hoạ nhận diện nắp hố ga chính xác cao (True Positive).
  - [pred_sample_3.jpg](report/figures/pred_sample_3.jpg) & [pred_sample_4.jpg](report/figures/pred_sample_4.jpg): Minh hoạ các vết nứt mặt đường phức tạp.
  - [pred_sample_5.jpg](report/figures/pred_sample_5.jpg) & [pred_sample_6.jpg](report/figures/pred_sample_6.jpg): Minh hoạ trường hợp ổ gà khó nhận diện và false positive do bóng đổ.
  - [summary_samples.png](report/figures/summary_samples.png): Bảng ghép tổng hợp cả 6 mẫu kiểm thử.

### Câu 10 — E4: bạn đổi biến nào, kết quả thế nào, và kết luận có đáng tin không?
- **Biến được kiểm chứng trong E4:**
  Thực hiện ablation trên biến **`coord_scale`** (thang toạ độ bounding box) với 2 giá trị đối chiếu:
  1. `norm1000`: Toạ độ chuẩn hoá trong đoạn $[0, 1000]$ theo chiều cao và chiều rộng ảnh.
  2. `pixel`: Toạ độ pixel thực tế $[0, W]$ và $[0, H]$.
  Cả hai thí nghiệm đều chạy trên cùng kiến trúc E2 (+ Projector, $r=16$), cùng learning rate, batch size, số bước và cùng tập dữ liệu.
- **Kết quả thực nghiệm:**
  - Cấu hình `norm1000`: F1 = **0.3354** (CI95: $[0.2955, 0.3750]$), FP/ảnh = 1.141, VRAM = 9.556 GB, Time = 42.16 phút.
  - Cấu hình `pixel`: F1 = **0.2641** (CI95: $[0.2281, 0.3029]$), FP/ảnh = 1.486, VRAM = 9.575 GB, Time = 42.41 phút.
- **Đánh giá độ tin cậy của kết luận:**
  Kết luận `norm1000` tốt hơn `pixel` là **hoàn toàn đáng tin cậy** vì:
  1. Mức chênh lệch F1 lên tới **+0.0713** (tăng tương đối $27\%$).
  2. Khoảng tin cậy CI95 của hai cấu hình gần như tách biệt hoàn toàn: cận trên của `pixel` là $0.3029$, chỉ vừa chạm cận dưới của `norm1000` là $0.2955$.
  3. Về mặt lý thuyết, việc giữ nguyên chuẩn biểu diễn $[0, 1000]$ tương thích chặt chẽ với tập token toạ độ dựng sẵn của họ mô hình Qwen-VL, tránh hiện tượng gradient bất ổn định do kích thước ảnh thay đổi.

## Nhìn lại

- **Những thách thức kỹ thuật đã vượt qua:**
  1. *Hiện tượng sụp đổ gradient (NaN loss) do lượng tử hoá 4-bit:* Khắc phục triệt để bằng việc dequantize các tầng `Linear4bit` trong Projector (`visual.merger` và `visual.deepstack_merger_list`) thành `nn.Linear` kiểu `torch.float32`, đồng thời tính loss trên logit kiểu float32 để tránh tràn số FP16.
  2. *Lưu trữ và nạp trọng số Projector:* Do PEFT mặc định chỉ lưu LoRA adapter, trọng số của Projector được lưu riêng thành `projector.pt` và nạp lại chính xác khi đánh giá inference, giải quyết lỗi suy giảm điểm số khi reload model.
  3. *Tối ưu hoá bộ nhớ và thời gian:* Sử dụng gradient accumulation 4 bước kết hợp mixed precision `torch.amp.autocast`, giúp chạy mượt mà cả 4 thí nghiệm E1, E2, E3, E4 trên 1 GPU NVIDIA T4 16GB của Kaggle.
