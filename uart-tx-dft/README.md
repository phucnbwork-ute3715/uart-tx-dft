# UART TX DFT: Full Scan và Partial Scan dựa trên SCOAP

Đồ án môn Design for Testability (DFT): thiết kế scan chain cho UART TX và so sánh full scan với partial scan được lựa chọn dựa trên SCOAP kết hợp vai trò thanh ghi.

Phạm vi chính là **UART TX**, cấu hình `CLKS_PER_BIT = 16`, tổng cộng **18 FF**. Mô phỏng bằng Vivado/XSim; tổng hợp logic bằng Yosys; tính SCOAP bằng Python.

## Quy trình

1. Kiểm tra chức năng UART TX gốc.
2. Tổng hợp UART TX gốc và xuất netlist JSON.
3. Tính SCOAP và chọn FF cho partial scan.
4. Kiểm tra chức năng và hoạt động shift/capture của thiết kế scan.
5. Mô phỏng cùng danh sách 36 lỗi RTL cho full scan và partial scan.
6. So sánh độ phủ lỗi, số cổng logic và chu kỳ kiểm thử.

## Cấu trúc thư mục

| Thư mục | Nội dung |
|---|---|
| `rtl/baseline/` | RTL gốc, gồm `uart_tx.v` |
| `rtl/common/` | Scan DFF và ví dụ scan chain |
| `rtl/full_scan/` | RTL full scan, gồm `uart_tx_scan.v` |
| `rtl/partial_scan/` | RTL partial scan, bản chính là `uart_tx_partial_scoap.v` |
| `tb/` | Testbench chức năng, scan chain và fault campaign |
| `scripts/yosys/` | Script tổng hợp và xuất netlist |
| `scripts/python/` | Script phân tích netlist và tính SCOAP |
| `results/` | Log mô phỏng, log tổng hợp và bảng SCOAP |
| `docs/` | Ghi chú tài liệu tham khảo |

Các module UART RX, FIFO, UART TOP và các campaign cũ được giữ làm nội dung bổ sung. Kết quả chính của README dùng hai campaign có hậu tố `fault_campaign36` và bản partial có hậu tố `scoap`. Log `tb_uart_tx_partial_compare.log` thuộc bản partial cũ, không phải bằng chứng chạy bản SCOAP.

## Cấu trúc scan và lựa chọn FF

Thứ tự đóng gói thanh ghi:

```verilog
{uart_tx, state[1:0], clk_count[3:0], bit_index[2:0], data_reg[7:0]}
```

- Full scan: đưa toàn bộ 18 FF vào scan chain.
- Partial SCOAP: chọn `clk_count[3:0]` và `bit_index[1:0]`, tổng cộng 6 FF.
- FF không scan tiếp tục cập nhật theo logic chức năng trong khi shift.

Script chính: [`scoap_uart_tx.py`](scripts/python/scoap_uart_tx.py).
Kết quả: [`tx_baseline_scoap_ff.csv`](results/tx_baseline_scoap_ff.csv).

Script cắt tất cả FF của netlist baseline: Q trở thành pseudo-primary input, D trở thành pseudo-primary output; đặt `rst = 0`. Đây là **SCOAP tổ hợp trên mô hình cắt FF**, không phải SCOAP tuần tự của mạch partial scan.

Điểm xếp hạng dùng trong đồ án:

```text
score = max(CC0(D), CC1(D)) + CO(Q)
```

Đây là tiêu chí lựa chọn heuristic của đồ án. `clk_count[3:1]` có điểm 23, `bit_index[1]` có điểm 21 và `bit_index[0]` có điểm 20. Bổ sung `clk_count[0]` (điểm 13) để có khả năng nạp trực tiếp toàn bộ bộ đếm thời gian bit. Vì vậy lựa chọn kết hợp SCOAP với vai trò thanh ghi, không đơn thuần lấy sáu điểm cao nhất và chưa chứng minh tối ưu.

## Kết quả chính

Kết quả bốn pattern, giữ nguyên cấu hình đã chạy PASS:

| Tiêu chí | Full scan | Partial SCOAP |
|---|---:|---:|
| Tổng FF | 18 | 18 |
| FF trong scan chain | 18 | 6 |
| Cổng AND | 99 | 75 |
| Cổng OR | 63 | 49 |
| Cổng NOT | 21 | 16 |
| Cổng XOR | 2 | 1 |
| Cổng XNOR | 1 | 2 |
| **Tổng cổng tổ hợp** | **186** | **143** |
| Số pattern | 4 | 4 |
| Số lỗi phát hiện / khảo sát | 36/36 | 36/36 |
| Coverage trên tập lỗi khảo sát | 100% | 100% |
| Chu kỳ kiểm thử, gồm reset/chuẩn bị | 164 | 870 |
| Thời gian với clock 10 ns | 1,64 µs | 8,70 µs |
| Errors / trạng thái campaign | 0 / PASS | 0 / PASS |

Nguồn kết quả:

