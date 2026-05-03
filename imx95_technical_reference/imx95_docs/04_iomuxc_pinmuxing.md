# NXP i.MX95 EVK — IOMUXC & Pin Muxing
> Phần 7 của tài liệu NXP i.MX95 Deep-Dive

---

## 7. Cấu Hình Phần Cứng Tầng Đáy: IOMUXC & Pin Muxing

### 7.1 Kiến Trúc IOMUXC Trên i.MX95

IOMUX Controller (IOMUXC) là IP block trung tâm điều khiển toàn bộ pin routing và PAD electrical characteristics của SoC.

```
i.MX95 IOMUXC Architecture:

  Physical Package Pins (1215 BGA balls trên i.MX9592)
       │
       ▼
  ┌──────────────────────────────────────────────────────────┐
  │   IOMUXC (I/O Multiplexer Controller)                   │
  │   Base Address: 0x443C_0000                              │
  │                                                          │
  │  ┌────────────────┐   ┌────────────────────────────────┐ │
  │  │  MUX_CTL regs  │   │  PAD_CTL regs                  │ │
  │  │  (Pad Mux)     │   │  (Pad Configuration)           │ │
  │  │                │   │                                 │ │
  │  │  For each pin: │   │  For each pin:                  │ │
  │  │  MUX_MODE[3:0] │   │  PULL_UP_DOWN_EN               │ │
  │  │  → selects     │   │  PULL_SELECT                   │ │
  │  │    1-of-8      │   │  DRIVE_STRENGTH[2:0]           │ │
  │  │    ALT funcs   │   │  SLEW_RATE                     │ │
  │  │  SION bit      │   │  OPEN_DRAIN                    │ │
  │  │  (input on)    │   │  INPUT_SCHMITT_EN              │ │
  │  └────────────────┘   └────────────────────────────────┘ │
  │                                                          │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │  SELECT_INPUT regs (Daisy Chain)                   │  │
  │  │  → Khi nhiều pins có thể là input của cùng 1 IP:  │  │
  │  │    UART1_RXD_SELECT_INPUT: chọn pin nào làm RXD   │  │
  │  └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
       │
       ▼
  Peripheral IP Blocks (I2C, SPI, SAI, UART, GPIO, ...)
```

**Register Map IOMUXC:**

```
IOMUXC Register Layout (0x443C_0000):

Offset  Register                      Description
──────────────────────────────────────────────────────────────
0x0000  IOMUXC_SW_MUX_CTL_PAD_0      Pin 0 mux control
0x0004  IOMUXC_SW_MUX_CTL_PAD_1      Pin 1 mux control
...
0x0NNN  IOMUXC_SW_MUX_CTL_PAD_N      Pin N mux control
(~350 MUX_CTL registers, 1 per pad)

After all MUX_CTL:
0xXXXX  IOMUXC_SW_PAD_CTL_PAD_0      Pin 0 PAD config
0xXXXX  IOMUXC_SW_PAD_CTL_PAD_1      Pin 1 PAD config
...
(~350 PAD_CTL registers, mirrored 1-to-1 with MUX_CTL)

After all PAD_CTL:
0xYYYY  IOMUXC_SELECT_INPUT_N        Daisy chain select inputs
(~150 SELECT_INPUT registers)

MUX_CTL Register Format (32-bit):
  Bit [3:0]  MUX_MODE   : 0=ALT0, 1=ALT1, ..., 7=ALT7, 5=GPIO
  Bit [4]    SION       : Software Input On
                          0=normal, 1=force input regardless of MUX_MODE
                          Phai set SION=1 khi IP cần loopback (e.g., I2C)

PAD_CTL Register Format (32-bit, i.MX95 specific):
  Bit [0]    PULL_UP_DOWN_VOLTAGE:
               0 = weak pull (100kΩ typical @ 1.8V)
               1 = strong pull (22kΩ typical @ 3.3V)
  Bit [1]    PULL_UP_DOWN_ENABLE:
               0 = keeper (maintains last driven value)
               1 = pull resistor enabled
  Bit [2]    PULL_SELECT:
               0 = pull down
               1 = pull up
  Bit [3]    OPEN_DRAIN:
               0 = push-pull output
               1 = open-drain output ← BẮT BUỘC cho I2C!
  Bit [4]    SCHMITT_TRIGGER_ENABLE:
               0 = CMOS input
               1 = Schmitt trigger (hysteresis, for I2C/SPI noise immunity)
  Bits [6:5] SLEW_RATE:
               0x0 = SLOW (±2ns rise/fall, low EMI) ← I2C, UART
               0x1 = MEDIUM
               0x2 = FAST  (±0.5ns, for SPI high speed)
               0x3 = ULTRA_FAST (clock outputs)
  Bits [9:7] DRIVE_STRENGTH:
               0x0 = HIGH_IMPEDANCE (Hi-Z, tristate)
               0x1 = 255_OHM
               0x2 = 105_OHM
               0x3 = 75_OHM
               0x4 = 85_OHM (Rds on)
               0x5 = 65_OHM ← Thường dùng cho I2C/SPI
               0x6 = 45_OHM
               0x7 = 40_OHM ← LPDDR5 signals, high speed
  Bit [10]   INPUT_ENABLE:
               0 = output only
               1 = input buffer enabled ← Phải set cho bất kỳ pin nào là input
```

