# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: Gehihi (lớp L3A)
- Thành viên: xem `TEAM.md` — Lý Thành Lập ( workflow), Nguyễn Tiến Cường (single), Lê Đoàn Dương (orchestrator, security)
- Repo: https://github.com/lythanhlap/K5-L3A-DAY08-Gehihi-FromWorkflowsToAgents, nhánh `main`
- Commit chốt (trùng `report/team/decision_adr.md`): `24ba8c247b9c97bca67e1f3fb21e870eaaa744df`
- Shape chọn: `workflow`

## 2. Mục tiêu và cách tiếp cận

**Bài toán.** Xử lý claim hoàn phí vận chuyển của LogiX-VN bằng bốn kiến trúc có cùng chữ ký `run_claim(claim_id, *, seed, store_path, crash_at=None)`, rồi dựa trên số đo để chọn kiến trúc **ít tự chủ nhất mà vẫn làm được việc**. Nhóm hiểu bài toán qua ba nguyên tắc rút ra từ `docs/business_rules.md` và `docs/contracts.md`:

1. **Tiền do code quyết.** Số tiền đến từ `compute_compensation` trên dòng order và policy hiện hành (`is_current`, `effective_date <= 2026-11-15`). Model không chọn số tiền, không ghi ledger.
2. **Host là cửa ghi duy nhất.** Mọi shape đưa **quote đầy đủ** vào `durability.issue_refund`; host kiểm approval (đúng claim, còn hạn, chưa thu hồi), trừ phần đã hoàn của cùng đơn và chống ghi trùng theo khóa `refund:{claim_id}`.
3. **Ticket là dữ liệu, trace là ranh giới bảo mật.** Không lấy số ngày trễ hay mã đơn từ chữ trong ticket; trace không chứa ticket, `parent_secret`, SĐT, địa chỉ, tên hàng, ghi chú nội bộ.

**Chia việc.** Mỗi người một shape trên nhánh riêng, nên ba file shape không đụng nhau và merge qua PR không bị conflict code:

| Người | Shape / phần việc | Nhánh | Báo cáo |
| --- | --- | --- | --- |
| Lý Thành Lập | A `workflow`, integration, benchmark, crash suite, tự kiểm hidden | `lythanhlap` | `decision_adr.md` |
| Nguyễn Tiến Cường | B `single`, ablation workflow vs single | `feat/single` | `research_card.md` (R4) |
| Lê Đoàn Dương | C `orchestrator`, security audit (approval, secret/PII) | `LeDoanDuong` | `group_report.md` |

**Thứ tự làm.** (1) CP1: cả nhóm cài môi trường, chạy `python -m lab8.smoke` và cổng host, đọc business rules, contracts và `swarm_reference.py`. (2) CP2: viết workflow trước vì đơn giản nhất và làm khuôn chung (mở store → đọc claim → kiểm `order_id` → `lookup_order` → quote → `issue_refund` → `finish`); single và orchestrator dựng lại cùng khung, chỉ thay phần thu thập dữ kiện. (3) CP3: truyền `crash_at`, không nuốt `InjectedCrash`. (4) CP4: crash suite và rà approval/secret trên cả ba shape. (5) CP5: benchmark 4 shape × 5 seed, tự kiểm trên `student_drop/`, viết ADR, research card và báo cáo. Swarm (D) có sẵn, chỉ dùng để đối chứng.

## 3. Giải pháp

### Khung chung của ba shape

Cả ba shape đều: mở store bằng `lab8.db.connect`; tạo `Budget` và `Trace(make_run_id(shape, claim_id, seed))`; đọc claim qua `tools.get_claim_view` (chỉ `claim_id, principal_id, order_id, ticket`, không `SELECT *`); trả kết quả qua một hàm `done()` nội bộ gọi `common.finish(...)`, nên **mọi** nhánh return đều ghi trace và `refund_effects` / `committed_audit_count` lấy từ ledger chứ không tự điền. Ánh xạ trạng thái giống nhau:

