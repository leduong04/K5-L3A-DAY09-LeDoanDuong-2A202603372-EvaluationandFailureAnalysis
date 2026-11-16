# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: workflow
- Commit hash: 24ba8c247b9c97bca67e1f3fb21e870eaaa744df
- Run id dẫn chứng: R-workflow-C-007-s11 (thiếu approval → denied, có policy_denied), R-workflow-C-012-s11 (crash → không trả 2 lần), R-workflow-C-014-s11 (quote 150000, host trả phần chênh 120000) — trong src/reports/benchmark.json
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000000 (public 14 claim × 5 seed = 70 trial, src/reports/benchmark.json)
- Cost per success (mô phỏng, VND): 28.87 (single 32.71, orchestrator 46.43, swarm 49.57; mean steps 4.43, p95 5)
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 / 0 trên 70 trial public (mọi trial workflow qua trial_pass, gồm forbid_substrings và approval_ok)

Ví dụ một dòng đã điền: `- Shape chọn: workflow`, `- Crash safety: 20/20 (src/reports/crash.json)`.

## Tự kiểm hidden (student_drop)

Chạy trong `src/` ở commit 24ba8c2, kết quả ở `src/reports/hidden_selfcheck.json`:

`python -m lab8.benchmark --shapes workflow,single,orchestrator --cases student_drop/hidden_claims.jsonl --world student_drop/hidden_world_state.json --out reports/hidden_selfcheck.json`

| Shape | Trial | Lỗi (`error`) | `leak_suspect` | Mean / p95 steps | Mean model calls | Cost tổng (mô phỏng, VND) |
| --- | --- | --- | --- | --- | --- | --- |
| workflow | 200 | 0 | 0 | 4.53 / 5 | 1.00 | 6002.6 |
| single | 200 | 0 | 0 | 5.53 / 6 | 3.98 | 6743.1 |
| orchestrator | 200 | 0 | 0 | 5.50 / 6 | 0.00 | 9700.0 |

- Phân bố status của cả 3 shape: 90 `resolved`, 95 `denied`, 10 `escalated`, 5 `needs_info`.
- Cả 40 claim đều cho cùng một status ở cả 3 shape và cả 5 seed: không có variance theo seed, các shape không lệch nhau.
- Bộ này không có nhãn nên không tính được pass rate. Pass rate hidden chính thức do grader của giảng viên chấm. Kết quả trên chỉ cho thấy workflow chạy hết 200 trial không lỗi, trace đúng contract, không nghi lộ secret, và vẫn rẻ nhất trong 3 shape.

## Vì sao shape này

Viết 1 đoạn. So cùng corpus và cùng seed, không so với một lần chạy lẻ.

## Có đáng đổi lấy multi-agent không

Nếu pass nhỉnh hơn mà cost và variance cao hơn, nhóm giữ hay đổi? Điều kiện nào làm quyết định đảo lại?

## Trace và bằng chứng

Trace của nhóm ghi những event nào? Field nào cố ý **không** ghi, và vì sao? Nếu trace nằm ngoài chỗ scanner quét được, đội security chứng minh thế nào là hệ thống không lộ secret?

## Giới hạn

MockLLM không phải model sản xuất. Latency để `unavailable`. Không sửa JSON cho đủ 0.75.
