# Decision record

- Shape chọn: orchestrator
- Commit hash: 2e75657
- Run id dẫn chứng: R-orchestrator-C-001-s11
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000
- Cost per success (mô phỏng, VND): 46.43
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 trường hợp phát hiện trong các kiểm thử security audit; 9/9 audit tests và 22/22 authorization, handoff, trace, orchestrator tests passed.

## Vì sao shape này

Nhóm chọn Orchestrator vì kiến trúc này phân tách rõ quyền hạn giữa các thành phần. Các investigator chỉ thực hiện tác vụ đọc thông tin; Orchestrator chịu trách nhiệm tổng hợp kết quả, kiểm tra điều kiện nghiệp vụ và gọi thao tác refund. Trong benchmark gồm 14 claim và 5 seed, Orchestrator đạt 70/70 lượt thành công (pass rate 1.000), tương đương Workflow. Tuy nhiên, chi phí mô phỏng của Orchestrator là 46.43 VND/lần thành công, cao hơn Workflow (28.87 VND). Nhóm chấp nhận phần chi phí tăng thêm để đổi lấy ranh giới quyền hạn và khả năng kiểm soát luồng thực thi rõ ràng hơn.

## Có đáng đổi lấy multi-agent không

Trong kết quả hiện tại, multi-agent không làm tăng pass rate so với Workflow: cả hai đều đạt 100%. Vì vậy, lựa chọn Orchestrator không dựa trên tuyên bố rằng multi-agent chính xác hơn, mà dựa trên khả năng phân quyền và kiểm soát side effect. Chi phí mô phỏng tăng khoảng 60.8% so với Workflow. Nhóm sẽ cân nhắc chuyển về Workflow nếu chi phí trở thành ràng buộc chính và không còn yêu cầu tách quyền investigator khỏi quyền thực hiện refund. Ngược lại, Orchestrator phù hợp khi cần giới hạn quyền công cụ, kiểm tra approval và kiểm soát thao tác có side effect.

## Trace và bằng chứng

Trace được dùng để theo dõi các bước thực thi, handoff giữa Orchestrator và investigator, kết quả kiểm tra nghiệp vụ và trạng thái xử lý. Hệ thống không đưa secret, raw headers, số điện thoại hoặc địa chỉ cá nhân vào handoff/trace vì đây không phải dữ liệu cần thiết cho việc điều phối và kiểm toán. Nhóm kiểm tra điều này bằng các canary secret và các trường nhạy cảm trong `tests/test_team_orchestrator.py`, `tests/test_team_security_audit.py` và các kiểm thử authorization/handoff/trace. Các kiểm thử audit đã chạy đạt 9/9; nhóm kiểm thử authorization, approval binding, handoff secret, trace contract và orchestrator đạt 22/22. Kết quả này chứng minh các trường hợp đã kiểm thử không phát hiện secret leak hoặc approval bypass; không được hiểu là bảo đảm tuyệt đối cho mọi đầu vào hay môi trường production.

## Giới hạn

Benchmark sử dụng MockLLM seeded, 14 claim, 5 seed và SQLite riêng cho từng lượt thử. Chi phí là chi phí mô phỏng theo cấu hình trong benchmark, không phải giá API thực tế. Latency không khả dụng. Kết quả 100% chỉ phản ánh corpus và các seed đã kiểm thử, không bảo đảm hiệu năng trên dữ liệu production. Crash safety 20/20 phản ánh các kịch bản crash được định nghĩa trong bộ test, không bao phủ mọi lỗi có thể xảy ra trong môi trường thực tế.
