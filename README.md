# THIẾT KẾ BỘ LỌC FIR VÀ THỰC HIỆN TRÊN FPGA

 
## Mục lục
 
1. [Giới thiệu](#i-giới-thiệu)
2. [Cơ sở lý thuyết](#ii-cơ-sở-lý-thuyết)
3. [Thiết kế hệ thống trên phần mềm](#iii-thiết-kế-hệ-thống-trên-phần-mềm)
4. [Thiết kế phần cứng Verilog HDL](#iv-thiết-kế-phần-cứng-verilog-hdl)
5. [Mô phỏng và thực hiện trên FPGA](#v-mô-phỏng-và-thực-hiện-trên-fpga)
6. [Kết luận và đánh giá](#vi-kết-luận-và-đánh-giá)
---
 
## I. Giới thiệu
 
### 1.1 Lý do chọn đề tài
 
Xử lý tín hiệu số (DSP) là nền tảng của nhiều ứng dụng hiện đại: mạng 5G, hình ảnh AI, hệ thống thời gian thực (RTOS)... Các ứng dụng này yêu cầu bộ xử lý tín hiệu số có **tốc độ cao** và **tiêu thụ điện năng thấp**.
 
Truyền thống, các hệ thống DSP hiệu suất cao sử dụng ASIC hoặc chip DSP chuyên dụng. FPGA tạo ra giải pháp thay thế **linh hoạt và có thể tái lập trình**, đặc biệt phù hợp cho nghiên cứu và thử nghiệm trong phòng thí nghiệm.
 
### 1.2 Phạm vi đề tài
 
Điểm hạn chế khi thực hiện lọc FIR trên FPGA:
- Tính toán số **floating-point** gây tốn tài nguyên phần cứng
- Phép **nhân cộng tích lũy (MAC)** yêu cầu các khối DSP chuyên dụng
**Giải pháp đề xuất:** Sử dụng kỹ thuật **Số học phân tán (Distributed Arithmetic — DA)** thay thế phép nhân bằng tra cứu bảng LUT, qua hai phương pháp:
- **SDA** — Serial Distributed Arithmetic
- **PDA** — Parallel Distributed Arithmetic (2-bit)
### 1.3 Platform
 
| Phần mềm/Phần cứng | Chi tiết |
|-------------------|---------|
| MATLAB R2020a | Thiết kế hệ số, kiểm tra floating-point |
| Visual Studio Code + GCC | C model fixed-point |
| Intel Quartus Prime Lite | Tổng hợp mạch và lập trình FPGA |
| ModelSim | Mô phỏng HDL |
| Signal Tap Logic Analyzer | Kiểm tra tín hiệu trên board thực |
| Board | Terasic DE10-Standard (Intel Cyclone V SoC FPGA — 5CSXFC6D6F31C6N) |
 
---
 
## II. Cơ sở lý thuyết
 
### 2.1 Bộ lọc FIR (Finite Impulse Response)
 
Phương trình bộ lọc FIR nhân quả:
 
```
y(n) = Σ h(k) · x(n−k),   k = 0..N
```
 
Trong đó:
- `h(k)`: hệ số lọc (impulse response)
- `x(n)`: tín hiệu đầu vào
- `N`: bậc bộ lọc; số hệ số `M = N + 1`
**Ưu điểm FIR:**
- Pha tuyến tính (linear phase)
- Vô điều kiện ổn định (BIBO stable)
- Dễ thiết kế và thực hiện
### 2.2 Phương pháp cửa sổ Hamming
 
Hàm cửa sổ Hamming:
 
```
w(n) = 0.54 − 0.46·cos(2πn/M),    0 ≤ n ≤ M
```
 
Hệ số lọc thiết kế:
 
```
b(n) = h_ideal(n) · w(n)
```
 
**So sánh các loại cửa sổ:**
 
| Cửa sổ | Độ rộng chuyển tiếp | Suy giảm dải triệt | Nhận xét |
|--------|:-------------------:|:------------------:|---------|
| Rectangular | 1.8π/M | 21 dB | Dải chuyển tiếp hẹp nhất |
| Hanning | 6.2π/M | 44 dB | Trung bình |
| **Hamming** | **6.6π/M** | **53 dB** | **Cân bằng tốt nhất** |
| Blackman | 11.1π/M | 74 dB | Suy giảm mạnh nhất |
 
> **Hamming được chọn** vì dung hòa tốt nhất giữa độ rộng chuyển tiếp và suy giảm dải triệt.
 
### 2.3 Số floating-point và fixed-point
 
**Floating-point (IEEE 754):**
 
| Dạng | Sign | Exponent | Mantissa | Tổng |
|------|:----:|:--------:|:--------:|:----:|
| Single Precision | 1 bit | 8 bit | 23 bit | 32 bit |
| Double Precision | 1 bit | 11 bit | 52 bit | 64 bit |
 
Biểu diễn: `x = (−1)^Sign × 2^(Exponent−Bias) × (1.Mantissa)`
 
**Hạn chế floating-point trên FPGA:**
- Yêu cầu FPU (Floating Point Unit) chuyên dụng
- Tốc độ tính toán thấp, không phù hợp DSP thời gian thực
**Chuyển đổi sang fixed-point 8-bit:**
 
```
Bước 1: b = round(a × 2^F)      (F = số bit phần lẻ)
Bước 2: Clamp về [-128, 127]    (phạm vi int8_t)
```
 
| Tín hiệu | Hệ số tỷ lệ F | Phạm vi | Lý do |
|----------|:------------:|---------|-------|
| x(n) — mẫu tín hiệu | 7 | ×2⁷ = ×128 | Biên độ sin ∈ [−1, 1] |
| h(k) — hệ số lọc | 8 | ×2⁸ = ×256 | Hệ số nhỏ, cần độ chính xác cao |
 
### 2.4 Số học phân tán (Distributed Arithmetic)
 
#### 2.4.1 Nguyên lý
 
Biểu diễn mẫu x_k theo bù 2 (B bit):
 
```
x_k = −x_k(B−1) · 2^(B−1) + Σ x_kb · 2^b,   b = 0..B−2
```
 
Từ đó, phép MAC:
 
```
y(n) = Σ h_k · x_{n−k}
```
 
được viết lại thành **tổng có trọng số của các bit đầu vào**, thay vì nhân trực tiếp.
 
#### 2.4.2 Bảng LUT (Look-Up Table)
 
Với 4 đầu vào (A, B, C, D), bảng LUT 4-bit × 16 từ:
 
| Ai Bi Ci Di | Kết quả |
|:-----------:|---------|
| 0 0 0 0 | 0 |
| 0 0 0 1 | Y₀ |
| 0 0 1 0 | Y₁ |
| 0 0 1 1 | Y₀ + Y₁ |
| … | … |
| 1 1 1 1 | Y₃ + Y₂ + Y₁ + Y₀ |
 
> LUT lưu trước **tất cả tổ hợp tổng hệ số** → tra cứu thay thế phép nhân.
 
#### 2.4.3 Thiết kế hệ số đối xứng
 
Với bộ lọc FIR 16 hệ số đối xứng `h(k) = h(15−k)`:
 
```
y(n) = Σ h(k) · [x(n−k) + x(n−15+k)],   k = 0..7
```
 
→ **Giảm 1/2 số phép nhân** (8 phép thay vì 16).
 
---
 
## III. Thiết kế hệ thống trên phần mềm
 
### 3.1 Thông số thiết kế
 
| Thông số | Giá trị |
|----------|---------|
| Loại lọc | FIR Thấp qua (Low-pass) |
| Phương pháp cửa sổ | Hamming |
| Số hệ số | 16 (đối xứng) |
| Tần số lấy mẫu f_s | 50 kHz |
| Tần số cắt f_c | 4.5 kHz |
| Tín hiệu đầu vào | x1 = sin(2π·1kHz·t) + x2 = sin(2π·10kHz·t) |
| Số mẫu | 176 mẫu |
| Biểu diễn số | Fixed-point 8-bit có dấu |
 
### 3.2 Luồng thiết kế (Design Flow)
 
```
[Tín hiệu tổng hợp Xn = x1 + x2]
              │
              ▼ MATLAB
[Thiết kế bộ lọc FIR lý tưởng]
  h_ideal(n) = Lowpass(fc, fs, N)
              │
              ▼ MATLAB
[Áp dụng cửa sổ Hamming]
  bn = h_ideal(n) × w_hamming(n)
              │
              ▼ MATLAB/C
[Chuyển đổi sang Fixed-point 8-bit]
  x_fixed: Q1.7 (frac_bits=7)
  h_fixed: Q0.8 (frac_bits=8)
              │
              ▼ C Model
[Kiểm tra lọc FIR với fixed-point]
  So sánh kết quả C vs MATLAB
              │
              ▼ Verilog HDL
[Thiết kế 3 kiến trúc FPGA]
  MAC | SDA | PDA
              │
              ▼ Quartus Prime
[Tổng hợp và lập trình board]
              │
              ▼ ModelSim + Signal Tap
[Kiểm tra kết quả]
```
 
### 3.3 Hệ số lọc thu được
 
**MATLAB code:**
 
```matlab
Fs = 50000;  % Tần số lấy mẫu 50 kHz
N = 16;      % Số hệ số
f_c = 4500;  % Tần số cắt 4.5 kHz
 
hn = Lowpass(f_c, Fs, N);   % Hệ số lý tưởng
wn = Hamming(N-1);           % Cửa sổ Hamming
bn = hn .* wn;               % Hệ số cuối
```
 
**Hệ số lọc — Floating-point và Fixed-point (Q0.8):**
 
| STT | Floating-point | Fixed-point (×256) |
|:---:|:--------------:|:-----------------:|
| 1 | −0.00298562 | **−1** |
| 2 | −0.00298562 | **−1** |
| 3 | +0.00042211 | **0** |
| 4 | +0.01581831 | **4** |
| 5 | +0.04908484 | **13** |
| 6 | +0.09683242 | **25** |
| 7 | +0.14519433 | **37** |
| 8 | +0.17582589 | **45** |
| 9 | +0.17582589 | **45** ← đối xứng |
| 10 | +0.14519433 | **37** |
| 11 | +0.09683242 | **25** |
| 12 | +0.04908484 | **13** |
| 13 | +0.01581831 | **4** |
| 14 | +0.00042211 | **0** |
| 15 | −0.00298562 | **−1** |
| 16 | −0.00298562 | **−1** |
 
> Tính đối xứng `h(k) = h(15−k)` được xác nhận → cho phép tối ưu hóa phần cứng.
 
### 3.4 C Model — Kiểm tra Fixed-point
 
**Hàm chuyển đổi floating → fixed 8-bit:**
 
```c
int8_t float_to_fixed_8bit(float x, int frac_bits) {
    float scale = (float)pow(2, frac_bits);
    int16_t fp = (int16_t)round(x * scale);
    if (fp > 127)  fp = 127;
    if (fp < -128) fp = -128;
    return (int8_t)fp;
}
```
 
**Hàm lọc FIR fixed-point:**
 
```c
void Loc_FIR_fixed(int8_t *x, int8_t *h, int L, int N, int16_t *y) {
    for (int n = 0; n < L; n++) {
        y[n] = 0;
        for (int k = 0; k < N; k++)
            if ((n-k+1) >= 0 && (n-k+1) < L)
                y[n] += h[k] * x[n-k];   // int8×int8 → int16
    }
}
```
 
**Hàm scale kết quả 16-bit → 8-bit:**
 
```c
int8_t fixed16_to_fixed8(int16_t value, int frac_bits) {
    int16_t v = round(value >> frac_bits);  // dịch phải 8 bit
    if (v > 127)  v = 127;
    if (v < -128) v = -128;
    return (int8_t)v;
}
```
 
**Kết quả so sánh:** Dạng sóng C model và MATLAB **trùng khít** (sai số làm tròn < 1 LSB), xác nhận mô hình fixed-point chính xác.
 
---
 
## IV. Thiết kế phần cứng Verilog HDL
 
### 4.1 Tổng quan ba kiến trúc
 
| Kiến trúc | Nguyên lý | Chu kỳ/mẫu | Tài nguyên |
|-----------|-----------|:----------:|-----------|
| **MAC** | Nhân cộng tích lũy trực tiếp | 1 | Cao (DSP blocks) |
| **SDA** | DA tuần tự, 1 bit/clock | 9 | Thấp nhất |
| **PDA** | DA song song, 2 bit/clock | 5 | Trung bình |
 
---
 
### 4.2 MAC8FIR — Nhân cộng tích lũy
 
**File:** `MAC8FIR.v`
 
#### Nguyên lý
 
Khai thác tính **đối xứng hệ số**: cộng trước cặp mẫu cách đều, sau đó nhân với hệ số chung → giảm 8 phép nhân còn **8 phép cộng + 8 phép nhân**.
 
```
y(n) = (x[6]+x[7])×C7 + (x[5]+x[8])×C6 + (x[4]+x[9])×C5 + (x[3]+x[10])×C4
     + (x[2]+x[11])×C3 + (x[1]+x[12])×C2 + (x[0]+x[13])×C1 + (X+x[14])×C0
```
 
#### Port Interface
 
| Port | Hướng | Width | Mô tả |
|------|-------|:-----:|-------|
| `clk` | input | 1 | Clock hệ thống (50 MHz) |
| `RstN` | input | 1 | Reset active-low bất đồng bộ |
| `X` | input | 8 | Mẫu đầu vào (signed int8) |
| `Yn` | output | 16 | Kết quả lọc (signed int16) |
 
#### Hệ số lọc cứng (localparam)
 
```verilog
localparam signed C0 = -8'd1;   // h(0) = h(15)
localparam signed C1 = -8'd1;   // h(1) = h(14)
localparam signed C2 =  8'd0;   // h(2) = h(13)
localparam signed C3 =  8'd4;   // h(3) = h(12)
localparam signed C4 =  8'd13;  // h(4) = h(11)
localparam signed C5 =  8'd25;  // h(5) = h(10)
localparam signed C6 =  8'd37;  // h(6) = h(9)
localparam signed C7 =  8'd45;  // h(7) = h(8)
```
 
#### Hoạt động
 
```verilog
always @(posedge clk or negedge RstN) begin
    if (!RstN) begin
        // Reset thanh ghi trễ Xn[0..14] và yn về 0
    end else begin
        // 1. Tính y(n) = MAC với cộng cặp đối xứng
        yn = (Xn[6]+Xn[7])*C7 + (Xn[5]+Xn[8])*C6 + ...;
 
        // 2. Dịch thanh ghi trễ (shift pipeline)
        Xn[14]=Xn[13]; Xn[13]=Xn[12]; ... Xn[0]=X;
    end
end
```
 
- Kết quả `Yn` có sẵn **sau 1 clock** khi có mẫu mới (throughput = 1 mẫu/clock)
- Quartus tự động map phép `*` vào **DSP Blocks** của Cyclone V
#### Sơ đồ luồng dữ liệu
 
```
X ─→ [PSC]─→ Xn[0] ─→ Xn[1] ─→ ... ─→ Xn[14]
               │  │       │  │            │   │
               +──┘       +──┘    ...    +───┘
               ↓          ↓              ↓
              ×C0        ×C1   ...      ×C7
               │          │              │
               └──────────┴──────┬───────┘
                                 ↓
                            [Cộng dồn]
                                 ↓
                               Yn [>>1]
```
 
---
 
### 4.3 SDA8FIR — Serial Distributed Arithmetic
 
**File:** `SDA8FIR.v`
 
#### Nguyên lý
 
Thay vì nhân, dùng **LUT tra cứu tổng hệ số** và **cộng tuần tự từng bit** từ LSB đến MSB. Mỗi mẫu cần **9 clock** để xử lý (8 bit dữ liệu + 1 clock flush carry-out MSB).
 
**Công thức tích lũy:**
 
```
y(n) = Σ_{b=0}^{6} LUT[s_b] × 2^b  −  LUT[s_7] × 2^7
```
 
Trong đó `s_b` là tổ hợp bit thứ b của 8 cặp mẫu đối xứng → địa chỉ LUT.
 
#### Submodule `seriAdd` (Cộng tuần tự có nhớ)
 
```verilog
module seriAdd(
    input  clk, RstN,
    input  Ci,           // Carry-in từ chu kỳ trước
    input  A, B,         // Bit đơn từ hai mẫu đối xứng
    output S,            // Sum bit
    output Co            // Carry-out → Ci chu kỳ sau
);
    // S, Co = A + B + Ci  (full adder có đồng bộ clock)
```
 
- 8 instance `seriAdd` xử lý **song song 8 cặp mẫu**
- `Ci` được cập nhật mỗi clock theo `Co` của cycle trước
#### Hai bảng LUT
 
```
lut1[4-bit] = tổng hệ số C0,C1,C2,C3 (4 hệ số thấp)
lut2[4-bit] = tổng hệ số C4,C5,C6,C7 (4 hệ số cao)
val = lut1[s[3:0]] + lut2[s[7:4]]    ← cộng tức thì (combinational)
```
 
**Giá trị LUT1 (16 entry):**
 
```
[0]=0, [1]=-1, [2]=-1, [3]=-2,
[4]=0, [5]=-1, [6]=-1, [7]=-2,
[8]=4, [9]=3,  [10]=3, [11]=2,
[12]=4,[13]=3, [14]=3, [15]=2
```
 
**Giá trị LUT2 (16 entry):**
 
```
[0]=0,  [1]=13, [2]=25, [3]=38,
[4]=37, [5]=50, [6]=62, [7]=75,
[8]=45, [9]=58, [10]=70,[11]=83,
[12]=82,[13]=95,[14]=107,[15]=120
```
 
#### Cấu trúc RAM 144-bit
 
```
ram[143:0] = 16 mẫu × 9 bit/mẫu
             (8 bit dữ liệu + 1 bit sign extension ở MSB)
 
ram[143:135] ← {X[0],X[1],...,X[7], X[7]}  (MSB first + sign ext)
```
 
#### Luồng xử lý SDA (9 clock)
 
```
Clock 0 (b=0):  y_tmp += val            (LSB, không dịch)
Clock 1 (b=1):  y_tmp += val << 1
Clock 2 (b=2):  y_tmp += val << 2
...
Clock 7 (b=7):  y_tmp += val << 7
Clock 8 (b=8):  y_tmp -= val << 8       (MSB bit dấu → TRỪ)
                Yn = y_tmp; ram dịch phải 9 bit; reset b, Ci
```
 
#### Sơ đồ SDA
 
```
Xn[0..15] (9-bit mỗi thanh ghi, MSB first)
    │
    ├─[seriAdd0]: T0+T15 → s[0]─→┐
    ├─[seriAdd1]: T1+T14 → s[1]─→│
    ├─[seriAdd2]: T2+T13 → s[2]─→│ lut1[s[3:0]]─→ val ─→ << b ─→ y_tmp
    ├─[seriAdd3]: T3+T12 → s[3]─→│ lut2[s[7:4]]─→
    ├─[seriAdd4]: T4+T11 → s[4]─→│
    ├─[seriAdd5]: T5+T10 → s[5]─→│
    ├─[seriAdd6]: T6+T9  → s[6]─→│
    └─[seriAdd7]: T7+T8  → s[7]─→┘
                                     9 clocks → Yn
```
 
---
 
### 4.4 PDA8FIR — Parallel Distributed Arithmetic (2-bit)
 
**File:** `PDA8FIR.v`
 
#### Nguyên lý
 
Xử lý **2 bit đồng thời** mỗi clock → giảm còn **5 clock/mẫu** (4 cặp bit chẵn/lẻ + 1 clock MSB âm).
 
- **Lớp 1** (bit b): 8 `Add` tổ hợp → sum `s[7:0]` → LUT → `tmp1`
- **Lớp 2** (bit b+1): 8 `Add` tổ hợp → sum `s[15:8]` → LUT → `tmp2`
```
Mỗi clock: y_tmp += (tmp1 << b) + (tmp2 << (b+1))
```
 
#### Submodule `Add` (Cộng tổ hợp không đồng bộ)
 
```verilog
module Add(
    input  RstN, Ci, A, B,
    output reg S, Co
);
    always @(*) begin
        if (!RstN) {Co,S} = 2'b0;
        else       {Co,S} = Ci + A + B;  // cập nhật tức thì
    end
endmodule
```
 
> **Khác SDA:** `Add` là **combinational** (không clock) → kết quả có ngay khi input thay đổi → hai lớp adder chạy đồng thời trong 1 clock.
 
#### Kết nối hai lớp Adder
 
```
Lớp 1 (bit b,  b=0,2,4,6):
  add0..add7 → Ci[0..7] → s[7:0]  → LUT → tmp1
                    ↓ Ci1[0..7]
Lớp 2 (bit b1, b1=1,3,5,7):
  add8..add15 → Ci1[0..7] → s[15:8] → LUT → tmp2
                    ↓ Co[0..7]  (để Ci kế tiếp)
```
 
#### Luồng xử lý PDA (5 clock)
 
```
Clock 0 (b=0, b1=1): y_tmp +=  tmp1       + (tmp2 << 1)
Clock 1 (b=2, b1=3): y_tmp += (tmp1 << 2) + (tmp2 << 3)
Clock 2 (b=4, b1=5): y_tmp += (tmp1 << 4) + (tmp2 << 5)
Clock 3 (b=6, b1=7): y_tmp += (tmp1 << 6) + (tmp2 << 7)
Clock 4 (b=8):        y_tmp -= (tmp1 << 8)     ← bit dấu → TRỪ
                      Yn = y_tmp; ram dịch phải 9 bit; reset
```
 
---
 
### 4.5 So sánh chi tiết ba kiến trúc
 
#### Điểm tương đồng
 
- Cả ba đều nhận `X` (int8) và xuất `Yn` (int16)
- Reset bất đồng bộ active-low (`negedge RstN`)
- Dữ liệu thanh ghi trễ lưu theo **MSB-first + sign extension** trong SDA/PDA
- Tất cả dùng **hệ số lọc đối xứng** (giảm một nửa phép nhân/LUT)
#### Điểm khác biệt
 
| Tiêu chí | MAC | SDA | PDA |
|----------|:---:|:---:|:---:|
| Phép nhân | Trực tiếp | Không có (LUT) | Không có (LUT) |
| Adder type | — | `seriAdd` (clocked FF) | `Add` (combinational) |
| Số Adder instance | 0 | 8 | 16 |
| Clock/mẫu | **1** | **9** | **5** |
| Cần DSP block | ✅ Có | ❌ Không | ❌ Không |
| Phức tạp thiết kế | Thấp | Trung bình | Cao |
 
---
 
## V. Mô phỏng và thực hiện trên FPGA
 
### 5.1 Mô phỏng ModelSim
 
#### MAC — Kết quả mô phỏng
 
- Tín hiệu ra `Yn` có dạng analog hình sin → xác nhận chính xác
- Kết quả xuất hiện **ngay sau 1 clock** khi có mẫu mới
- Reset tích cực mức thấp hoạt động đúng
#### SDA — Kết quả mô phỏng
 
- Mỗi mẫu cần đúng **9 xung clock** để tính toán (8 bit + 1 flush carry)
- Tín hiệu `b` đếm từ 0 → 8, sau đó reset và dịch `ram`
- Kết quả `Yn` đúng với giá trị mong đợi từ C model
#### PDA — Kết quả mô phỏng
 
- Mỗi mẫu cần đúng **5 xung clock** (4 cặp 2-bit + 1 clock MSB)
- Tín hiệu `s[15:0]` là kết quả 16 adder song song
- `tmp1`, `tmp2` tổ hợp từ LUT cho hai bit song song
### 5.2 Chạy trên FPGA — Signal Tap Logic Analyzer
 
**Cấu hình chung cho cả ba thiết kế:**
 
| Tín hiệu | Kết nối board |
|----------|--------------|
| `clk` | `CLOCK_50` (50 MHz) |
| `RstN` | `KEY[0]` |
| Trigger | Rising edge của `KEY[0]` (sau nhấn và thả) |
 
**Kết quả Signal Tap:**
- **MAC**: X[7:0] đầu vào và Yn[15:0] đầu ra hiển thị đúng dạng sóng sin, kết quả tức thì
- **SDA**: Có thể quan sát `b` đếm từ 0→8, `ram` dịch đúng sau mỗi 9 cycles
- **PDA**: `add[Y:0]` hiển thị kết quả cộng tổ hợp, `b` bước 2 đơn vị/clock
### 5.3 Kết quả tổng hợp Quartus Prime
 
| Thông số | MAC | PDA | SDA |
|----------|:---:|:---:|:---:|
| **f_max (MHz)** | 66.08 | 101.72 | 80.87 |
| **ALM** | 50 | 163 | 131 |
| **DSP Blocks** | **3** | 0 | 0 |
| **Số thanh ghi** | 153 | 199 | 209 |
| **Clock/mẫu** | **1** | **5** | **9** |
| **Throughput (Msps @f_max)** | ~66 | ~20.3 | ~9.0 |
 
> **Nhận xét:** MAC sử dụng DSP blocks chuyên dụng (chiếm tài nguyên cao). SDA/PDA không dùng DSP blocks, tần số cao hơn nhưng throughput thực tế thấp hơn do cần nhiều clock/mẫu.
 
---
 
## VI. Kết luận và đánh giá
 
### 6.1 Kết quả đạt được
 
- ✅ Thiết kế thành công bộ lọc FIR 16 hệ số thấp qua dùng cửa sổ Hamming
- ✅ Chuyển đổi chính xác floating-point → fixed-point 8-bit (Q1.7 cho tín hiệu, Q0.8 cho hệ số)
- ✅ C model khớp với MATLAB (sai số < 1 LSB)
- ✅ Ba kiến trúc Verilog HDL hoạt động đúng trong ModelSim và trên board thực
- ✅ Kết quả Signal Tap xác nhận đúng hành vi phần cứng
### 6.2 So sánh ba phương pháp
 
| Tiêu chí | MAC | SDA | PDA |
|----------|:---:|:---:|:---:|
| **Tần số cực đại** | 66.08 MHz | 80.87 MHz | **101.72 MHz** |
| **Tài nguyên ALM** | **50** (ít nhất) | 131 | 163 |
| **DSP Blocks** | 3 (cao) | 0 | 0 |
| **Throughput thực** | **Cao nhất** | Thấp nhất | Trung bình |
| **Độ phức tạp thiết kế** | Đơn giản | Trung bình | Phức tạp |
| **Phù hợp khi** | Tốc độ cao, có DSP | Tiết kiệm tài nguyên | Cân bằng tốc độ-tài nguyên |
 
### 6.3 Hạn chế
 
- ⚠️ Số ký tự hỗ trợ còn hạn chế, chưa hỗ trợ floating-point trực tiếp
### 6.4 Hướng cải thiện
 
- 🔧 Tăng số hệ số lọc (32/64 tap) qua tham số hóa module
- 🔧 Triển khai PDA 4-bit hoặc 8-bit để đạt throughput cao hơn
- 🔧 Thêm giao tiếp Avalon MM để tích hợp vào SoC Nios II
- 🔧 Chuyển sang non-blocking assignment (`<=`) cho toàn bộ FSM
---
