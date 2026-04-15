---
name: co2-laser
description: CO2 medical laser firmware project context. STM32F4 bare-metal firmware controlling a CO2 laser with surgical and fractional modes, USB CDC communication with host PC, XY galvo scanner, safety interlocks. Use when working on the co2-stm32 repository.
---

# CO2 Laser Project Skill

## Rules

- **All outputs must be in English** (code, comments, commit messages, documentation, file content)
- Conversation with the user is in Slovak
- This is a **certified medical device** (EN 62304 Class B) – never mention certification authority requirements in any tracked file
- Commit messages must start with task ID: `FW-xxx short description` or `HW-xxx short description`
- Use feature branches (`FW-xxx_description`), squash merge to master, then delete branch
- Force-push is acceptable (solo developer)

### LASER SAFETY — CRITICAL

- **NEVER** send `STATE,1` (Ready) followed by pedal activation or any command sequence that could cause laser firing in automated scripts
- **NEVER** programmatically simulate pedal press — laser activation must ALWAYS be manual by the human operator
- Automated tests may ONLY: communicate, query state, set parameters, verify responses, read sensors
- Automated tests must VERIFY safety conditions (interlock closed, E-Stop released) but must NEVER bypass them
- **NEVER** send `DEVMD,1` in automated scripts without explicit user confirmation
- If a test requires laser firing, the script must PAUSE and PROMPT the user to manually press the pedal
- Fire hazard: laser can ignite paper/materials; injury hazard: laser can cause severe burns
- All test scripts must leave the system in IDLE state on exit (including on error/crash)

## Repository Structure

```
D:/mydoc/co2-stm32/
├── FW/                        Firmware (main focus)
│   ├── CMakeLists.txt         Top-level CMake (ACTIVE_PROJECT, BOARD_VARIANT)
│   ├── CMakePresets.json      Build presets (laser, laser_discovery, 01_led, 02_debug)
│   ├── cmake/                 Build system
│   │   ├── arm-gcc-toolchain.cmake   Cortex-M4F cross-compilation
│   │   ├── hal.cmake                 STM32F4 HAL static library
│   │   ├── usb.cmake                 USB CDC middleware + glue
│   │   └── project_template.cmake    stm32f4_add_executable() function
│   ├── Shared/                Common files for all projects
│   │   ├── inc/               stm32f4xx_hal_conf.h
│   │   ├── src/               system_stm32f4xx.c, syscalls.c, sysmem.c, hal_msp.c
│   │   ├── startup/           startup_stm32f405xx.s, startup_stm32f407xx.s
│   │   └── linker/            STM32F405RGTX_FLASH.ld, STM32F407VGTX_FLASH.ld
│   ├── Projects/
│   │   ├── laser/             Main application (Src/, Inc/, CMakeLists.txt)
│   │   ├── 01_led/            Test projects (each with Src/, Inc/, CMakeLists.txt)
│   │   ├── 02_debug/
│   │   └── 03_buttons..13_faults/
│   ├── driverlib/             Low-level peripheral drivers + USB_CDC/ glue
│   ├── HW/                    Board abstraction (hw.h → PCB/PCB_LASER3_2.h)
│   ├── STM32Cube_FW_F4_V1.28.1/  HAL drivers + USB middleware
│   └── .vscode/               VSCode config (settings, tasks, launch, IntelliSense)
│   └── tests/                 Python test scripts (smoke_test, fault_test, backup_calibration, capture_debug)
├── SW/DCM_test/               C# WinForms test application
├── HW/                        Altium PCB design files
├── doc/                       Datasheets, schematics, protocol diagrams
└── process/                   EN 62304 lifecycle docs (SDMP, SRS, SAD, backlog, SVP/SVR, STP, traceability)
```

## Architecture

```
Host PC (GUI) <──USB CDC──> STM32F405/F407 (LaserBoard v3.2/v3.3) ──> CO2 Laser + Galvo XY + Sensors
```

- **MCU**: STM32F405RG (production board) / STM32F407VG (discovery dev board)
- **Build**: CMake + Ninja + arm-none-eabi-gcc (VSCode, migrated from STM32CubeIDE in FW-111)
- **No RTOS**: Bare-metal super loop with cooperative task scheduling
- **Current FW version**: v3.0.10

## Operating Modes

