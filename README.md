# FPGA Wordle Game

Hardware implementation of the five-letter Wordle game written in **VHDL** across dual **Xilinx Spartan-6 (`XC6SLX9-2TQG144C`)** FPGA boards, featuring 640x480 @ 60Hz VGA display output, PS/2 keyboard input, and an inter-board synchronous serial protocol.

## System Architecture

```text
+-------------------------------------------------------------------+
|                        FPGA WORDLE SYSTEM                         |
+-------------------------------------------------------------------+

+------------------------+   Serial Link   +------------------------+
|        FPGA 1          | <-------------> |        FPGA 2          |
|      Game Logic        |   (2.5 MHz)     |    Display & Input     |
+------------------------+                 +------------------------+
  |-- word_rom_20mhz.vhd                     |-- vga_controller_20mhz.vhd (640x480 @ 60Hz)
  |-- word_comparator_20mhz.vhd              |-- display_renderer_3bit_rgb_Version2.vhd
  |-- serial_receiver.vhd                    |-- char_rom.vhd & ps2_keyboard_20mhz.vhd
  '-- serial_transmitter.vhd                 '-- clock_generator.vhd (20 MHz -> 25 MHz DCM)
```

### FPGA 1: Game Logic Board (`fpga1_logic/`)
- **Top-level:** `fpga1_top_serial.vhd` (constraints: `fpga1_constraints_final.ucf`)
- **Word ROM (`word_rom_20mhz.vhd`):** Stores 16 five-letter target words (`40-bit` ASCII encoding per word).
- **Word Comparator (`word_comparator_20mhz.vhd`):** Evaluates player guesses against the target word and encodes 5x3-bit color feedback (`15 bits`).
- **Inter-board Serial:** Receives 40-bit guesses and transmits 15-bit evaluation results.

### FPGA 2: Display & Input Board (`fpga2_vga/`)
- **Top-level:** `fpga2_top_3bit_rgb_Version2.vhd` (constraints: `fpga2_constraints_3bit_rgb_Version2.ucf`)
- **Clock Generator (`clock_generator.vhd`):** Uses Spartan-6 `DCM_SP` to derive a 25 MHz pixel clock from the 20 MHz onboard oscillator.
- **VGA Controller & Renderer:** Drives 640x480 @ 60Hz negative-sync timing (`vga_controller_20mhz.vhd`) and renders the 6x5 game grid (`display_renderer_3bit_rgb_Version2.vhd`) with an 8x8 font ROM (`char_rom.vhd`).
- **PS/2 Keyboard (`ps2_keyboard_20mhz.vhd`):** Decodes A-Z, Backspace, and Enter scancodes.
- **Standalone Mode (`fpga2_standalone_working.vhd`):** Tests VGA display + keyboard input independently without FPGA 1.

## Hardware Requirements

- 2x Spartan-6 XC6SLX9 FPGA boards (20 MHz onboard oscillator)
- VGA monitor (640x480 @ 60Hz) and VGA cable (FPGA 2 `K4` connector)
- PS/2 keyboard (FPGA 2 `K3` connector)
- Inter-board jumper wiring connecting `K1` serial/handshake lines and `K2` game status lines with common ground
- Xilinx ISE 14.7 + iMPACT USB JTAG programmer

## Project Structure

```text
fpga-wordle/
├── README.md
├── ARCHITECTURE.md                                  # Detailed hardware & protocol architecture
├── DEVELOPMENT_NOTES.md                             # VHDL & VGA debugging notes
├── serial_transmitter.vhd                           # Shared synchronous serial TX
├── serial_receiver.vhd                              # Shared synchronous serial RX
├── serial_communication_tb.vhd                      # VHDL simulation testbench
├── fpga1_logic/
│   ├── fpga1_top_serial.vhd
│   ├── fpga1_constraints_final.ucf
│   ├── word_rom_20mhz.vhd
│   └── word_comparator_20mhz.vhd
└── fpga2_vga/
    ├── fpga2_top_3bit_rgb_Version2.vhd
    ├── fpga2_constraints_3bit_rgb_Version2.ucf
    ├── fpga2_standalone_working.vhd
    ├── fpga2_standalone_working.ucf
    ├── clock_generator.vhd
    ├── vga_controller_20mhz.vhd
    ├── display_renderer_3bit_rgb_Version2.vhd
    ├── char_rom.vhd
    ├── ps2_keyboard_20mhz.vhd
    └── archive/                                     # Incremental VGA bring-up test benches
```

## Build & Simulation

