# Checklist nộp Lab Day 10

## Trạng thái hiện tại — 2026-10-14

- [x] `REPORT.md` có họ tên, dataset version, bảng kết quả E0/E1 và 10 câu trả lời nháp.
- [ ] `report/evidence/e0.json` đến `e3.json` được sinh trực tiếp bởi `scripts/score.py`.
- [ ] Có ít nhất hai file `report/evidence/e4_*.json`.
- [ ] Có tối thiểu 6 ảnh minh hoạ prediction/ground truth.
- [x] Không track `data/`, `work/`, `.env` hoặc model weights.
- [ ] Có tối thiểu 5 commits (hiện tại: 4).
- [ ] `python scripts/check_submission.py` trả về 0 FAIL, 0 WARN.

## Kết quả đã có

- E0: F1 0.073, CI95 0.051–0.097.
- E1: F1 0.202, CI95 0.168–0.239; cao hơn E0 0.129 điểm.
- Bản lưu tạm: `report/progress/`.

## Trước khi nộp

1. Đã retrain E1 từ model gốc sạch; cần chép evidence chính thức từ Kaggle vào repo.
2. Chạy E2, E3 và E4 trên test, lưu evidence bằng `scripts/score.py`.
3. Xuất ít nhất 6 ảnh minh hoạ và cập nhật Câu 9 trong `REPORT.md`.
4. Chạy lại checker, sau đó mới push GitHub.