### Surgical Mode
- Direct laser firing – set power (LASPW) and pulse modulation
- Modes: Single pulse (0), Repeated pulse (1), Continuous wave (2)
- TIM2 CH1 = laser power PWM, TIM3 = pulse timing, LaserSwitch GPIO = enable
- Pedal pressed = fire, released = stop

### Fractional Mode
- XY galvo scanner head attached
- Pattern generation (rectangle, circle, half-circle, L-triangle, R-triangle, dot)
- DAC1 + DAC2 for XY position, TIM7 + DMA for border drawing
- TIM3 = firing pulse, TIM4 = galvo settle time between dots
- Directions: spray (random), L→R up→down, L→R down→up, R→L up→down, R→L down→up

## Pin Mapping (LaserBoard v3.2 – PCB_LASER3_2.h)

### Safety Inputs (DigInputs indices)
| Index | Signal   | Pin  | Port  | Function                          |
|-------|----------|------|-------|-----------------------------------|
| 0     | Pedal    | PD2  | GPIOD | Fire trigger                      |
| 1     | Pedal2   | PB3  | GPIOB | Interlock safety loop             |
| 2     | CStop    | PC12 | GPIOC | Central/Emergency stop            |
| 3     | Vbus     | PA9  | GPIOA | USB power detect (auto-off 6s)    |
| 4     | Level    | PC11 | GPIOC | Coolant level (currently bypassed) |

### PWM Outputs
| Timer  | Channel | Pin  | Function        |
|--------|---------|------|-----------------|
| TIM1   | CH1     | PA8  | Fan / Vacuum    |
| TIM2   | CH1     | PA0  | Laser Power     |
| TIM2   | CH2     | PA1  | Aim Beam Power  |
| TIM3   | CH1     | PA6  | Laser Switch    |
| TIM5   | CH3     | PA2  | Peltier/Suction |
| TIM8   | CH1     | PC6  | Button LED      |
| TIM8   | CH2-4   | PC7-9| RGB LED         |
| TIM9   | CH1     | PA3  | Diode Switch (future) |

### Analog / DAC / Other
| Peripheral | Pin    | Function                  |
|------------|--------|---------------------------|
| ADC1 CH14  | PC4    | Temperature sensor 0      |
| ADC1 CH7   | PA7    | Temperature sensor 1      |
| ADC1 CH13  | PC3    | Temperature sensor 2      |
| DAC CH1    | PA4    | Galvo X position          |
| DAC CH2    | PA5    | Galvo Y position          |
| TIM11 IC   | PB9    | Flow meter (not implemented) |
| TIM12 IC   | PB14   | Moisture 555 (unused, reserved for HW-102 PWM monitoring) |
| USART3     | PB10/11| Debug UART                |
| USB OTG FS | PA11/12| USB CDC to host PC        |

### Control GPIOs
| Pin  | Port  | Function       |
|------|-------|----------------|
| PA10 | GPIOA | Heartbeat LED  |
| PB8  | GPIOB | Laser Unlock   |
| PB15 | GPIOB | Buzzer         |
| PB6  | GPIOB | Debug GPIO     |
| PC10 | GPIOC | Power Latch    |

## USB Protocol

**Format**: `ID;CMD,VAL;CMD,VAL\r\n`

### Commands (PC → STM32)
| Command | Values       | Description                    |
|---------|--------------|--------------------------------|
| STATE   | 0/1          | Set Standby/Ready              |
| OPEMD   | 0/1          | Set Surgical/Fractional mode   |
| LASPW   | 5-150        | Laser power (×0.1W)           |
| SURMD   | 0/1/2        | Single/Pulse/CW               |
| TIMON   | 10-1000      | Pulse ON time (ms)            |
| TIMOF   | 10-1000      | Pulse OFF time (ms)           |
| ENERG   | 10-150       | Fractional energy             |
| SHAPE   | 10000-52020  | Encoded: shape×10000 + x×100 + y |
| DIREC   | 0-4          | Firing direction              |
| DISTA   | 2-20         | Dot density                   |
| TIMMO   | 1-90         | Moving time                   |
| OVERL   | 1-5          | Overlay count                 |
| SUCPW   | 0-100        | Suction power %               |
| AIMPW   | 0-100        | Aim beam power %              |
| DEVMD   | 0/1          | Developer mode (bypass safety)|
| GETMD   | 0-5          | Query: surgical/fractional/general/calibration/metadata/API |
| HOWRU   | (none)       | Keepalive ping                |