| Điều kiện | `status` | `proposal_amount` | Gọi host? |
| --- | --- | --- | --- |
| Claim không tồn tại | `failed` | 0 | Không |
| Thiếu `order_id` | `needs_info` | 0 | Không |
| Đơn không tìm thấy / không thuộc principal | `escalated` | 0 | Không |
| Quote = 0 | `denied` | 0 | Không |
| Quote > 0, host từ chối | `denied` | quote đầy đủ | Có (host ghi `policy_denied`) |
| Quote > 0, host báo `conflict` | `failed` | quote đầy đủ | Có |
| Quote > 0, host chấp nhận | `resolved` | quote đầy đủ | Có (host cap, ghi ledger + audit) |

Lời gọi ghi tiền giống hệt nhau ở ba shape:

```python
try:
    outcome = durability.issue_refund(conn, claim_id, proposal, crash_at=crash_at)
except durability.InjectedCrash:
    trace.add("crash", {"at": crash_at})
    trace.write()
    raise   # không nuốt crash: T12 cần thấy crash thật
```

### A — Workflow (`src/lab8/shapes/workflow.py`)

Chuỗi bước Python cố định: `get_claim_view` → `lookup_order(order_id, principal_id)` → `compute_compensation(order)` → `issue_refund`. Ticket không được parse, không đưa vào trace, không đưa cho model. MockLLM được gọi **đúng một lần** với prompt `status=… proposal=…` để soạn câu trả lời. Vì không có nhánh nào phụ thuộc seed, workflow không bị điểm yếu của swarm ở seed 55.

### B — Single ReAct (`src/lab8/shapes/single.py`)

Một agent chạy bằng `lab8.runtime.run_react` với `max_steps=8` và `worker=True`, nên runtime chặn mọi tool không phải tool đọc. Mỗi bước tốn một lần gọi MockLLM, sau đó agent chọn tool đọc tiếp theo (claim, rồi đơn) hoặc dừng. Ranh giới code/model:

- **Code gắn tham số tool** (`claim_id`, `order_id`, `principal_id`) từ claim được giao và claim view; model không viết id nên không thể tra đơn của khách khác.
- **Code quyết sau vòng lặp**: status từ dữ kiện đã đọc, quote bằng `compute_compensation`, quote đầy đủ vào `issue_refund`. Agent dừng mà chưa đọc được đơn thì `failed`, không refund.
- Dùng cùng 3 tool đọc và cùng prompt soạn trả lời như workflow, để phép so A–B chỉ khác ở vòng ReAct (xem research card R4).

### C — Orchestrator (`src/lab8/shapes/orchestrator.py`)

Orchestrator đọc claim, kiểm `order_id`, rồi dựng handoff bằng `lab8.handoff.make_handoff`:

```python
make_handoff(request_id=f"REQ-{claim_id}", claim_id=claim_id,
             task="verify_order_and_calculate_compensation", facts={},
             allowed_tools=["lookup_order", "lookup_policy", "compute_compensation"])
```

`facts` để trống: investigator không nhận ticket, `parent_secret`, header hay claim row. `run_readonly_investigators` chạy các tool đọc trong phạm vi `principal_id`; investigator không có registry ghi. Orchestrator đọc order view và quote mà investigator trả về, rồi **chính orchestrator** gọi `issue_refund`. Trace ghi `safe_dump(handoff)` (bản đã qua allowlist), không ghi parent context — tóm tắt bằng LLM không được dùng thay cho redaction.

### Ranh giới code / model

| Việc | Ai quyết |
| --- | --- |
| Lọc theo principal, policy hiện hành, số tiền | Code (`lookup_order`, `compute_compensation`) |
| Approval, trần theo đơn, idempotency, ghi ledger | Host (`durability.issue_refund`, một transaction) |
| Chọn tool đọc tiếp theo (chỉ ở B) | Model, trong giới hạn 8 bước và tool đọc |
| Soạn câu trả lời khách | Model (MockLLM), chỉ thấy status và số tiền |
| Retry sau crash | Ledger (khóa `refund:{claim_id}`), không phải checkpoint đơn lẻ |

### Trace

