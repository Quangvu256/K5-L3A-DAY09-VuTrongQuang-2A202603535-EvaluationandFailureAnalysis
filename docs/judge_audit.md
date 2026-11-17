# Judge audit

Chạy: `python -m lab9.judge --csv <file> --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| `tone_ok` (`judge_tone_public.csv`) | 100 | 0.757 | 0.857 | 0.800 | **PASS** (0.757≥0.6, 0.857≥0.8, 0.800≥0.7) | **dùng** |
| `helpful` (`judge_helpful_public.csv`) | 100 | 0.375 | 1.000 | 0.300 | **FAIL** (κ=0.375<0.6, R=0.300<0.7) | **loại** |

- Ma trận nhầm lẫn (positive = `FAIL`):
  - `tone_ok`: `tp=24, fp=4, fn=6, tn=66`; tỉ lệ fail quan sát `0.28` → Rogan-Gladen hiệu chỉnh `0.30`.
  - `helpful`: `tp=9, fp=0, fn=21, tn=70`; tỉ lệ fail quan sát `0.09` → Rogan-Gladen hiệu chỉnh `0.30` (bỏ sót 21/30 ca lỗi thật).

## ≥5 case audit thủ công
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | `H-008` ("Em đã hoàn tiền cho anh rồi nha, anh check giúp em.") | `PASS` | `FAIL` | **Người đúng** | Judge coi là "thân thiện"; sửa rubric cấm đại từ suồng sã (`em`/`nha`/`bạn` với khách) và bắt buộc `Dạ` + `anh/chị` + `ạ`. |
| 2 | `H-029` ("Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm.") | `PASS` | `FAIL` | **Người đúng** | Judge bị đánh lừa bởi cụm `"Dạ, vui lòng"` ở đầu câu, bỏ qua giọng ra lệnh `"không cần nhắn thêm"`; thêm rule phát hiện câu mệnh lệnh/cấm đoán khách. |
| 3 | `H-051` ("Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ.") | `FAIL` | `PASS` | **Người đúng** | Judge phạt sai chỉ vì từ `"refund"`; cập nhật rubric ghi rõ danh sách thuật ngữ CS LogiX được phép dùng. |
| 4 | `H-063` ("Dạ, LogiX đã hoàn 20.000đ và sẽ hoàn thêm cho anh/chị ạ.") | `PASS` | `FAIL` | **Người đúng** | Judge không bắt được lời hứa ngoài kết quả hệ thống (`"sẽ hoàn thêm"`); bổ sung tiêu chí cấm cam kết tiền ngoài ledger. |
| 5 | `H-070` ("Dạ, LogiX đã hoàn 90.000đ cho đơn O-909 ạ 😂😂") | `PASS` | `FAIL` | **Người đúng** | Judge bỏ qua emoji cười cợt trong ngữ cảnh khiếu nại; thêm rule cấm emoji vào rubric tone. |
| 6 | `H-081` ("Dạ, đơn O-107 chưa đủ điều kiện hoàn theo chính sách hiện hành ạ.") | `FAIL` | `PASS` | **Judge đúng** (sau sửa nhãn) | Chú thích CSV xác nhận người gán nhãn đọc nhầm ticket dẫn tới nhãn người sai; sửa lại nhãn ground-truth và giữ prompt judge. |
| 7 | `HL-015` ("Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 ạ." — ledger ghi `40.000đ`) | `PASS` | `FAIL` | **Người đúng** | Judge chỉ đọc văn bản reply, không có quyền truy cập ledger nên tưởng câu trả lời đầy đủ; **tách kiểm tra số tiền sang code grader (FM-08)** thay vì dùng judge. |
| 8 | `HL-016` (Xin lỗi dài dòng 2 lần, nêu sai tiền `55.000đ` vs ledger `35.000đ`) | `PASS` | `FAIL` | **Người đúng** | Judge mắc **verbosity bias** (thấy lời xin lỗi lịch sự, dài dòng liền chấm `PASS`), bỏ qua lỗi sai số tiền; loại judge này khỏi gate. |

## Kết luận
- **Tiêu chí bị loại:** `helpful` (`judge_helpful_public.csv`) **không được dùng làm gate release**. Ma trận nhầm lẫn cho thấy `fn=21` trên tổng số `30` ca lỗi thật (`Recall = 0.30`, `κ = 0.375`), nghĩa là judge bỏ sót 70% ca sai lệch (chủ yếu là các ca báo sai số tiền `FM-08` được ngụy trang bằng câu xin lỗi dài dòng và sai xưng hô), làm tụt tỉ lệ lỗi quan sát xuống `0.09` dù tỉ lệ thật sau hiệu chỉnh Rogan-Gladen là `0.30`.
- **Phân công kiểm soát:** `tone_ok` (`κ = 0.757`, `P = 0.857`, `R = 0.800`) đạt cả 3 ngưỡng FLOORS nên có thể dùng để theo dõi chất lượng giọng văn; toàn bộ phần kiểm tra tính đúng đắn của số tiền (`FM-08`) phải giao cho code grader đối chiếu trực tiếp với ledger (`reply_amount_accurate`).
- **Rủi ro còn lại:** Judge chỉ được kiểm trên dữ liệu replay đã công khai (100 mẫu mỗi tiêu chí); cần thu thập tập nhãn người mới trên các lượt chạy thật để có held-out thật, tránh overfit khi chỉnh rubric.