### 7.2 Pinmux trong Linux: pinctrl subsystem

```c
/* drivers/pinctrl/freescale/pinctrl-imx.c */
/* Generic i.MX pinctrl driver base */

/* Pin descriptor structure: */
struct imx_pin {
    unsigned int pin;        /* Pin ID (index trong MUX_CTL table) */
    union {
        struct imx_pin_mmio mmio;
    } pin_conf;
};

struct imx_pin_mmio {
    u32 mux_mode;       /* MUX_CTL value: ALT0-7, SION bit */
    u32 input_reg;      /* SELECT_INPUT register offset (0 nếu không có) */
    u32 input_val;      /* SELECT_INPUT value */
    u32 pad_setting;    /* PAD_CTL value */
};

/* Khi kernel apply pinctrl state: */
static int imx_pmx_set_one_pin(struct imx_pinctrl *ipctl,
                                struct imx_pin *pin)
{
    struct imx_pin_mmio *pin_mmio = &pin->pin_conf.mmio;
    
    /* 1. Write MUX_CTL: chọn ALT function */
    writel(pin_mmio->mux_mode,
           ipctl->base + pin_mmio->mux_offset);
    
    /* 2. Write SELECT_INPUT (daisy chain) nếu cần */
    if (pin_mmio->input_reg) {
        writel(pin_mmio->input_val,
               ipctl->base + pin_mmio->input_reg);
    }
    
    /* 3. Write PAD_CTL: electrical characteristics */
    writel(pin_mmio->pad_setting,
           ipctl->base + pin_mmio->pad_offset);
}
```

### 7.3 Ví Dụ Thực Tế: DTS Node IOMUXC trên i.MX95

#### Case 1: I2C3 Bus (TAS5828M Amplifier, Audio Path)