Trace theo contract v1 (`{"run_id", "steps", "events"}`, file `traces/<run_id>.json`), ghi ở mọi nhánh kể cả return sớm và crash. Event của workflow: `claim` (found, has_order_id), `lookup_order` (visible, reason; khi thấy được: order_id, status, delay_days, shipping_fee_vnd, damage_verified), `quote` (amount_vnd, reason, clause_ids), `issue_refund` (ok, reason, conflict, amount_vnd), `crash` (at), `decision` (status, proposal_amount, reason). Single thêm event cho từng bước ReAct (số bước, tên tool, tóm tắt kết quả đã allowlist); orchestrator thêm `handoff` và `investigator`. Cố ý **không** ghi: nội dung ticket, `parent_secret`, raw header, `approval_id`, SĐT, địa chỉ, tên hàng, internal note.

## 4. Kết quả

### Benchmark public (`src/reports/benchmark.json`)

Điều kiện kiểm soát (ghi trong file): 14 claim `C-001`…`C-014`, seed `11, 22, 33, 44, 55`, `as_of` 2026-11-15, MockLLM, giá mô phỏng (`tool_call_vnd` 10, input 20 / output 80 VND mỗi 1.000 token), SQLite mới cho mỗi trial, latency `unavailable`. Mỗi shape 70 trial.

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean / p95 steps | Model calls / trial | Tool calls / trial |
| --- | ---: | ---: | --- | ---: | ---: | --- | ---: | ---: |
| A workflow | **1.000** (70/70) | **1.000** | 1.000–1.000 | 0 | **28.87** | 4.43 / 5 | 1.00 | 2.79 |
| B single | 1.000 (70/70) | 1.000 | 1.000–1.000 | 0 | 32.71 | 5.43 / 6 | 3.93 | 2.79 |
| C orchestrator | 1.000 (70/70) | 1.000 | 1.000–1.000 | 0 | 46.43 | 5.36 / 6 | 0.00 | 4.64 |
| D swarm | 0.986 (69/70) | 0.929 | 0.929–1.000 | 0.000816 | 49.57 | 2.80 / 3 | 3.00 | 4.66 |

Nhận xét:

- Ba shape nhóm viết đều đạt 70/70, variance 0. Swarm trượt đúng một trial: `C-002` (T02) ở seed 55 ra `resolved` thay vì `denied`, vì ở seed chia hết cho 5 swarm tin số "trễ 5 ngày" trong ticket thay cho `delay_days = 2` của đơn. Ba shape còn lại ra `denied` ở mọi seed (ví dụ `R-workflow-C-002-s55`, `R-single-C-002-s55`).
- Khi độ đúng ngang nhau, khác biệt nằm ở chi phí: single đắt hơn workflow 13% (thêm ~2.9 lần gọi model mỗi trial, cùng số tool call); orchestrator đắt hơn 61% vì investigator gọi thêm `lookup_policy` và tool call (10 VND) chiếm phần lớn chi phí.
- `steps` là số event trong trace nên phụ thuộc cách mỗi shape ghi trace; swarm ghi ít event hơn nên steps thấp dù tốn nhiều lời gọi hơn.

### Durability (`src/reports/crash.json`)

`crash_safety` = **20/20**: 5 điểm `before_effect`, 7 điểm `after_effect_before_checkpoint`, 5 điểm `after_checkpoint`, cộng `amount_conflict_on_retry`, `retry_without_approval` và `swarm_after_effect_before_checkpoint`. Sau mỗi lần resume còn đúng một dòng `refund_ledger` và một audit `refund_committed`. T11 (gọi lại cùng claim) và T12 (crash rồi resume) xanh trên cả ba shape.

### Policy và bảo mật

- Ví dụ trên workflow: `R-workflow-C-007-s11` thiếu approval → `denied`, có audit `policy_denied`; `R-workflow-C-014-s11` quote 150.000, host chỉ ghi phần chênh 120.000 (đơn đã hoàn 30.000 ship trước đó); T13 (đơn đã hoàn đủ) → `denied`, `proposal_amount` 35.000, không có dòng ledger.
- Secret leak / approval bypass: **0 / 0** trên 70 trial public của workflow (`trial_pass` quét kết quả và mọi file `<run_id>*` theo danh sách chuỗi cấm, và đối chiếu approval với ledger).
- `tests/test_team_security_audit.py` (9 test) kiểm cả ba shape với canary secret, approval hỏng và approval mượn của claim khác; `tests/test_team_orchestrator.py` kiểm handoff chỉ có field allowlist và investigator chỉ có tool đọc.

