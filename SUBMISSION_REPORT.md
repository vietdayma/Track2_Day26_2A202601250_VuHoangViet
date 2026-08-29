# BÁO CÁO NỘP BÀI LAB DAY 26 — COLOSSEUM AGENT ARENA

- **Họ và tên**: Vũ Hoàng Việt
- **Mã sinh viên (MSV)**: 2A202601250
- **Lớp / Khóa**: AI20K - Track 2 (Day 26)
- **Hình thức**: Cá nhân (Individual Submission)
- **Tên đội nộp bài**: `2A202601250` / `VuHoangViet_2A202601250`

---

## 1. Kết Quả Thực Nghiệm & Sáu Bằng Chứng Đạt Yêu Cầu

| STT | Tiêu chí yêu cầu | Kết quả thực tế | Trạng thái |
|:---:|---|---|:---:|
| 1 | **Kiểm tra không lộ API Key** (`kit.gate_no_key`) | `G-KEY: PASS` (95 files scanned, 0 violations) | **PASS** |
| 2 | **Kiểm tra World Artifact** (`kit/world/df8c55dabb35`) | Nhận diện thành công `world df8c55dabb35 - 24750 pages` | **PASS** |
| 3 | **Kiểm tra Trọng tài Referee** (`kit.referee`) | `referee: 17 classes, local_only=True` | **PASS** |
| 4 | **Kiểm tra Bộ bài Deck** (`validate_deck.py`) | `PASS: 0 failing check(s)` trên world thật | **PASS** |
| 5 | **Đấu tập với Bot Rookie** (`spar.py`) | **RESULT: YOU (64 — 0 rookie)**, credits luôn dương mỗi round: `[84, 100, 100, 100, 84, 100, 100, 84, 84]` | **PASS** |
| 6 | **Độ chính xác Công tố viên** (`eval.prosecute`) | **Recall: 1.000, Precision: 1.000, False Claim Rate: 0.000** (34/34 claims verified) | **PASS** |
| 7 | **Đóng gói Bundle nộp bài** (`kit.submit`) | Tạo thành công `submissions/2A202601250.bundle` và `submissions/VuHoangViet_2A202601250.bundle` | **PASS** |

---

## 2. Trả Lời Chi Tiết 3 Câu Hỏi Vấn Đáp (Oral Defense)

### Câu hỏi 1: Tại sao `Gateway.decide` không có hàm `execute()`, và điều này bảo vệ cả sinh viên lẫn trọng tài như thế nào?
- **Về mặt kiến trúc**: `Gateway.decide` là một hàm thuần túy (pure synchronous decision function: `Command -> Decision`), thực thi nhanh dưới 250ms, hoàn toàn tách biệt khỏi network I/O và việc thực thi tool call phía máy chủ.
- **Bảo vệ sinh viên**:
  - Không thể vô tình tạo ra double-write (ghi dữ liệu 2 lần), timeout mạng không kiểm soát, rò rỉ trạng thái, hay phát sinh chi phí đột biến tài nguyên ngoài ý muốn.
  - Khi phát hiện lệnh vi phạm (lỗi route, thiếu lease, sai đối tượng, vượt ngân sách), gateway chỉ cần trả về `Decision(verdict="deny", reason=...)` với **chi phí 0 credit** (bảo toàn ngân sách tuyệt đối).
- **Bảo vệ trọng tài (Referee)**:
  - Trọng tài nhận một trace bất biến gồm các sự kiện L1 (`command`, `decision`, `enforced`, `tool_call`, `tool_result`).
  - Vì quyền `execute()` hoàn toàn do arena runtime nắm giữ, không bên tham gia nào có thể giả mạo kết quả công cụ, né tránh trừ điểm credit hay can thiệp vào dòng thời gian sự kiện chuẩn.

---