### 1. Build FPGA 1 (Game Logic)
1. Open Xilinx ISE and create a project targeting **Spartan-6 XC6SLX9-2TQG144C**.
2. Add `fpga1_logic/*.vhd`, `serial_receiver.vhd`, `serial_transmitter.vhd`, and `fpga1_logic/fpga1_constraints_final.ucf`.
3. Set `fpga1_top_serial` as top-level entity, synthesize, generate bitstream, and program Board 1 via iMPACT.

### 2. Build FPGA 2 (Display & Input)
1. Create a second ISE project with `fpga2_vga/*.vhd` (excluding `archive/` and `fpga2_standalone_*`), `serial_receiver.vhd`, `serial_transmitter.vhd`, and `fpga2_vga/fpga2_constraints_3bit_rgb_Version2.ucf`.
2. Set `fpga2_top_3bit_rgb_Version2` as top-level entity, synthesize, generate bitstream, and program Board 2.
3. Verify `led_locked` (`P82`, LED L0) turns ON to confirm DCM clock lock.

### 3. Simulation & Standalone Testing
- **Serial Protocol Testbench:** Simulate `serial_communication_tb.vhd` in ISim/ModelSim to verify 40-bit guess and 15-bit color frame transfers.
- **Standalone VGA + Keyboard Test:** Synthesize `fpga2_standalone_working.vhd` with `fpga2_standalone_working.ucf` to validate VGA rendering and PS/2 input without FPGA 1 connected.

## VGA Display & Pin Summary

### 3-bit RGB Color Encoding
| Color | RGB | Hex | Usage |
| :--- | :---: | :---: | :--- |
| Black | `000` | `0` | Background |
| Green | `010` | `2` | Correct letter, correct position |
| Yellow | `110` | `6` | Correct letter, wrong position |
| Magenta | `101` | `5` | Letter not in word (replaces gray in 3-bit RGB) |
| White | `111` | `7` | Cell borders and characters |

### Key Pin Assignments (Spartan-6 XC6SLX9-2TQG144C)
| Board | Signal | Pin | Description |
| :--- | :--- | :---: | :--- |
| FPGA 1 & 2 | `clk` | `P123` | 20 MHz onboard oscillator |
| FPGA 1 & 2 | `rst` | `P51` | Reset push button (`PB5`) |
| FPGA 2 | `vga_hsync` / `vga_vsync` | `P126` / `P131` | VGA sync signals (`K4`) |
| FPGA 2 | `vga_r` / `vga_g` / `vga_b` | `P133` / `P137` / `P139` | 3-bit RGB output (`K4`) |
| FPGA 2 | `ps2_clk` / `ps2_data` | `P124` / `P127` | PS/2 keyboard (`K3`) |
| FPGA 2 | `led_locked` | `P82` | DCM clock lock indicator (`L0`) |

## Documentation

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — Hardware architecture, clock tree, and protocol specification
- [`DEVELOPMENT_NOTES.md`](DEVELOPMENT_NOTES.md) — Debugging logs, VGA timing fixes, and VHDL guidelines

---

## สรุปภาพรวมโปรเจกต์ (ภาษาไทย)

โปรเจกต์เกม Wordle บนบอร์ด **FPGA Spartan-6 (`XC6SLX9-2TQG144C`) จำนวน 2 บอร์ด** พัฒนาด้วยภาษา **VHDL** และเครื่องมือ **Xilinx ISE** โดยแยกหน้าที่การทำงานระหว่างบอร์ดหลักและบอร์ดแสดงผล:

- **FPGA บอร์ดที่ 1 (Game Logic):** เก็บคลังคำศัพท์ 5 ตัวอักษร (`word_rom_20mhz.vhd`), เปรียบเทียบคำทายและคำนวณรหัสสี (`word_comparator_20mhz.vhd`) และสื่อสารผ่านโปรโตคอล Serial แบบ Synchronous
- **FPGA บอร์ดที่ 2 (Display & Input):** สร้าง Pixel Clock 25 MHz จาก Clock 20 MHz ผ่าน DCM (`clock_generator.vhd`), ขับสัญญาณ VGA 640x480 @ 60Hz แบบ 3-bit RGB (`vga_controller_20mhz.vhd` และ `display_renderer_3bit_rgb_Version2.vhd`) และรับค่าจากคีย์บอร์ด PS/2 (`ps2_keyboard_20mhz.vhd`) พร้อม Testbench จำลองการรับ-ส่งข้อมูล (`serial_communication_tb.vhd`)