### Tự kiểm hidden (`src/reports/hidden_selfcheck.json`)

40 claim × 5 seed = 200 trial mỗi shape trên `student_drop/` (không có nhãn nên không có pass rate):

| Shape | Lỗi (`error`) | `leak_suspect` | Mean / p95 steps | Cost tổng (VND mô phỏng) |
| --- | ---: | ---: | --- | ---: |
| workflow | 0 | 0 | 4.53 / 5 | 6002.6 |
| single | 0 | 0 | 5.53 / 6 | 6743.1 |
| orchestrator | 0 | 0 | 5.50 / 6 | 9700.0 |

Cả ba shape cho cùng phân bố status (90 `resolved`, 95 `denied`, 10 `escalated`, 5 `needs_info`) và cùng status trên từng claim ở cả 5 seed. Pass rate hidden chính thức do grader của giảng viên chấm.

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md` (chọn **workflow**: đúng ngang hai shape kia, rẻ nhất, ít bước nhất, không phụ thuộc seed). Phiếu nghiên cứu và ablation: xem `report/team/research_card.md` (R4: workflow vs single cùng status trên 70/70 cặp claim–seed và cùng số tool call, nên chênh 13% chi phí đến từ vòng ReAct).

## 5. Khó khăn và cách xử lý

| Khó khăn | Biểu hiện | Cách xử lý |
| --- | --- | --- |
| Ghi trace ở **mọi** nhánh return | Return sớm (`needs_info`, `escalated`) hoặc crash dễ quên ghi trace → grader fail-closed với `missing_trace` / `stale_trace` | Mọi nhánh đi qua một hàm `done()` gọi `common.finish()`; nhánh crash ghi event `crash` rồi `trace.write()` trước khi `raise`. Test `test_crash_still_flushes_trace` khóa hành vi này |
| Bẫy trừ hai lần ở T14 | Nếu shape tự trừ phần đã hoàn rồi mới gửi host, host trừ thêm lần nữa và T14 fail | Truyền quote đầy đủ; dùng `lookup_refund_history` chỉ để đọc. Test `test_t14_passes_full_quote_and_host_pays_difference` |
| `InjectedCrash` bị nuốt | `except Exception` chung sẽ làm T12 không thấy crash | Chỉ bắt đúng `InjectedCrash`, ghi trace rồi `raise` lại; không có `except` rộng quanh `issue_refund` |
| Rò dữ liệu qua trace khi debug | Ghi nguyên claim row / order row sẽ lộ `parent_secret`, SĐT, địa chỉ | Chỉ ghi field đã chọn từ `lookup_order` (vốn đã allowlist); handoff để `facts={}`; thêm test quét trace bằng canary secret |
| Nhóm bất đồng về shape chọn | Trong lúc làm, một bản ADR đề xuất orchestrator (vì tách quyền đọc/ghi), bản kia đề xuất workflow; merge `origin/main` vào `lythanhlap` bị conflict ở `decision_adr.md` (commit `358ac5a`) | Cả nhóm đối chiếu lại `benchmark.json`: pass rate ngang nhau, orchestrator đắt hơn 61%, còn workflow đã đạt tách quyền bằng cách chỉ có một đường code gọi `issue_refund`. Chốt workflow; điều kiện đổi sang orchestrator ghi trong ADR |
| Số liệu lệch giữa các bản báo cáo | Một bảng benchmark ghi trong báo cáo cá nhân có single 0/70, không khớp `benchmark.json` hiện tại (single 70/70) | Lấy `src/reports/benchmark.json` sinh trên `main` sau khi merge đủ ba shape (commit `24ba8c2`) làm nguồn duy nhất; chạy lại lần hai cho số giống hệt (chỉ khác ký tự xuống dòng) |
| Môi trường Windows | Gọi `python -m pytest` bằng Python hệ thống báo `No module named 'pytest'` vì chưa activate venv | Activate venv trong `src/`, hoặc gọi thẳng `.venv\Scripts\python.exe -m pytest -q` |

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`:

- Lý Thành Lập — workflow (`6234dbe`), `tests/test_team_workflow.py` (`0226eb1`), benchmark + crash (`24ba8c2`), ADR + tự kiểm hidden (`defe1b2`), merge PR.
- Nguyễn Tiến Cường — single + `tests/test_team_single.py` (`bcea95e`), research card R4 + ablation (`bd247b5`).
- Lê Đoàn Dương — orchestrator (`c4f47bd`), `tests/test_team_orchestrator.py` (`44b0572`), `tests/test_team_security_audit.py` (`2e75657`), báo cáo nhóm.

## 7. Hạn chế và hướng cải thiện

- **MockLLM không phải model sản xuất.** Nó không suy luận, nên benchmark chỉ đo chi phí của vòng điều phối, không đo được lợi ích mà một model thật có thể đem lại cho single hay orchestrator (ví dụ hiểu ticket mơ hồ để hỏi lại khách). Latency là `unavailable`.
- **Chi phí là mô phỏng** theo `config/pricing.yaml`; tool call chiếm phần lớn. Nếu giá model tương đối cao hơn, chênh lệch giữa workflow và single sẽ lớn hơn 13%.
- **Corpus nhỏ.** Bộ public chỉ có 14 claim; bộ `student_drop/` không có nhãn nên chỉ chứng minh "chạy không lỗi, không nghi lộ", chưa chứng minh đúng.
- **Phạm vi kiểm bảo mật.** Test và scanner chỉ chứng minh các trường hợp đã kiểm (canary, chuỗi cấm đã biết), không phải bảo đảm cho mọi đầu vào. Crash 20/20 chỉ phủ các điểm crash mà `crash_test` định nghĩa.
- **Điều kiện làm kết luận đổi:** (1) claim thật cần điều tra nhiều nguồn, nhiều bước không cố định mà luồng tuyến tính không mô tả được; (2) dùng model thật và pass rate của single/orchestrator vượt workflow trên cùng corpus và seed; (3) yêu cầu kiểm toán bắt buộc tách vai đọc và vai ghi thành agent riêng. Nếu pass chỉ nhỉnh hơn trong khi cost và variance cao hơn, nhóm vẫn giữ workflow.
- **Hướng cải thiện:** thêm test property-based cho trần theo đơn (nhiều claim trên một đơn), mở rộng scanner trace sang dạng homoglyph/biến thể của PII, và chạy ablation orchestrator có/không `lookup_policy` để tách phần chi phí do tool thừa.

## 8. Tài liệu tham khảo

- Tài liệu của lab: `README.md`, `docs/business_rules.md`, `docs/contracts.md`, `docs/architecture.md`, `docs/metrics.md`, `src/lab8/shapes/swarm_reference.py`.
- Zhang và cộng sự, *Rethinking the Evaluation of Efficiency Methods for Multi-Agent Systems*, EMNLP 2026 (main), https://aclanthology.org/2026.emnlp-main.798/ — nguồn của research card R4.
- Google Research 2026, *Towards a Science of Scaling Agent Systems*; MAST, *Why Do Multi-Agent LLM Systems Fail?*, NeurIPS 2025; AgentArch, AACL 2026 — thư viện tham khảo của buổi lý thuyết (slide 68–69), dùng để đặt câu hỏi R1–R3.
- Công cụ AI: ChatGPT và Claude Code dùng để đọc hiểu contract, gợi ý test và tóm tắt số liệu từ file JSON; mọi con số trong báo cáo đối chiếu lại với `src/reports/*.json`. Chi tiết từng người ghi trong báo cáo cá nhân.

## Phụ lục — lệnh tái lập (chạy trong `src/`)

```text
python -m pytest -q
python -m pytest -q -m student -o addopts=
python -m lab8.crash_test --count 20 --out reports/crash.json
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
python -m lab8.benchmark --shapes workflow,single,orchestrator --cases student_drop/hidden_claims.jsonl --world student_drop/hidden_world_state.json --out reports/hidden_selfcheck.json
python -m lab8.check_submission
```