```dts
/* arch/arm64/boot/dts/freescale/imx95-19x19-evk.dts */

/* ─── IOMUXC pinmux definitions ─── */
&iomuxc {
    pinctrl_lpi2c3: lpi2c3grp {
        fsl,pins = <
            /* MX95_PAD_GPIO_IO28__LPI2C3_SDA */
            /* Giải thích macro: */
            /* MX95_PAD_GPIO_IO28 = tên physical pad */
            /* LPI2C3_SDA = ALT function được chọn */
            /* Macro expand thành: <mux_reg mux_val pad_reg pad_val input_reg input_val> */

            MX95_PAD_GPIO_IO28__LPI2C3_SDA    0x40000b9e
            /*                                │├──────── PAD_CTL value: 0x40000b9e
             *                                │  Bit[0]  PULL_VOLT  = 0 (weak 100kΩ)
             *                                │  Bit[1]  PUE        = 1 (pull enabled)
             *                                │  Bit[2]  PUS        = 1 (pull UP)
             *                                │  Bit[3]  ODE        = 1 (open drain ← I2C!)
             *                                │  Bit[4]  HYS        = 1 (Schmitt trigger)
             *                                │  Bits[6:5] SLEW     = 0 (SLOW)
             *                                │  Bits[9:7] DSE      = 5 (65Ω drive)
             *                                │  Bit[30] SION       = 1 (input on, loopback)
             *                                │  → 0x4000_0b9e:
             *                                │    bit30=1(SION), bit9-7=101(65Ω),
             *                                │    bit4=1(HYS), bit3=1(ODE),
             *                                │    bit2=1(PUS↑), bit1=1(PUE), bit0=0
             */
            
            MX95_PAD_GPIO_IO29__LPI2C3_SCL    0x40000b9e
            /*    ↑
             *    SCL cũng open-drain + pull-up + Schmitt
             *    I2C spec: cả SDA và SCL PHẢI là open-drain
             *    Pull-up resistor trên board: 2.2kΩ đến VDDIO (1.8V hoặc 3.3V)
             *    Nếu thiếu pull-up → I2C bus stuck HIGH, không có START condition
             */
        >;
    };

    /* ─── UART1 (Debug serial, 115200 bps) ─── */
    pinctrl_uart1: uart1grp {
        fsl,pins = <
            MX95_PAD_UART1_RXD__LPUART1_RX    0x31e
            /*                                 │├── PAD_CTL: 0x0000_031e
             *                                 │   Bit[0]  PULL_VOLT = 0
             *                                 │   Bit[1]  PUE       = 1 (pull enabled)
             *                                 │   Bit[2]  PUS       = 1 (pull up)
             *                                 │   Bit[3]  ODE       = 0 (push-pull ← UART)
             *                                 │   Bit[4]  HYS       = 1 (Schmitt)
             *                                 │   Bits[6:5] SLEW    = 0 (SLOW)
             *                                 │   Bits[9:7] DSE     = 6 (45Ω)
             *                                 │   SION=0 (không cần loopback cho UART RX)
             *                                 │   → 0x031e: PUE+PUS+HYS+DSE=6
             */
            
            MX95_PAD_UART1_TXD__LPUART1_TX    0x31e
            /*   TX: push-pull, pull-up (maintain HIGH khi idle)
             *   UART idle = MARK = logic '1' = HIGH
             *   Pull-up đảm bảo không float khi UART chưa drive
             */
        >;
    };

    /* ─── LPSPI5 (External ADC/DAC, 50MHz SPI) ─── */
    pinctrl_lpspi5: lpspi5grp {
        fsl,pins = <
            MX95_PAD_GPIO_IO18__LPSPI5_PCS0   0x3fe
            /*                                 │├── PAD_CTL: 0x0000_03fe
             *                                 │   Bit[0]  = 0 (weak pull)
             *                                 │   Bit[1]  PUE  = 1
             *                                 │   Bit[2]  PUS  = 1 (CS idle HIGH)
             *                                 │   Bit[3]  ODE  = 0 (push-pull ← SPI)
             *                                 │   Bit[4]  HYS  = 1
             *                                 │   Bits[6:5] SLEW = 1 (MEDIUM)  ← SPI cần nhanh hơn I2C
             *                                 │   Bits[9:7] DSE = 7 (40Ω)     ← impedance matching
             */
            
            MX95_PAD_GPIO_IO19__LPSPI5_SIN    0x3fe  /* MISO */
            MX95_PAD_GPIO_IO20__LPSPI5_SOUT   0x3fe  /* MOSI */
            MX95_PAD_GPIO_IO21__LPSPI5_SCK    0x3fe  /* SCLK: SLEW=MEDIUM, DSE=7 */
            /*   SPI CLK 50MHz → period = 20ns
             *   Rise time phải < 30% period = 6ns → SLEW=FAST hoặc MEDIUM + DSE=40Ω
             *   Nếu dùng SLOW: ringing, bit errors ở 50MHz
             */
        >;
    };

    /* ─── SAI3 (TAS5828 Audio I2S, Master mode) ─── */
    pinctrl_sai3: sai3grp {
        fsl,pins = <
            MX95_PAD_GPIO_IO12__SAI3_TX_DATA00  0x31e
            MX95_PAD_GPIO_IO13__SAI3_TX_SYNC    0x31e  /* LRCLK */
            MX95_PAD_GPIO_IO14__SAI3_TX_BCLK    0x31e  /* BCLK: SAI master drives */
            MX95_PAD_GPIO_IO15__SAI3_MCLK       0x31e  /* MCLK: 256fs = 12.288MHz @ 48kHz */
            /*   Audio: push-pull (không open-drain)
             *   SLOW slew rate để giảm EMI (audio clock noise → SNR degradation)
             *   DSE=6 (45Ω) cân bằng giữa drive strength và EMI
             */
        >;
    };

    /* ─── GPIO (TAS5828 Reset Pin, active low) ─── */
    pinctrl_tas5828_reset: tas5828resetgrp {
        fsl,pins = <
            MX95_PAD_GPIO_IO22__GPIO2_IO22      0x31e
            /* ALT function = GPIO (ALT5 thường là GPIO trên i.MX)
             * DSE=6, SLOW slew, pull-up: reset pin idle HIGH = not in reset
             * Driver sẽ drive LOW để reset TAS5828
             */
        >;
    };
};

/* ─── Peripheral nodes: reference pinctrl groups ─── */
&lpi2c3 {
    clock-frequency = <400000>;    /* Fast mode I2C: 400kHz */
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&pinctrl_lpi2c3>;  /* Apply khi device active */
    pinctrl-1 = <&pinctrl_lpi2c3_sleep>;  /* Low-power state */
    status = "okay";

    tas5828m: amplifier@4c {
        compatible = "ti,tas5828m";
        reg = <0x4c>;
        reset-gpios = <&gpio2 22 GPIO_ACTIVE_LOW>;
        #sound-dai-cells = <0>;
    };
};

&lpspi5 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lpspi5>;
    cs-gpios = <&gpio2 18 GPIO_ACTIVE_LOW>;  /* CS software-controlled */
    num-cs = <1>;
    status = "okay";
};
```

