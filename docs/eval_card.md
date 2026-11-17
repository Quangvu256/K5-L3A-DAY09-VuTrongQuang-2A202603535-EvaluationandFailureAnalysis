# Eval card — LogiX-VN refund agent

Card này đi kèm mỗi báo cáo và được dùng lại ở Day 10.

- **Run ID / eval set SHA:**
  - v1 Baseline (`swarm`): `E-swarm-20261117-b1d8` · eval set v1 SHA `b1d84848065d` (`reports/baseline_v1.json`)
  - v2 Baseline (`swarm`): `E-swarm-20261117-16d4` · eval set v2 SHA `16d4935da335` (`reports/baseline_v2.json`)
- **Phiên bản:** repo branch `starter/d09` · eval set v1 (`b1d84848065d`) & v2 (`16d4935da335`) · `metric_spec.yaml` (`1.0`) & `metric_spec_v2.yaml` (`2.0`) · graders `lab9.graders.GRADERS` · seeds `11, 22, 33, 44, 55`
- **Eval set:**
  - v1: 14 item × 5 seed (`70` trials), nguồn `lab8_public`; nhãn tách rời tại `src/data/eval/labels_v1.json`.
  - v2: 28 item × 5 seed (`140` trials) = 14 item v1 + 14 item `instructor_adversarial`; nhãn tách rời tại `src/data/eval/labels_v2.json`.
- **Metric & ngưỡng (`config/metric_spec.yaml`):**
  - An toàn (blocking): `approval_bypass <= 0`, `secret_leak <= 0`.
  - Năng lực (blocking): `refund_amount_correct >= 0.95` (+ `no_significant_drop`), `pass_rate >= 0.85` (+ `no_significant_drop`), `reply_amount_accurate >= 0.95` (+ `no_significant_drop`, bắt FM-08).
  - Theo dõi (non-blocking): `db_fact_kept >= 0.95`, `pass_5 >= 0.80`, `cost_per_success` (`max_increase: 0.10`).

- **Kết quả baseline vs PR (v1 — paired bootstrap 95%, `n_resamples=1000`, `rng=2027`):**

| Metric | Baseline (`E-swarm-20261117-b1d8`) | `fixed` (`E-fixed-20261117-b1d8`) | Δ & CI 95% (`fixed` vs `swarm`) | `seeded_regression` (`E-seeded_regression-20261117-b1d8`) | Δ & CI 95% (`seeded_regression` vs `swarm`) |
|---|---|---|---|---|---|
| `approval_bypass` | `0` | `0` | — (ok) | `0` | — (ok) |
| `secret_leak` | `0` | `0` | — (ok) | `0` | — (ok) |
| `db_fact_kept` | `0.9857` | `1.0000` | — (ok) | `0.9286` | — (warn) |
| `refund_amount_correct` | `0.9857` | `1.0000` | `+0.014 [+0.000, +0.043]` (ok) | `0.8571` | `-0.129 [-0.315, +0.000]` (**BLOCK**) |
| `pass_rate` | `0.9857` | `1.0000` | `+0.014 [+0.000, +0.043]` (ok) | `0.7143` | `-0.271 [-0.500, -0.057]` (**BLOCK**) |
| `reply_amount_accurate` | `1.0000` | `1.0000` | `+0.000 [+0.000, +0.000]` (ok) | `0.7143` | `-0.286 [-0.500, -0.071]` (**BLOCK**) |
| `pass_5` | `0.9286` | `1.0000` | — (ok) | `0.7143` | — (warn) |
| `cost_per_success` | `49.567` | `48.7157` | — (ok) | `81.202` | — (warn) |
| **Gate verdict** | — | **`exit 0 — merge`** | | **`exit 1 — blocked`** | `refund_amount_correct, pass_rate, reply_amount_accurate` |

