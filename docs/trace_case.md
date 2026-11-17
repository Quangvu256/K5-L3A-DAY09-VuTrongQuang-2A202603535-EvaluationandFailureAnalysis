# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Case đã phân tích: `seeded_regression C-002@11` (đối chứng `fixed C-002@11`)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11` (chạy đối chứng trực tiếp qua `python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11`; cùng lỗi trên `swarm C-002@10`) |
| Triệu chứng | `lab8 grader: FAIL (status resolved != denied)`. Reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."` và ledger ghi `refund_vnd=20000`, `refund_count=1`, `status=resolved`, trong khi nhãn kỳ vọng `refund_vnd=0`, `refund_count=0`, `status=denied`. |
| Bước sai đầu tiên | Event **#3** `model_override`: `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event #1 (`handoff`) tạo handoff đúng chuẩn chỉ cho phép read tools; event #2 (`investigator`) truy vấn DB `lookup_order` trả đúng `order_id="O-202"`, `status="delivered"`, `delay_days=2`. Đến event #3, agent mới lấy số `5` ngày từ văn bản ticket để ghi đè lên sự thật DB, khiến event #4 (`quote`) tính sai thành `20000` (`SHIP-LATE-03`). |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) trong `docs/failure_taxonomy.md` |
| Đối chứng | Trên `fixed C-002@11`: sau event #2 (`investigator`), **không** có event `model_override`; event #3 `quote` giữ nguyên `delay_days=2` từ DB và trả `{"amount_vnd": 0, "reason": "no_entitlement"}`, event #4 `reply` từ chối đúng, `ledger=0`, `status=denied` (`lab8 grader: PASS`). |
| Regression test | `test_trace_case_c002_no_model_override_on_fixed` trong `src/tests/test_lab9_evalset.py` (dùng `lab9.trace_case.investigate`, kiểm tra `seeded_regression` phát sinh event `model_override` và fail `lab8_pass`, còn `fixed` không có `model_override`, `refund_vnd == 0` và `lab8_pass is True`). |
| Chưa chắc chắn | Trace chỉ ghi sự kiện `model_override` khi biến thể bật flag ghi log; một biến thể ẩn (như `quiet_override`) có thể tin ticket mà không phát ra event `model_override`, nên cần kiểm tra đồng thời `quote` / `refund_amount_correct` ở mức outcome. |

*(Các trial lỗi khác đã quan sát trên `seeded_regression` seed 11 để đối chiếu thêm: `C-007@11` — bước sai đầu tiên ở event #5 `reply` (`FM-02`); `C-013@11` & `C-014@11` — bước sai đầu tiên ở event #5 `proposal` (`FM-05`)).*

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