#### Case 2: Phân Tích Chi Tiết PAD Value 0x40000b9e

```
PAD_CTL value: 0x40000b9e = 0100_0000_0000_0000_0000_1011_1001_1110b

Bit 31 (reserved)    = 0
Bit 30 (SION)        = 1  → Software Input On (Force input enable)
                             Cần cho I2C: khi SDA là output, vẫn cần đọc ngược
                             để detect arbitration loss và ACK/NAK
Bit 29-10 (reserved) = 0000_0000_0000_0000_00
Bit 9-7 (DSE)        = 101 = 5 → Drive Strength: 65Ω
                             I2C fast mode 400kHz: 65Ω là hợp lý với 2.2kΩ pull-up
                             Quá mạnh (40Ω): over-drive, gây undershoot
                             Quá yếu (Hi-Z): sụt áp khi drive LOW
Bit 6-5 (SLEW)       = 00  = 0 → SLOW slew rate
                             I2C 400kHz: period 2.5μs, rise time target ~300ns
                             SLOW slew: ~2-5ns (pad intrinsic) + RC của bus
                             Bus RC dominant: C_bus * R_pullup ≈ 50pF * 2.2kΩ = 110ns ✓
Bit 4 (HYS)          = 1   → Schmitt Trigger enabled
                             BẮT BUỘC cho I2C: chống noise trên slow rise edges
                             Threshold: ~0.3*VDD (LOW), ~0.7*VDD (HIGH)
Bit 3 (ODE)          = 1   → Open Drain output
                             BẮT BUỘC cho I2C (spec: I2C devices chỉ pull-down)
                             Push-pull sẽ drive HIGH và SHORT với slave drive LOW
Bit 2 (PUS)          = 1   → Pull Up (not pull down)
Bit 1 (PUE)          = 1   → Pull Up/Down Enable (resistor enabled)
                             NOTE: Đây là weak internal pull (~100kΩ)
                             THƯỜNG KHÔNG ĐỦ cho I2C fast mode
                             → Cần external 2.2kΩ pull-up resistor trên board
                             Internal pull chỉ là backup / bus hold
Bit 0 (PULL_VOLT)    = 0   → Weak pull voltage reference

Tổng kết điện tử: I2C SDA/SCL pin sẽ là:
  - Open-drain output: chỉ có thể kéo xuống GND
  - External 2.2kΩ từ VDDIO (1.8V) kéo lên
  - Schmitt trigger input: không bị trigger nhầm bởi slow edges
  - Drive strength 65Ω: phù hợp với C_load ~50pF trên PCB
  - SION=1: kernel I2C driver luôn đọc SDA để detect NACK
```