### Responses (STM32 → PC)
| Response   | Meaning                           |
|------------|-----------------------------------|
| ALLOK      | Command accepted                  |
| EXECF,N    | Error code N (2=CStop, 8=Pedal, 256=Interlock, 512=InvalidCalib) |

### Keepalive
- PC must send HOWRU within 5000ms, otherwise STM32 goes to standby

## Calibration

- Stored in internal Flash with CRC32 verification
- Three sections: GCALI (general), FCALI (fractional power), SCALI (surgical power)
- Written via USB protocol with `{key:value,...}` format
- Read back and verified after write

## Task Tracking

- Backlog: `process/01_planning/Backlog.md` (single source of truth)
- Release history: `process/01_planning/Release_History.md`
- Task IDs: `FW-xxx` for firmware, `HW-xxx` for hardware
- Current task range: FW-100..FW-122, HW-101..HW-102

## Firmware Upgrade

- **Upgrade script**: `FW/tests/upgrade_fw.py` — automated backup → migrate → flash → restore → verify
- When `GeneralCali_t` struct changes, existing flash CRC becomes invalid → calibration must be migrated
- The upgrade script handles calibration migration (adds new fields with defaults)
- Can be run standalone by clients: `python upgrade_fw.py` (no PI console needed)
- Requires: ST-LINK (SWD) + USB CDC, STM32CubeProgrammer, built ELF
- `--dry-run` flag for backup+migrate without flashing

### Upgrade History

| From | To | Changes | Migration |
|------|------|---------|-----------|
| v3.0.9 | v3.0.10 | FW-116: RGB LED color in GCALI (rgbIdleR/G/B + _reserved) | Add RGB fields with gold default (0x67FF, 0x27FF, 0x0), fix 'lu' timestamp bug |

## Firmware Clone (Service)

- **Clone script**: `FW/tests/clone_fw.py` — download FW binary from working device, flash to others preserving calibration
- Use case: service of multiple devices with various older FW versions, no upgrade needed
- Two-phase workflow:
  - `clone_fw.py download --sn <LABEL>` — reads flash via SWD (`-u`), extracts calibration via CDC, captures boot banner via debug UART
  - `clone_fw.py clone <firmware.bin> --sn <TARGET_LABEL>` — backup target calib → flash → restore calib → verify
- `--sn` parameter = physical label on the machine/PCB (not stored in FW/calibration), used for file naming and traceability
- Downloads do NOT modify the source device (read-only: SWD upload + CDC GETMD queries)
- STM32CubeProgrammer CLI: `-u addr size file` for read, `-w file addr` for write

## Hardware Testing Procedures

### Two USB connections — CRITICAL DISTINCTION

| Connection | Port (typical) | Hardware | Survives MCU reset? | Purpose |
|------------|---------------|----------|--------------------|---------|
| **Debug UART** | COM21 | ST-Link VCP, USART3 (PB10/PB11) | **YES** — always connected, independent of MCU | Boot banner, log capture, fault output |
| **USB CDC (VCP)** | COM7 | MCU USB OTG FS (PA11/PA12) | **NO** — disconnects on MCU reset, re-enumerates after boot | Protocol communication, calibration R/W, commands |

**Key rules:**
- **Debug UART (ST-Link VCP)** — open BEFORE reset/flash to capture boot banner
- **USB CDC** — will **disconnect** on MCU reset (SWD flash, `-rst`, power cycle). Must wait for re-enumeration after boot (~2–5s). COM port number may change.
- Correct sequence for any flash/reset operation:
  1. Do all USB CDC work first (backup calib, read params) — MCU must be running
  2. Open debug UART (ST-Link VCP)
  3. Flash/reset via SWD — MCU resets, CDC disconnects, debug UART stays
  4. Capture boot banner from debug UART
  5. Wait for CDC to re-enumerate, then reconnect for post-flash operations

### Debug UART capture
- **Always open UART capture BEFORE flashing/resetting MCU** — boot banner is printed immediately after reset and will be missed otherwise
- Debug UART: ST-Link VCP (COM21 typical), 460800 baud, USART3 (PB10/PB11)
- Capture script: `FW/tests/capture_debug.py [COMxx]`
- Programmatic: open `serial.Serial(port, 460800)` first, then flash/reset via STM32_Programmer_CLI or GDB

