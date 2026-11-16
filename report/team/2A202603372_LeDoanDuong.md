# Báo cáo cá nhân — 2A202603372 Le Doan Duong

## 1. Thông tin thành viên

- **Họ tên:** Lê Đoàn Dương
- **MSSV:** 2A202603372
- **Email:** leduong04st@gmail.com
- **GitHub:** [leduong04](https://github.com/leduong04)
- **Repository:** [K5-L3A-DAY08-Gehihi-FromWorkflowsToAgents](https://github.com/lythanhlap/K5-L3A-DAY08-Gehihi-FromWorkflowsToAgents)
- **Branch:** `LeDoanDuong`

## 2. Phần việc phụ trách

Tôi phụ trách **Person 3 – Orchestrator, Security và Group Report**.

Các nhiệm vụ chính:

- Tìm hiểu business rules, contracts, handoff và runtime để nắm luồng xử lý claim/refund.
- Xây dựng Orchestrator chịu trách nhiệm điều phối các investigator, tổng hợp kết quả và quyết định thời điểm thực hiện refund.
- Thiết kế handoff theo nguyên tắc allowlist, chỉ truyền các trường dữ liệu cần thiết cho tác vụ.
- Đảm bảo investigator chỉ có quyền thực hiện các thao tác đọc (read-only), không trực tiếp gọi các công cụ có side effect như `issue_refund` hoặc `append_audit`.
- Kiểm tra cơ chế xử lý lỗi và crash injection, bảo đảm lỗi `InjectedCrash` được truyền ra ngoài thay vì bị bắt và bỏ qua.
- Viết kiểm thử cho Orchestrator, handoff và security audit; kiểm tra nguy cơ rò rỉ secret, thông tin cá nhân và approval bypass.
- Tham gia hoàn thiện Decision ADR, bằng chứng benchmark, crash test và báo cáo nhóm.

## 3. Đóng góp cụ thể

### 3.1. Xây dựng Orchestrator

Triển khai luồng điều phối theo hướng tách biệt trách nhiệm:

1. Tiếp nhận thông tin claim.
2. Tạo handoff chỉ chứa những trường được cho phép.
3. Gọi các investigator với quyền read-only để thu thập thông tin.
4. Tổng hợp kết quả điều tra và áp dụng business rules.
5. Orchestrator chịu trách nhiệm quyết định và gọi `issue_refund` khi các điều kiện nghiệp vụ được đáp ứng.

Cách thiết kế này giúp giới hạn quyền của từng thành phần, tránh việc các investigator tự thực hiện thao tác hoàn tiền hoặc ghi audit ngoài phạm vi trách nhiệm.

### 3.2. Kiểm soát handoff và bảo mật

- Sử dụng allowlist để giới hạn dữ liệu được truyền giữa các thành phần.
- Kiểm tra việc loại bỏ secret, raw headers và các trường dữ liệu nhạy cảm không cần thiết.
- Kiểm tra quyền công cụ của investigator, bảo đảm không thể gọi trực tiếp các công cụ có side effect.
- Bổ sung kiểm thử phát hiện secret leakage trong trace và handoff.
- Kiểm tra điều kiện approval, bao gồm approval không hợp lệ hoặc approval thuộc claim khác, nhằm ngăn việc refund khi chưa đáp ứng điều kiện cho phép.

### 3.3. Kiểm thử và đánh giá

Các kiểm thử do tôi bổ sung và thực hiện:

- `tests/test_team_orchestrator.py`: 2 test kiểm tra handoff allowlist, quyền read-only và việc không để lộ dữ liệu nhạy cảm trong trace.
- `tests/test_team_security_audit.py`: 9 test phục vụ security audit.
- Nhóm kiểm thử authorization, approval binding, handoff secret, trace contract và orchestrator: 22 test đạt yêu cầu.

Kết quả kiểm thử được ghi nhận:

| Nội dung | Kết quả |
|---|---:|
| Team orchestrator tests | 2 passed |
| Team security audit tests | 9 passed |
| Authorization, approval, handoff, trace và orchestrator tests | 22 passed |
| Toàn bộ pytest | 96 passed, 248 deselected |
| Smoke test | PASS |
| Crash safety | 20/20 |

### 3.4. Phân tích benchmark và quyết định kiến trúc

Tôi tham gia phân tích kết quả benchmark với 14 claim, 5 seed và 70 lượt thử cho mỗi shape.

| Shape | Thành công | Pass rate | Cost/success (VND, mô phỏng) |
|---|---:|---:|---:|
| Workflow | 70/70 | 100% | 28.87 |
| Single | 0/70 | 0% | Không có |
| Orchestrator | 70/70 | 100% | 46.43 |
| Swarm | 69/70 | 98.57% | 49.57 |

Từ kết quả này, tôi tham gia đề xuất lựa chọn Orchestrator vì khả năng phân tách quyền hạn rõ ràng giữa investigator và thành phần thực hiện side effect. Workflow có chi phí thấp hơn và cùng đạt tỷ lệ thành công 100%, do đó việc chọn Orchestrator là sự đánh đổi giữa chi phí mô phỏng và khả năng kiểm soát quyền thực thi, không phải vì Orchestrator có pass rate cao hơn Workflow.

Crash test đạt 20/20 trong các kịch bản được định nghĩa, cho thấy cơ chế xử lý crash và retry hoạt động đúng trong phạm vi kiểm thử.

## 4. Bằng chứng

Các bằng chứng có thể kiểm tra trực tiếp trong repository:

- `src/lab8/orchestrator.py`: phần triển khai Orchestrator.
- `src/tests/test_team_orchestrator.py`: kiểm thử Orchestrator, handoff và trace.
- `src/tests/test_team_security_audit.py`: kiểm thử security audit.
- `reports/benchmark.json`: kết quả benchmark giữa các shape.
- `reports/crash.json`: kết quả crash safety.
- `report/team/decision_adr.md`: quyết định kiến trúc, lý do lựa chọn và bằng chứng.
- `TEAM.md`: bảng đóng góp của thành viên.

Các lệnh kiểm thử đã thực hiện:

```bash
python -m lab8.smoke
python -m pytest -q
python -m lab8.check_submission
```

Kết quả kiểm tra submission gần nhất:

- Ba shape workflow, single và orchestrator đều đạt kiểm tra format và 12/12 claim public ở seed 11.
- Benchmark hợp lệ: 4 shape × 14 claim × 5 seed.
- Crash report hợp lệ: 20/20.
- Decision ADR hợp lệ.
- Bảng đóng góp trong TEAM.md được checker nhận diện.

Commit code được checker ghi nhận trong quá trình hoàn thiện báo cáo: `2e75657`. Cần cập nhật hash cuối cùng nếu commit mới thay đổi nội dung được dùng làm bằng chứng.

## 5. Công cụ AI đã dùng

Tôi sử dụng ChatGPT như một công cụ hỗ trợ trong quá trình thực hiện:

- Phân tích yêu cầu và phân chia các nhiệm vụ trong phần Orchestrator.
- Hỗ trợ đọc hiểu luồng xử lý, contracts, handoff và runtime.
- Hỗ trợ xây dựng hướng kiểm thử cho quyền truy cập công cụ, secret leakage và approval binding.
- Hỗ trợ phân tích kết quả benchmark, crash test và diễn giải các đánh đổi kiến trúc.
- Hỗ trợ rà soát cấu trúc báo cáo và Decision ADR.

Tôi tự kiểm tra các đề xuất bằng cách triển khai và chạy trực tiếp trên repository, sử dụng pytest, smoke test, benchmark và `check_submission`. Các kết quả được ghi trong báo cáo dựa trên output thực tế của chương trình, không chỉ dựa trên nội dung do AI đề xuất.

## 6. Tự đánh giá

Qua Lab 8, tôi hiểu rõ hơn sự khác biệt giữa workflow và agent orchestration, đặc biệt là cách phân chia trách nhiệm và quyền hạn trong một hệ thống có nhiều thành phần AI.

Phần quan trọng nhất tôi học được là không nên trao toàn bộ quyền thực thi cho mọi agent. Một investigator có thể cần quyền đọc dữ liệu để phân tích claim, nhưng không nhất thiết phải có quyền thực hiện refund hoặc ghi audit. Việc sử dụng handoff allowlist và giới hạn tool permissions giúp giảm rủi ro khi hệ thống được mở rộng.

Tôi cũng hiểu rõ hơn tầm quan trọng của việc kiểm tra approval binding, kiểm soát dữ liệu nhạy cảm trong trace và xử lý crash/retry đối với các thao tác có side effect. Một hệ thống có kết quả đúng trên các test thông thường vẫn cần được kiểm tra trong những tình huống lỗi và retry để tránh thực hiện side effect sai hoặc nhiều lần.

Bên cạnh phần lập trình, việc phân tích benchmark giúp tôi nhận ra rằng không thể đánh giá một kiến trúc chỉ bằng pass rate. Cần xem xét đồng thời chi phí, số bước thực thi, khả năng kiểm soát quyền hạn và độ ổn định giữa các seed.

Tôi đánh giá mình đã hoàn thành các nhiệm vụ kỹ thuật chính được giao, có đóng góp vào kiểm thử bảo mật và phần tài liệu quyết định kiến trúc. Trong các dự án tiếp theo, tôi muốn cải thiện thêm khả năng thiết kế test case từ business rules, xây dựng cơ chế observability và đánh giá hệ thống agent bằng dữ liệu thực tế thay vì chỉ dựa trên môi trường mô phỏng.