#### Case 3: Cấu Hình Cho LPDDR5 Pads (ví dụ so sánh)

```dts
/* LPDDR5 DQ signals (nội bộ, không cần khai báo trong DTS) */
/* Được cấu hình tự động bởi DDR PHY training SPL */
/* PAD_CTL điển hình cho LPDDR5 DQ @ 3200MT/s: */

/* PAD_CTL = 0x0000_01E0 */
/* Bit[9:7] DSE = 0b011 = 3 → 75Ω (matched to LPDDR5 ZQ calibration) */
/* Bit[6:5] SLEW = 0b11 = 3 → ULTRA_FAST                               */
/*   → LPDDR5 @3200MT/s = 1.56GHz: rise time < 150ps, cần cực fast    */
/* Bit[4]   HYS = 0 → CMOS (LPDDR5 dùng VREF thay Schmitt)            */
/* Bit[3]   ODE = 0 → push-pull                                         */
/* Bit[1]   PUE = 0 → NO pull (LPDDR5 sử dụng ODT trong chip)         */
```

### 7.4 Debugging IOMUXC: Phát Hiện Lỗi Pinmux

```bash
# ─── Xem toàn bộ pinctrl state ───
adb shell cat /sys/kernel/debug/pinctrl/pinctrl-maps
# Output: mapping table, device → pin group → mux state

# Xem pin state của một controller
adb shell cat /sys/kernel/debug/pinctrl/443c0000.pinctrl/pinmux-pins
# pin 156 (GPIO_IO28): LPI2C3 SDA

# Xem PAD register dump (nếu debugfs exposed)
adb shell cat /sys/kernel/debug/pinctrl/443c0000.pinctrl/pins | grep -i "gpio_io28"
# pin 156 (GPIO_IO28) mux 0x443C0090 0x40000b9e

# ─── Đọc trực tiếp register qua devmem ───
# Cần biết offset của pad trong MUX_CTL table
# GPIO_IO28: thường là pin 156, MUX_CTL offset = 156 * 4 = 0x270
adb shell devmem 0x443C0270 32    # Đọc MUX_CTL
# 0x00000006 → ALT6 = LPI2C3_SDA, SION=0 (nếu chưa apply)

adb shell devmem 0x443C0270 32 0x40000006  # Set ALT6 + SION

# PAD_CTL base offset = MUX_CTL size (350 regs * 4 = 0x578)
adb shell devmem 0x443C0270+0x578 32       # Đọc PAD_CTL

# ─── gpio-watch: Monitor GPIO input state ───
adb shell cat /sys/kernel/debug/gpio
# Hiển thị tất cả GPIO với direction, value, label
# gpio-22 (TAS5828 RESET): out hi   ← không trong reset
# gpio-54 (INT từ amplifier): in  lo ← có interrupt pending

# ─── i2cdetect để verify I2C pinmux hoạt động ───
adb shell i2cdetect -y -r 3
# Kết quả bình thường (TAS5828 tại 0x4C):
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 40:                                     -- -- -- 4c --
# Nếu không thấy 0x4c → check pinmux (dùng devmem đọc MUX_CTL)
#                      → check clock lpi2c3 enabled
#                      → check reset GPIO state

# ─── Common pinmux bugs ───
# Bug 1: MUX_MODE = GPIO (ALT5) thay vì I2C → i2cdetect timeout
# Bug 2: ODE=0 cho I2C → SDA/SCL drive HIGH phá I2C bus
# Bug 3: SION=0 cho I2C → NACK detection fail → driver hang
# Bug 4: SLEW=FAST cho I2C 400kHz → quá nhiều EMI, ringing
# Bug 5: Thiếu external pull-up → VOH quá thấp (I2C FAIL ở fast mode)
# Bug 6: SELECT_INPUT sai → đúng ALT nhưng sai signal routing

# ─── SELECT_INPUT debugging ───
# Ví dụ LPI2C3_SDA có thể đến từ GPIO_IO28 HOẶC GPIO_IO06
# SELECT_INPUT register quyết định cái nào:
# IOMUXC_LPI2C3_IPP_IND_LPI_SDA_SELECT_INPUT: offset 0x700 (ví dụ)
# Value 0x0 = GPIO_IO06 là SDA input
# Value 0x1 = GPIO_IO28 là SDA input
adb shell devmem 0x443C0700 32   # Kiểm tra SELECT_INPUT
```

