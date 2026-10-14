# Báo cáo lỗi và lưu ý kỹ thuật

## Lỗi đã gặp

### 1. Runtime/timeout trên Colab

E0 chạy khoảng 6–10 giây mỗi ảnh. Notebook đã được thiết kế để ghi `work/e0.jsonl` sau từng ảnh và resume các ảnh còn thiếu. Trên Kaggle, dataset được đọc từ `/kaggle/input`; trên Colab dùng Kaggle Secrets và Google Drive backup.

### 2. bitsandbytes

Kaggle runtime từng báo yêu cầu `bitsandbytes>=0.46.1`. Notebook đã thêm lệnh nâng cấp package trước khi load model; cần restart session sau khi cài để Python nạp phiên bản mới.

### 3. PEFT bị bọc lồng

Chạy `apply_setup()` nhiều lần trên cùng model gây cảnh báo `peft_config` và tạo nhiều adapter. Mỗi E1/E2/E3 phải reload model gốc trước khi áp dụng setup mới.

### 4. E1 bị lệch lớp

Run E1 ban đầu chỉ sinh `crack`: pothole và manhole có F1 bằng 0. Vòng train đã được chỉnh để lấy mẫu cân bằng ba lớp và tích luỹ gradient 4 micro-batch; kết quả mới cần chạy lại và kiểm chứng bằng test.

## Lưu ý nộp bài

- Không đưa `data/`, `work/`, `.env` hoặc weights vào Git.
- Không sửa `scripts/` và `tests/` của đề.
- Không dùng các file trong `report/progress/` thay cho evidence cuối nếu chúng không được sinh bởi `scripts/score.py`.
- Chỉ nộp khi checker trả về `0 FAIL, 0 WARN` và REPORT dùng số liệu thật của các run cuối.
