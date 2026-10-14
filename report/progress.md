# Lab Day 10 — tiến độ tạm thời

Đây là nhật ký kết quả đã chạy trên Kaggle, chưa phải bộ evidence cuối để nộp.

| Run | F1 | CI95 | Parse valid | Ghi chú |
|---|---:|---:|---:|---|
| E0 | 0.073 | 0.051–0.097 | 0.862 | Baseline zero-shot, đủ 311 ảnh |
| E1 | 0.0588 | 0.0388–0.0779 | 0.9357 | LoRA LLM; bị collapse sang `crack`, cần retrain |

E1 đã train 300 optimizer steps với 33,030,144 tham số trainable, peak VRAM 3.422 GB và 6.24 phút. Kết quả E1 hiện thấp hơn E0 nên chưa đạt tiêu chí rubric; chưa chạy/ghi nhận E2, E3, E4.

Các file JSON trong `report/progress/` là bản lưu tạm để không mất số liệu. Khi có output thật từ `scripts/score.py`, chép chúng sang `report/evidence/` rồi mới hoàn thiện `REPORT.md`.