### 7.5 Bảng So Sánh PAD Settings Theo Protocol

```
Protocol    ODE  HYS  SLEW    DSE      PUE/PUS   Notes
──────────────────────────────────────────────────────────────────────
I2C 100kHz   1    1    SLOW   65-75Ω    1/UP    External 4.7kΩ pull-up
I2C 400kHz   1    1    SLOW   65Ω       1/UP    External 2.2kΩ pull-up
I2C 1MHz     1    1    MEDIUM 65Ω       1/UP    External 1kΩ pull-up
UART RX      0    1    SLOW   45-65Ω    1/UP    Pull-up for idle HIGH
UART TX      0    0    SLOW   45Ω       1/UP    Idle HIGH pullup
SPI CLK      0    1    MEDIUM 40-45Ω    0/-     No pull (matched)
SPI MOSI     0    0    MEDIUM 45Ω       0/-     No pull
SPI MISO     0    1    MEDIUM Hi-Z      1/UP    Weak pull (no contention)
SPI CS       0    0    SLOW   45Ω       1/UP    Idle HIGH = deselected
SAI I2S      0    1    SLOW   45-65Ω    1/-     Low EMI for audio
SAI MCLK     0    0    MEDIUM 40Ω       0/-     Clean clock, no pull
GPIO OUT     0    0    SLOW   65Ω       0/-     Depends on load
GPIO IN      0    1    -      Hi-Z      1/UP    Schmitt + pull-up
LPDDR5 DQ    0    0    ULTRA  40-75Ω    0/-     ZQ calibrated
USB DP/DM    0    0    FAST   40Ω       0/-     USB PHY controls

Ghi chú Drive Strength vs Load Capacitance (PCB rule of thumb):
  C_load < 20pF  → DSE = 105Ω (2) hoặc 75Ω (3)
  C_load 20-50pF → DSE = 65Ω (5) ← I2C, UART thông thường
  C_load 50-100pF → DSE = 45Ω (6) ← SPI với trace dài
  C_load > 100pF → DSE = 40Ω (7) ← LPDDR5, USB

Matching impedance:
  PCB trace 50Ω → DSE = 40-45Ω + series resistor ~10Ω on PCB
  PCB trace 75Ω → DSE = 75Ω
```

---

---

