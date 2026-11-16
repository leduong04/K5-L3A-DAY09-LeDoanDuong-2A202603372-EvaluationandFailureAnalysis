# Team

## Thông tin nhóm

- Tên nhóm: Gehihi
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-Gehihi-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/lythanhlap/K5-L3A-DAY08-Gehihi-FromWorkflowsToAgents, nhánh `main`, commit `24ba8c2` (ghi trong [report/team/decision_adr.md](report/team/decision_adr.md))
- Shape chọn: `workflow`

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Lý Thành Lập | 2A202603414 | 26ai.laplt@vinuni.edu.vn | [lythanhlap](https://github.com/lythanhlap) | Nhóm trưởng. Shape A `workflow` (shape chọn), integration/merge, benchmark, crash suite, tự kiểm hidden, decision ADR |
|   2 | Nguyễn Tiến Cường | 2A202603425 | cuongdelldz@gmail.com | [cuong-sadboi](https://github.com/cuong-sadboi) | Shape B `single` (ReAct có giới hạn), research card R4, ablation workflow vs single |
|   3 | Lê Đoàn Dương | 2A202603372 | leduong04st@gmail.com | [leduong04](https://github.com/leduong04) | Shape C `orchestrator`, handoff allowlist, security audit (approval, secret/PII), group report |

Phân công theo checkpoint:

- CP1: cả nhóm cài môi trường, chạy smoke, đọc business rules và contracts. Nhóm trưởng tạo repo nhóm.
- CP2–CP3: mỗi người viết một shape trên nhánh riêng (`lythanhlap`, `feat/single`, `LeDoanDuong`), kèm `crash_at` và test riêng. Nhóm trưởng merge vào `main` qua PR.
- CP4: Người 1 chạy crash suite (`20/20`). Người 3 rà approval bypass và rò secret/PII trên cả 3 shape.
- CP5: Người 1 chạy benchmark và điền ADR. Người 2 làm research card và ablation. Người 3 viết group report. Mỗi người viết báo cáo cá nhân.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Ly Thanh Lap | Workflow (shape chọn), integration/merge, benchmark + crash + tự kiểm hidden, decision ADR | 6234dbe; 0226eb1; 24ba8c2; defe1b2 | tests/test_team_workflow.py |
| Nguyen Tien Cuong | Single agent, research card R4, ablation workflow vs single | bcea95e; bd247b5 | tests/test_team_single.py |
| Le Doan Duong | Orchestrator, security audit, group report | c4f47bd; 44b0572; 2e75657 | tests/test_team_orchestrator.py; tests/test_team_security_audit.py |

## Cá nhân

Mỗi thành viên có báo cáo riêng trong `report/team/`:

| Thành viên | Báo cáo cá nhân |
| --- | --- |
| Lý Thành Lập | [report/team/2A202603414_LyThanhLap.md](report/team/2A202603414_LyThanhLap.md) |
| Nguyễn Tiến Cường | [report/team/2A202603425_NguyenTienCuong.md](report/team/2A202603425_NguyenTienCuong.md) |
| Lê Đoàn Dương | [report/team/2A202603372_LeDoanDuong.md](report/team/2A202603372_LeDoanDuong.md) |

### 2A202603414_LyThanhLap

#### Phần việc phụ trách

- Shape A `workflow`: `src/lab8/shapes/workflow.py`, shape nhóm chọn để chấm hidden.
- Integration: tạo repo nhóm, merge các nhánh vào `main` (PR #3, #5), gỡ conflict.
- Bằng chứng: `src/reports/benchmark.json`, `src/reports/crash.json`, `src/reports/hidden_selfcheck.json`, `report/team/decision_adr.md`.

#### Đóng góp cụ thể

- `6234dbe`: viết shape workflow. Tiền do `compute_compensation` tính, quote đầy đủ và `crash_at` được truyền vào `durability.issue_refund`.
- `0226eb1`: viết `tests/test_team_workflow.py`.
- `24ba8c2`: chạy benchmark 4 shape × 5 seed và crash suite `20/20`.
- `defe1b2`: điền decision record và kết quả tự kiểm trên `student_drop/`.

#### Bằng chứng

- `src/tests/test_team_workflow.py`, `src/reports/benchmark.json`, `src/reports/crash.json`.

### 2A202603425_NguyenTienCuong

#### Phần việc phụ trách

- Shape B `single`: `src/lab8/shapes/single.py`, ReAct tối đa 8 bước, chỉ dùng tool đọc.
- Research card R4 và ablation workflow vs single.

#### Đóng góp cụ thể

- `bcea95e`: viết shape single và `tests/test_team_single.py`.
- `bd247b5`: điền `report/team/research_card.md`, thêm `src/reports/ablation_workflow_vs_single.json`.

#### Bằng chứng

- `src/tests/test_team_single.py`, `report/team/research_card.md`, `src/reports/ablation_workflow_vs_single.json`.

### 2A202603372_LeDoanDuong

#### Phần việc phụ trách

- Shape C `orchestrator`: `src/lab8/shapes/orchestrator.py`, handoff allowlist, investigator chỉ đọc.
- Security audit: approval bypass, rò secret/PII trong trace trên cả 3 shape.
- Báo cáo nhóm `report/team/group_report.md`.

#### Đóng góp cụ thể

- `c4f47bd`: viết shape orchestrator.
- `44b0572`: viết `tests/test_team_orchestrator.py`.
- `2e75657`: thêm `tests/test_team_security_audit.py` kiểm approval và trace trên các shape.

#### Bằng chứng

- `src/tests/test_team_orchestrator.py`, `src/tests/test_team_security_audit.py`, `report/team/group_report.md`.