### Flash + verify pattern
```python
# 1. Open debug UART first (ST-Link VCP — survives reset)
ser = serial.Serial('COM21', 460800, timeout=0.5)
ser.reset_input_buffer()
# 2. Flash (resets MCU automatically — CDC disconnects here)
subprocess.run(['STM32_Programmer_CLI.exe', '-c', 'port=SWD', '-w', 'laser.elf', '-rst'])
# 3. Read boot banner from debug UART
time.sleep(1)
while line := ser.readline():
    print(line.decode().rstrip())
# 4. Wait for CDC to re-enumerate before sending commands
```

## Fault Handling (FW-122)

- **Error_Handler()** — macro in `fault.h` captures `__FILE__`, `__LINE__`, `__func__` at call site
- **HardFault/MemManage/BusFault/UsageFault** — ASM wrappers in `stm32f4xx_it.c` extract stacked exception frame (R0-R3, R12, LR, PC, xPSR), C handler in `fault.c` decodes CFSR/HFSR bits
- **NMI** — detects Clock Security System (HSE) failure
- **Polling UART output** — direct `USARTx->DR` register write, no IRQ/HAL dependency, works in any fault context
- **DIV_0 trap** — `SCB->CCR |= DIV_0_TRP` — integer division by zero triggers UsageFault
- **Fault sub-handlers enabled** — `SCB->SHCSR` — MemManage/BusFault/UsageFault not escalated to HardFault
- **Stack canary** — 4×32-bit `0xDEADC0DE` at stack bottom, checked every super-loop iteration via `fault_checkStack()`
- **HW indication** — Buzzer ON + Heartbeat LED ON on fault (direct GPIO, no HAL)
- **Recovery** — WDT (1.5s) resets MCU after fault output
- **GDB test** — `fault_triggerTest(type)` kept in binary via `__attribute__((section))`, callable from GDB: `set $r0=N, set $pc=fault_triggerTest, continue`
- **Test script** — `FW/tests/fault_test.py` — injects 5 fault types via GDB, captures UART output, writes report to `reports/`
- **Design docs** — `FW/debugRefactor.md` (Part 4), `FW/audit.md` (items 29, 30)

## Debug & Logging (FW-103)

- **Log API**: `LOG_E(mod, fmt, ...)`, `LOG_I(...)`, `LOG_D(...)` — compile-time filtered via `LOG_LEVEL` CMake define
- **Output format**: `00012345 E MOD   message\r\n` — 8-digit timestamp, level char, 5-char module tag
- **Ring buffer**: 6000B async UART TX via `HAL_UART_Transmit_IT`, overflow counter
- **Compile-time**: Release = ERROR+INFO, Debug = all levels
- **Module tags**: MAIN, DIGI, USB, CMD, CALI, PROTO, UI, STATE

## Naming Convention (FW-104)

| Module | File(s) | Prefix |
|--------|---------|--------|
| USB transport | `usb.c` | `usb_` |
| Protocol parser | `proto_parse.c` | `parse_` |
| Command executor | `proto_cmd.c` | `cmd_` |
| Serialization | `proto_seriali.c` | `seriali_` |
| State machine | `state.c` | `state_` |
| Digital inputs | `digi.c` | `digi_` |
| UI | `ui.c` | `ui_` |
| Calibration | `cali.c` | `cali_` |
| Laser driver | `laser.c` | `laser_` |
| Geometry | `laser_geom.c` | `geom_` |
| Logging | `debug.c` | `log_` |
| Fault handler | `fault.c` | `fault_` |

## Security Audit (FW-121)

- **Audit doc**: `FW/audit.md` — 30 findings (25 fixed, 5 remaining)
- Critical fixes: NULL deref guards, buffer overflow SAFE_APPEND, VLA→fixed arrays, array bounds checks
- Remaining: weak srand, atoi validation, USB ISR race, global buffer, extern cleanup

## Known Issues / Technical Debt

- Water level bypassed: `sprintf(ts, "LEVEL,%d;", 0)` hardcoded
- `srand(time(NULL))` on embedded – time() likely returns 0 always
- `lastCalibrationAt` / `nextCalibrationAt` printf bug – shows "lu" instead of value (UInt64 format)