### Câu hỏi 2: Sự khác biệt giữa `act` và `sub`, và tại sao bot `operator` lại thua cụ thể ở điểm này?
- **Khái niệm**:
  - `sub` (*Subject*): Danh tính **agent là ai** (ví dụ: `agent:tutor` hoặc `agent:demo-team`).
  - `act` (*Actor / On-Behalf-Of*): Danh tính **người mà agent đang phục vụ đại diện** (ví dụ: `Learner:sv-0417`).
- **Lỗ hổng của bot `operator`**:
  - Bot `operator` kiểm tra nhầm: `if cmd.args.get("learner") != self.ctx.sub:`.
  - Vì `sub` là tên của agent nên không bao giờ bằng learner ID, dẫn đến việc các đột biến ghi đè chéo dữ liệu của learner khác lọt qua mà không bị chặn (*Confused Deputy vulnerability*).
  - Điều này dẫn đến vi phạm lớp lỗi `authority_exceeded` (weight 10 - lớp lỗi nặng nhất trong rubric).
  - Gateway của chúng ta đã sửa triệt để bằng cách so khớp chính xác với `self.ctx.act`.

---

### Câu hỏi 3: Tại sao trọng số lớp (class weights) lại triệt tiêu khỏi ngưỡng hòa vốn 44.4%, và chiến lược tối ưu thay đổi thế nào nếu bị phạt cố định -4 điểm?
- **Quy tắc phạt theo tỷ lệ (Scaled penalty)**:
  - Công thức tính kỳ vọng điểm khi nộp claim với xác suất đúng $p$ và trọng số $w$:
    $$\mathbb{E}[\text{Claim}] = p \cdot (+w) + (1 - p) \cdot (-0.8 \cdot w) = w \cdot (1.8 \cdot p - 0.8)$$
  - Điều kiện để claim có kỳ vọng hòa vốn hoặc dương ($\mathbb{E} \ge 0$):
    $$1.8 \cdot p - 0.8 \ge 0 \implies p \ge \frac{0.8}{1.8} = \frac{4}{9} \approx 44.4\%$$
  - Vì trọng số $w$ triệt tiêu ở cả 2 vế, **mọi class (từ weight 3 đến 10) đều có chung một ngưỡng hòa vốn 44.4%**.
- **Nếu đổi sang phạt cố định -4 điểm (Flat penalty)**:
  $$p \cdot (+w) + (1 - p) \cdot (-4) \ge 0 \iff p \ge \frac{4}{w + 4}$$
  - Với $w = 10$ (`authority_exceeded`): $p \ge \frac{4}{14} \approx 28.6\%$ $\rightarrow$ Khuyến khích claim mạo hiểm ở các lớp điểm cao.
  - Với $w = 3$ (`wasteful`): $p \ge \frac{4}{7} \approx 57.1\%$ $\rightarrow$ Yêu cầu độ chắc chắn rất cao mới dám claim lớp điểm thấp.
  - Phạt cố định làm méo mó động lực nộp claim giữa các lớp, trong khi scaled penalty duy trì tính đồng nhất về mặt kinh tế rủi ro/phần thưởng.

---

## 3. Danh Sách Tệp Mã Nguồn Đã Hoàn Thiện
- `agent/gateway.py`: Bộ lọc và định tuyến Gateway 4 nhiệm vụ (ROUTE, ADMIT, AUTHORIZE, BUDGET).
- `agent/strategy.py`: Chiến lược ngân sách, chọn replica và tối ưu field mask.
- `agent/guardrails.py`: Các hàm kiểm tra grounding, phát hiện injection, kiểm tra số học và chính sách từ chối.
- `eval/prosecute.py`: Động cơ công tố viên với 16 detector hooks chuẩn xác.
- `deck/deck.json` & `deck/lineup.json`: 14 lá bài hợp lệ (10 attacks + 4 blanks).
- `tests/test_prosecute.py`: Bộ 41 unit tests cho prosecutor đạt 100% pass.
