# Decision record

- Shape chọn: workflow
- Commit hash: 24ba8c247b9c97bca67e1f3fb21e870eaaa744df
- Run id dẫn chứng: R-workflow-C-007-s11 (thiếu approval → denied, có policy_denied), R-workflow-C-012-s11 (crash → không trả 2 lần), R-workflow-C-014-s11 (quote 150000, host trả phần chênh 120000) — trong src/reports/benchmark.json
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000000 (public 14 claim × 5 seed = 70 trial, src/reports/benchmark.json)
- Cost per success (mô phỏng, VND): 28.87 (single 32.71, orchestrator 46.43, swarm 49.57; mean steps 4.43, p95 5)
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 / 0 trên 70 trial public (mọi trial workflow qua trial_pass, gồm forbid_substrings và approval_ok)


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

Nhóm chọn Workflow. Trên cùng corpus public (14 claim) và cùng 5 seed (11, 22, 33, 44, 55), Workflow, Single và Orchestrator đều đạt 70/70 trial (pass rate 1.000, pass^5 1.000, variance 0), còn Swarm đạt 0.986 vì trượt C-002 ở seed 55 khi tin số ngày trễ trong ticket. Khi độ đúng ngang nhau, Workflow thắng ở các chỉ số còn lại: chi phí mô phỏng thấp nhất (28.87 VND/success, so với 32.71 của Single và 46.43 của Orchestrator), ít bước nhất (mean 4.43, p95 5) và chỉ gọi model đúng 1 lần để soạn câu trả lời. Mọi quyết định tiền đều nằm trong code: đọc claim qua `get_claim_view`, quote bằng `compute_compensation` từ dòng order, rồi đưa đủ quote cho `durability.issue_refund` để host kiểm approval, trần theo đơn và idempotency. Model không thấy ticket và không chọn số tiền, nên kết quả không phụ thuộc seed. Tự kiểm trên `student_drop/` (40 claim × 5 seed) cũng cho 0 lỗi, 0 nghi lộ secret, status giống hệt hai shape kia và chi phí thấp nhất.

## Có đáng đổi lấy multi-agent không

Hiện tại thì không. Orchestrator không tăng pass rate (cả hai đều 1.000 trên public, và cùng phân bố status trên hidden) nhưng tốn hơn khoảng 61% chi phí mô phỏng (46.43 so với 28.87 VND/success) và thêm khoảng 1 bước mỗi trial. Lợi ích chính của Orchestrator là tách investigator chỉ có quyền đọc khỏi quyền refund. Workflow đã đạt ranh giới này theo cách khác: chỉ có một đường code duy nhất gọi `issue_refund`, và host là bên kiểm approval và trần theo đơn, nên model không có đường nào chạm tới side effect. Nhóm sẽ đổi sang Orchestrator nếu (1) claim thật cần điều tra nhiều nguồn hoặc nhiều bước không cố định mà một luồng tuyến tính không mô tả được, (2) investigator cần dùng model thật để suy luận và pass rate của Orchestrator vượt Workflow trên cùng corpus và seed, hoặc (3) yêu cầu kiểm toán bắt buộc tách vai đọc và vai ghi thành các agent riêng. Nếu pass chỉ nhỉnh hơn trong khi cost và variance cao hơn, nhóm vẫn giữ Workflow.

## Trace và bằng chứng

Trace của Workflow theo Trace contract v1 (`{"run_id", "events"}`) và được ghi ở mọi đường return, kể cả return sớm và khi crash. Các event: `claim` (found, has_order_id), `lookup_order` (visible, reason; khi thấy được thì order_id, status, delay_days, shipping_fee_vnd, damage_verified), `quote` (amount_vnd, reason, clause_ids), `issue_refund` (ok, reason, conflict, amount_vnd), `crash` (at) và `decision` (status, proposal_amount, reason). Trace cố ý **không** ghi nội dung ticket (là dữ liệu người dùng, có thể chứa injection hoặc số liệu sai), `parent_secret`, `raw_headers`, `approval_id`, và các field order nhạy cảm như số điện thoại, địa chỉ, tên hàng, internal note. Lý do là các field này không cần cho việc kiểm toán quyết định tiền, và bỏ chúng đi thì không có gì để lộ. Bằng chứng không lộ secret không chỉ dựa vào scanner: `trial_pass` quét kết quả cùng mọi file trace của run id theo `forbid_substrings` trên 70 trial public; `tests/test_team_workflow.py` kiểm trực tiếp rằng trace không chứa ticket hay `parent_secret` và đơn ngoài phạm vi không lộ PII; `tests/test_team_security_audit.py` kiểm cả 3 shape với canary secret và approval sai hoặc mượn của claim khác. Các kết quả này chỉ chứng minh những trường hợp đã kiểm, không phải bảo đảm cho mọi đầu vào.

## Giới hạn

MockLLM không phải model sản xuất, nên benchmark chỉ đo chi phí của vòng điều phối chứ không đo được lợi ích mà một model thật có thể mang lại cho Single hay Orchestrator. Latency để `unavailable`. Chi phí là mô phỏng theo `config/pricing.yaml`, không phải giá API thật; tool call (10 VND) chiếm phần lớn chi phí. Kết quả public chỉ có 14 claim; kết quả `student_drop/` không có nhãn nên không ra pass rate, pass rate hidden chính thức do grader của giảng viên chấm. Crash safety 20/20 chỉ phủ các điểm crash mà `crash_test` định nghĩa. Không sửa JSON cho đủ 0.75.
