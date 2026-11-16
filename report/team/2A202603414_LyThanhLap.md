# Báo cáo cá nhân — 2A202603414_LyThanhLap

## 1. Thông tin thành viên

- Họ tên: Lý Thành Lập
- MSSV: 2A202603414
- Email: lylap090@gmail.com
- GitHub: [lythanhlap](https://github.com/lythanhlap)
- Vai trong nhóm: Người 1 — workflow + integration, bằng chứng benchmark và decision record

## 2. Phần việc phụ trách

- Shape A `workflow` (shape nhóm chọn để chấm hidden): [src/lab8/shapes/workflow.py](../../src/lab8/shapes/workflow.py).
- Test riêng cho shape workflow: [src/tests/test_team_workflow.py](../../src/tests/test_team_workflow.py).
- Integration: repo nhóm, merge nhánh `lythanhlap` vào `main` (PR #3, #5) và gỡ conflict khi merge `origin/main`.
- Bằng chứng nộp bài: [src/reports/benchmark.json](../../src/reports/benchmark.json), [src/reports/crash.json](../../src/reports/crash.json), tự kiểm hidden [src/reports/hidden_selfcheck.json](../../src/reports/hidden_selfcheck.json).
- Decision record: [report/team/decision_adr.md](decision_adr.md).

## 3. Đóng góp cụ thể

| Commit | Nội dung |
| --- | --- |
| `6234dbe` | Viết shape workflow |
| `0226eb1` | Viết `tests/test_team_workflow.py` (88 test) |
| `24ba8c2` | Chạy lại benchmark public 4 shape × 14 claim × 5 seed và crash suite 20/20 |
| `defe1b2` | Điền decision record, thêm kết quả tự kiểm trên `student_drop/` |
| `358ac5a` | Merge `origin/main`, gỡ conflict `decision_adr.md`, chốt shape workflow |

Quyết định kỹ thuật trong shape workflow:

1. Đọc claim qua `get_claim_view` (allowlist field), không `SELECT *`. Ticket là dữ liệu người dùng: không parse, không ghi vào trace, không đưa cho model.
2. Thiếu `order_id` thì trả `needs_info` ngay, không đoán mã đơn từ ticket. Đơn ngoài phạm vi principal thì trả `escalated`, không chép SĐT, địa chỉ, tên hàng hay ghi chú nội bộ.
3. Số tiền tính trong code bằng `compute_compensation` từ dòng order, bỏ qua số ngày trễ ghi trong ticket. Quote bằng 0 thì `denied` và không gọi host.
4. Truyền **quote đầy đủ** và `crash_at` vào `durability.issue_refund`; host tự kiểm approval, trừ phần đã hoàn cho cùng đơn (T13, T14) và idempotency. Không tự trừ để tránh trừ hai lần. Không bắt `InjectedCrash`, chỉ ghi event `crash` rồi flush trace trước khi ném tiếp.
5. MockLLM chỉ được gọi một lần để soạn câu trả lời từ `status` và `proposal_amount`; model không chọn số tiền và không commit refund.
6. Trace theo contract v1, ghi ở mọi đường return: `claim`, `lookup_order`, `quote`, `issue_refund`, `crash`, `decision`.

Test đã thêm trong `src/tests/test_team_workflow.py` (88 test, chạy trong `python -m pytest -q` mặc định):

- `test_public_cases_pass_on_every_seed`: T01–T10, T13, T14 × 5 seed, chấm bằng `trial_pass`.
- `test_workflow_is_deterministic_across_seeds`: cùng claim cho cùng status, proposal và reason ở mọi seed.
- `test_t11_retry_is_idempotent`, `test_t12_crash_propagates_and_rerun_does_not_double_pay`, `test_crash_still_flushes_trace`: chạy lại và crash không tạo refund thứ hai.
- Từng nhánh: `test_unknown_claim_fails_without_ledger`, `test_missing_order_id_stops_before_lookup`, `test_order_outside_scope_is_escalated_without_leaking_pii`, `test_zero_quote_denied_without_host_call`, `test_policy_denied_keeps_full_quote`, `test_t14_passes_full_quote_and_host_pays_difference`.
- `test_trace_contract_and_no_ticket_text`, `test_model_only_drafts_once`: trace đúng contract, không chứa ticket hay `parent_secret`, model chỉ gọi một lần.

## 4. Bằng chứng

- Benchmark public (`src/reports/benchmark.json`): workflow pass rate 1.000, pass^5 1.000, seed min–max 1.000–1.000, variance 0, cost per success 28.87 VND (thấp nhất trong 4 shape), mean steps 4.43, p95 5.
- Run id: `R-workflow-C-007-s11` (thiếu approval → `denied`, có `policy_denied`), `R-workflow-C-012-s11` (crash → không trả hai lần), `R-workflow-C-014-s11` (quote 150000, host trả phần chênh 120000).
- Crash safety 20/20 (`src/reports/crash.json`).
- Tự kiểm hidden (`src/reports/hidden_selfcheck.json`): workflow chạy 200 trial, 0 lỗi, 0 `leak_suspect`, tổng chi phí mô phỏng 6002.6 VND (single 6743.1, orchestrator 9700.0).
- `python -m pytest -q` xanh; `python -m lab8.check_submission` không còn FAIL.

## 5. Công cụ AI đã dùng

- Claude Code (Claude Opus) dùng để:
  - Viết `tests/test_team_workflow.py` dựa trên `workflow.py` và các test có sẵn.
  - Chạy benchmark, crash suite và tự kiểm trên `student_drop/`, rồi tóm tắt số liệu từ file JSON.
  - Điền các dòng số liệu trong `decision_adr.md` và gỡ conflict khi merge `origin/main`.
- Cách tự kiểm tra:
  - Chạy `python -m pytest -q` sau mỗi thay đổi.
  - Chạy `python -m lab8.check_submission`.
  - Chạy lại benchmark và crash test ở commit `24ba8c2` và so với file đã commit: số liệu giống hệt, chỉ khác ký tự xuống dòng.
  - Đối chiếu từng con số và run id trong ADR với `benchmark.json`.
  - Kiểm tra `hidden_selfcheck.json` không chứa ticket, nhãn hay world state trước khi commit.
- Không dùng AI để sửa tay file JSON trong `src/reports/` hay để bịa số liệu.

## 6. Tự đánh giá

Kết quả: Workflow đạt 1.000 trên 70 trial public với variance 0. Trên 200 trial hidden, workflow có 0 lỗi và 0 leak_suspect. Crash safety đạt 20/20, chi phí mỗi lần thành công là 28,87 VND, thấp nhất trong 4 shape.
Khó khăn: Phần em thấy khó là bảo đảm workflow ghi đủ trace ở mọi nhánh trả về, kể cả khi có lỗi.
Nếu làm lại: Em sẽ cùng nhóm chốt sớm giữa workflow và orchestrator trước khi viết ADR để giảm công sửa tài liệu.
