# Lab Day 10 — tiến độ tạm thời

Nhật ký kết quả đã chạy trên Kaggle -> chưa phải bộ evidence cuối

| Run | F1 | CI95 | Parse valid | Ghi chú |
|---|---:|---:|---:|---|
| E0 | 0.073 | 0.051–0.097 | 0.862 | Baseline zero-shot, đủ 311 ảnh |
| E1 | 0.2016 | 0.1679–0.2387 | 0.9389 | LoRA LLM; run mới đã vượt E0 +0.1286 |

E1 đã train 300 optimizer steps với 33,030,144 tham số trainable, peak VRAM 5.242 GB và 25.32 phút. Run mới đạt F1 0.202, cao hơn E0 0.073 +0.129; chưa chạy/ghi nhận E2, E3, E4.

Các file JSON trong `report/progress/` là bản lưu tạm để không mất số liệu. Khi có output thật từ `scripts/score.py`, chép chúng sang `report/evidence/` rồi mới hoàn thiện `REPORT.md`.
