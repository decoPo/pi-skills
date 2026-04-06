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
├── SW/DCM_test/               C# WinForms test application
├── HW/                        Altium PCB design files
├── doc/                       Datasheets, schematics, protocol diagrams
└── process/                   EN 62304 lifecycle docs (SDMP, backlog, SVP/SVR, checklists)
```

## Architecture

```
Host PC (GUI) <──USB CDC──> STM32F405/F407 (LaserBoard v3.2/v3.3) ──> CO2 Laser + Galvo XY + Sensors
```

- **MCU**: STM32F405RG (production board) / STM32F407VG (discovery dev board)
- **Build**: CMake + Ninja + arm-none-eabi-gcc (VSCode, migrated from STM32CubeIDE in FW-111)
- **No RTOS**: Bare-metal super loop with cooperative task scheduling
- **Current FW version**: v3.0.8

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
- Current task range: FW-100..FW-117, HW-101..HW-102

## Known Issues / Technical Debt

- `HAL_Delay()` calls in main loop (task_digitalInputs.c) – blocks bare-metal loop
- Water level bypassed: `sprintf(ts, "LEVEL,%d;", 0)` hardcoded
- `srand(time(NULL))` on embedded – time() likely returns 0 always
- Duplicate `.timmo` initialization in laserProperties (5 overwritten by 1)
- Dead code: `firePattern()`, `stopFire()`, old `Fractional_Firing_Stop()`
- Duplicate logic in `FractionalFireShape_FillArray()` (cross=0 vs cross=1)
- Many TODO comments need triage (FW-113)
- `str_response[2048]` – potential buffer overflow risk in protocol formatting