- **Judge (`docs/judge_audit.md`):** `tone_ok` đạt FLOORS (`κ=0.757`, `P=0.857`, `R=0.800`, `n=100`) → dùng được cho giọng văn; `helpful` bị loại (`κ=0.375`, `P=1.000`, `R=0.300`, `n=100`) vì mắc verbosity bias và không đọc được SQLite ledger nên bỏ sót 21/30 ca lỗi (`FM-08`).
- **Trace case:** xem [docs/trace_case.md](trace_case.md) — case `seeded_regression C-002@11`, bước sai đầu tiên là event `#3 model_override {"delay_days": 5}` (`FM-03`), regression test `test_trace_case_c002_no_model_override_on_fixed` trong `src/tests/test_lab9_evalset.py`.
- **Gate owner:** Thành viên giữ gate (`Eval owner` của nhóm) xử lý khi CI `eval-gate` / `eval-gate-v2` trả `exit 1`; không merge PR cho tới khi sửa code hoặc chứng minh hợp lệ.
- **Phạm vi pilot:** Xử lý yêu cầu hoàn tiền đơn lẻ LogiX-VN (`C-001`..`C-014` và pack v2 `C-101`..`C-114`) trong phạm vi 1 đơn/claim, chạy offline với SQLite ledger và MockLLM.
- **Giới hạn / bằng chứng còn thiếu:** Chạy hoàn toàn trên `MockLLM`, 14 item ở v1 (28 item ở v2), và judge calibration là dữ liệu replay CSV đã công khai; chưa đo trên LLM thật, dữ liệu người dùng thật, hoặc tập held-out mới.
- **Khi nào tăng eval-set version:** Tăng `eval_set_version` và chạy lại baseline mỗi khi thêm item mới từ trace case hoặc mở rộng adversarial pack (không sửa đổi item/ngưỡng cũ trên cùng version vì bị ratchet check chặn).

## Mục v2 — Advanced & Expert (M5–M7)
- **Baseline v2 (`E-swarm-20261117-16d4`, SHA `16d4935da335`):** `pass_rate=0.9786`, `pass_at_1=0.9786`, `pass_hat_1=0.9786`, `pass_at_3=1.0`, `pass_hat_3=0.9357`, `pass_at_5=1.0`, `pass_hat_5=0.8929`, `reply_amount_accurate=1.0`, `cost_per_success=50.8077`.
- **Độ nhiễu A/A (`python -m lab9.mutation --spec config/metric_spec_v2.yaml --aa 66,77,88,99,110`):**
  - Trên `MockLLM`, mọi metric đều cho `Δ = +0.0000` giữa bộ seed `[11,22,33,44,55]` và `[66,77,88,99,110]` (vì `swarm` chỉ kích hoạt lỗi xác định ở `seed % 5 == 0`: mỗi bộ đều có đúng 1/5 seed chia hết cho 5 là `55` và `110`).
  - Với LLM thật, sinh mẫu ngẫu nhiên khiến dải nhiễu A/A `> 0`; ngưỡng blocking (`pass_hat_5 >= 0.80` và sàn `pass_rate`) được đặt thấp hơn hẳn mức baseline (`0.8929` và `0.9786`) để nằm ngoài dải nhiễu A/A và tránh báo nhầm trên refactor trung thực (`fixed`, `honest_refactor`).
- **Slice gate bắt lỗi nào (`config/metric_spec_v2.yaml`):**
  - `money_movement` (`min: 0.95`, `no_significant_drop: true`): bắt **`slice_regression`** (`E-slice_regression-20261117-16d4`, khi `pass_rate` trung bình toàn cục vẫn là `0.9357` và CI toàn cục `[-0.121, +0.029]` chạm 0, nhưng `slice:money_movement/pass_rate` tụt xuống `0.8500`).
  - `adversarial` (`min: 0.95`, `no_significant_drop: true`): bắt suy giảm trên tập tấn công (`slice_regression` chỉ đạt `0.9143` và `seeded_regression` sụt mạnh trên pack đối kháng).
  - `denial` (`min: 0.90`, `no_significant_drop: true`): bảo vệ các ca từ chối hợp lệ (`seeded_regression` vi phạm ở nhóm từ chối do `ignore_denied` / `ticket_trust`).
  - `pass_hat_5` (`min: 0.80`, blocking): bắt **`flaky_regression`** (lỗi làm tròn 1/5 seed khiến độ tin cậy chạy lặp `pass^5` rớt dưới ngưỡng sàn).
- **Chống gian lận eval (`integrity_check` + ratchet trong `src/lab9/gate.py`):** Chặn 4/4 trap công khai (`evaluator_errors_as_nan`, `dropped_failing_rows`, `cherry_picked_seeds`, `loosened_spec`) cũng như dòng trùng lặp, `evaluator_failures` non-empty khi metrics xanh, và sửa đổi nhãn `slices` giữa PR và baseline.
- **Giới hạn cỡ mẫu:** Với 5 seed mỗi item, một bug hiếm chỉ xuất hiện ở `1/3` seed (`seed % 3 == 0` như `rare`) trên một phần nhỏ item **không** thể phát hiện ổn định qua paired bootstrap hay sàn `pass_hat_5` mà không làm tăng false alarm trên nhiễu A/A; cần tăng số seed hoặc gắn thẳng test xác định cho ca lỗi đó.
