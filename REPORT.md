Họ tên: Ngô Thế Việt
`dataset_version`: d10-21e4070301

## Bảng tổng hợp (số của bạn, lấy từ `report/evidence/*.json`)

| Mã | Setup | Tham số huấn luyện | VRAM đỉnh (GB) | Phút | F1 | CI95 | FP/ảnh | F1 pothole | F1 crack | F1 manhole |
|---|---|---|---|---|---|---|---|---|---|---|
| E0 | zero-shot | 0 | chưa ghi | chưa ghi | 0.073 | 0.051–0.097 | 0.99 | 0.150 | 0.015 | 0.158 |
| E1 | LoRA LLM | 33,030,144 | 5.242 | 25.32 | 0.2016 | 0.1679–0.2387 | 1.424 | 0.141 | 0.131 | 0.441 |
| E2 | + projector | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy |
| E3 | + LoRA ViT | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy |
| E4 | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy | chưa chạy |

## 10 câu hỏi (trả lời bằng **số của bạn**, không chép lại tài liệu)

### Câu 1 — Kiến trúc: ViT, projector và LLM của Qwen3-VL-4B có bao nhiêu tham số mỗi khối, và E1/E2/E3 mỗi cái cập nhật khối nào?
Theo cấu hình của lab, Qwen3-VL-4B có khoảng 4.0B tham số ở LLM/embedding, khoảng 415M ở ViT, trong đó merger chính khoảng 27M và deepstack merger khoảng 82M. E1 chỉ đặt LoRA ở LLM; E2 mở thêm projector; E3 mở thêm LoRA ở ViT.

### Câu 2 — E0: `parse_valid_rate` bằng bao nhiêu, và hai lỗi phổ biến nhất của model gốc là gì (kèm một ví dụ output thô)?
E0 có parse_valid_rate = 0.862. Các lỗi quan sát được gồm output không parse được thành JSON thuần và dự đoán sai lớp/toạ độ hoặc box thừa. Đây là zero-shot nên model chưa bám ổn định format grounding của bài.

### Câu 3 — Thang 0–1000: nếu chấm output theo pixel (hoặc ngược lại) thì F1 ra sao và vì sao? Bạn đã kiểm chứng điều này thế nào?
Chưa chạy ablation pixel ở thời điểm hiện tại. Các kết quả hiện tại đều được chấm theo `norm1000`, đúng với quy ước nhãn của dataset và Qwen3-VL.

### Câu 4 — Mask loss: vì sao chỉ tính loss trên phần trả lời? Điều gì xảy ra nếu tính trên cả prompt và token ảnh?
Loss chỉ nên tính trên token câu trả lời vì prompt và token ảnh là điều kiện đầu vào, không phải mục tiêu cần model sinh lại. Nếu tính loss trên prompt/ảnh, gradient sẽ khuyến khích model học bắt chước phần điều kiện và làm loãng tín hiệu học JSON bbox.

### Câu 5 — Vì sao không thể huấn luyện đầy đủ projector nếu ViT bị lượng tử hoá 4-bit, và bạn đã xử lý thế nào? Vì sao projector phải ở float32?
Projector cần được giữ ở float32 khi train để tránh mất độ chính xác ở phép biến đổi đặc trưng thị giác sang không gian LLM. Nếu ViT/projector bị quantize 4-bit, các tham số projector không còn là trọng số float phù hợp để cập nhật đầy đủ. Cách xử lý là skip `visual` và `lm_head` khỏi quantization, đồng thời đưa projector trainable về float32 ở E2/E3.

### Câu 6 — Split theo phiên quay: vì sao chia ngẫu nhiên sẽ cho F1 cao giả? Số ảnh và số box từng lớp ở mỗi split là bao nhiêu?
Split theo session giúp các frame gần nhau không bị rơi vào cả train và test, tránh leakage làm F1 cao giả. Số liệu cố định: train 1,422 ảnh/3,515 box (pothole 1,074; crack 1,750; manhole 691), val 276 ảnh/639 box (66; 470; 103), test 311 ảnh/583 box (121; 299; 163).

### Câu 7 — So sánh E1/E2/E3: mở projector và LoRA trên ViT có đáng không (F1, CI95, VRAM, thời gian)?
E0 đạt F1 0.073. Sau khi cân bằng lấy mẫu và tích luỹ gradient, E1 đạt F1 0.202, cao hơn E0 0.129 điểm; CI95 của E1 là 0.168–0.239. E1 vẫn có chênh lệch giữa các lớp, trong đó manhole tốt nhất (F1 0.441), crack thấp nhất (F1 0.131). E2 và E3 chưa chạy.

### Câu 8 — Tập test có 311 ảnh: khoảng tin cậy rộng thế nào, và điều đó giới hạn kết luận so sánh các setup ra sao?
Test chỉ có 311 ảnh nên CI95 còn rộng: E0 là 0.051–0.097, E1 là 0.0388–0.0779. Vì vậy các chênh lệch nhỏ giữa setup chưa đủ mạnh để kết luận nếu khoảng tin cậy còn chồng lấn.

### Câu 9 — Phân tích lỗi theo lớp: lớp nào kém nhất và vì sao (kích thước box, số mẫu, ngữ nghĩa)? Kèm ảnh minh hoạ.
Trong E1 hiện tại, lỗi rõ nhất là model gần như chỉ sinh `crack`, với 440 false positive và không có true positive cho pothole/manhole. Đây mới là phân tích tạm thời; cần ảnh minh hoạ và các run E2/E3 trước khi kết luận nguyên nhân do kích thước box hay ngữ nghĩa lớp.

### Câu 10 — E4: bạn đổi biến nào, kết quả thế nào, và kết luận có đáng tin không?
E4 chưa chạy. Dự kiến dùng ablation rank với hai giá trị r = 4 và r = 64, giữ các biến khác cố định và so sánh F1 kèm CI95.

## Nhìn lại

Điểm cần kiểm tra tiếp: E1 từng bị PEFT bọc lồng khi chạy lại cùng model; sau khi train lại với model gốc, lấy mẫu cân bằng ba lớp và tích luỹ gradient 4 micro-batch, F1 đã tăng lên 0.202.