- [Full scan: mô phỏng](results/tb_uart_tx_full_fault_campaign36.log)
- [Partial SCOAP: mô phỏng](results/tb_uart_tx_partial_scoap_fault_campaign36.log)
- [Full scan: tổng hợp](results/tx_full_synth.log)
- [Partial SCOAP: tổng hợp](results/tx_partial_scoap_synth.log)

Partial SCOAP giảm **66,67% số FF trong scan chain** và **23,12% số cổng tổ hợp**. Với quy trình kiểm thử hiện tại, số chu kỳ tăng **5,30 lần** do chuẩn bị chức năng và quan sát thêm 192 chu kỳ mỗi pattern.

### Phạm vi diễn giải

- Tập lỗi gồm SA0/SA1 tại 18 bit `scan_d` ở RTL, mỗi lần chèn một lỗi bằng `force/release`. Coverage 100% chỉ áp dụng cho 36 lỗi này, không đại diện toàn bộ lỗi gate-level.
- Lỗi được chèn sau reset/chuẩn bị, trước scan load; kết quả giả định giai đoạn chuẩn bị không lỗi.
- Hai bản dùng cùng danh sách lỗi nhưng pattern và cách quan sát khác nhau. Partial dùng thêm `uart_tx` và `tx_ready` trong chuỗi quan sát.
- Số chu kỳ báo cáo là chi phí áp dụng một bộ bốn pattern, không phải thời gian chạy toàn bộ các lần mô phỏng lỗi và recovery.
- Số cổng là thống kê logic generic của Yosys; loại trừ `$scopeinfo`. Chưa phải diện tích ASIC, LUT FPGA, công suất hoặc kết quả timing.

## Chạy tổng hợp và SCOAP

Mở PowerShell tại thư mục chứa README. Cần Python 3 và Yosys trong PATH. Script SCOAP không yêu cầu thư viện Python bổ sung.

```powershell
New-Item -ItemType Directory -Force -Path "results", "build/tx_baseline", "build/tx_full", "build/tx_partial_scoap" | Out-Null

yosys -l results/tx_baseline_synth.log scripts/yosys/synth_tx_baseline.ys
if ($LASTEXITCODE -ne 0) { throw "Baseline synthesis failed" }

yosys -l results/tx_full_synth.log scripts/yosys/synth_tx_full.ys
if ($LASTEXITCODE -ne 0) { throw "Full-scan synthesis failed" }

yosys -l results/tx_partial_scoap_synth.log scripts/yosys/synth_tx_partial_scoap.ys
if ($LASTEXITCODE -ne 0) { throw "Partial-scan synthesis failed" }

python scripts/python/scoap_uart_tx.py build/tx_baseline/netlist.json 2>&1 | Tee-Object -FilePath results/tx_baseline_scoap_run.log
if ($LASTEXITCODE -ne 0) { throw "SCOAP analysis failed" }
```

Nếu dùng OSS CAD Suite tại ổ D, thêm PATH trong terminal trước khi chạy:

```powershell
$env:PATH = "D:\oss-cad-suite\oss-cad-suite\bin;D:\oss-cad-suite\oss-cad-suite\lib;" + $env:PATH
```

Các thư mục `build/` được tạo lại bằng lệnh trên và không đưa vào Git. Số cổng có thể thay đổi theo phiên bản công cụ; log đính kèm ghi Yosys `0.69+152`.

## Chạy mô phỏng Vivado

Tạo project mô phỏng và thêm các Design Sources sau:

```text
rtl/baseline/uart_tx.v
rtl/common/scan_dff.v
rtl/full_scan/uart_tx_scan.v
rtl/partial_scan/uart_tx_partial_scoap.v
```

Thêm testbench cần chạy vào Simulation Sources. Chọn từng module bên dưới làm **Set as Top**, chạy **Run Behavioral Simulation → Run All**:

| Testbench | Mục đích |
|---|---|
| `tb_uart_tx_compare` | So sánh chức năng baseline và full scan |
| `tb_uart_tx_partial_scoap_compare` | So sánh chức năng baseline và partial SCOAP |
| `tb_uart_tx_scan_chain` | Kiểm tra scan chain full scan |
| `tb_uart_tx_partial_scoap_scan_chain` | Kiểm tra scan chain partial SCOAP |
| `tb_uart_tx_full_fault_campaign36` | 36 lỗi RTL và chu kỳ kiểm thử full scan |
| `tb_uart_tx_partial_scoap_fault_campaign36` | 36 lỗi RTL và chu kỳ kiểm thử partial SCOAP |

Sau mỗi lần chạy, lưu bản sao `simulate.log` từ thư mục mô phỏng XSim vào `results/<ten_testbench>.log`. Lưu log trước khi chạy testbench tiếp theo. Không thêm tùy chọn `-log` trùng với tùy chọn Vivado tự tạo.

## Tài liệu và phạm vi chia sẻ

Các PDF tham khảo trong ZIP gốc không nằm trong repository được chuẩn bị này. Xem [docs/REFERENCES.md](docs/REFERENCES.md). Mã nguồn và log hiện có được giữ nguyên; README và cấu hình Git được bổ sung để thuận tiện tái lập kết quả.
