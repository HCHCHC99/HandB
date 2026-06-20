# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Motor control system based on HC32F460 (Cortex-M4) with RS485/Modbus RTU communication. Controls DC motors with Hall sensor feedback, over-current/voltage detection, rotation angle limiting, and fault handling.

All source code lives under `T_v.4.36.15_20260428/HC32F460_DDL_Rev3.3.0/`.

## Build System

- **Primary IDE:** Keil MDK (uVision 5)
- **Project file:** `T_v.4.36.15_20260428/HC32F460_DDL_Rev3.3.0/projects/ev_hc32f460_lqfp100_v2/template/MDK/template.uvprojx`
- **MCU:** HC32F460xE (512KB Flash) — linker script is `template/MDK/config/linker/HC32F460xE.sct`
- **Debug probe:** JLink (Cortex-M4)
- **Build output:** `template/MDK/output/debug/` (`.axf`, `.hex`, `.bin`, `.map`)
- Open the `.uvprojx` in Keil, click Build (F7), or use `UV4.exe -r template.uvprojx -o output.txt` from command line
- IAR EWARM and Eclipse/GCC project files also exist under `template/EWARM/` and `template/GCC/` but are not actively maintained

## Source Tree

```
projects/ev_hc32f460_lqfp100_v2/
├── Adp/          # Hardware adapter layer (rs485, PWM, ADC, DMA, GPIO, timers, flash)
├── App/          # Application logic (motor control, comm, fault, realtime, project glue)
├── Dev/          # Device drivers (motor, ADC, hall, voltage, sensor, EventBus, DeviceManager)
├── Utils/        # Utilities (ring_buf, msg_queue, lock, param_manager, TickTimer)
├── RTT/          # SEGGER RTT debug output
└── template/
    ├── source/   # main.c, main.h, hc32f4xx_conf.h
    └── MDK/      # Keil project, startup, linker scripts, JLink config
```

### Key files

- **`App/App_Motor_Project.c/h`** — Glue layer: `ESystem_Init()` registers all 16 devices with DeviceManager, sets up EventBus subscribers, and initializes the simulation struct. `ESystem_MainLoop()` calls `DeviceManager_Update()` and runs the simulation tick.
- **`template/source/main.c`** — Entry point. Owns the init sequence (see below), the 4-channel PWM globals, and the main `while(1)` loop.

## Initialization Sequence (main.c)

```
Hardware_Init()           # Clocks, GPIO, NVIC, peripherals
App_Comm_Init(&cfg)       # 4-layer comm stack (RS485 -> Comm_HAL -> Modbus -> App)
ESystem_Init()            # DeviceManager, EventBus subscribers, simulation struct
FaultHandler_Init()       # Subscribes to TOPIC_VOLTAGE_ALARM + TOPIC_CURRENT_ALARM
Motor_Pwm_Init()          # 4-channel PWM on PB6-PB9, 20kHz, active-low
EventBus_Enable()         # Replays deferred events, enables real-time publish

while(1):
    ESystem_MainLoop()    # DeviceManager_Update() + simulation tick
    App_Comm_Poll()       # Comm_HAL_RecvFrame -> ModbusRTU_ProcessFrame -> callbacks
    PWM_Update() x4       # Reload PWM duty registers
    Param_PrintAllValues()# Debug: prints all params via RTT every loop
```

When adding a new module, insert its init between `ESystem_Init()` and `EventBus_Enable()`. If it subscribes to EventBus topics, init after `ESystem_Init()` so the topics are registered.

## PWM Hardware Configuration

- **MCU pins:** PB6 (CH1), PB7 (CH2), PB8 (CH3), PB9 (CH4)
- **Timer:** TMRA_4, sawtooth up-counting mode
- **Frequency:** 20kHz (period = 6000 counts)
- **Polarity:** Active-low (`PWM_ACTIVE_LOW`)
- **Global instances:** `g_motor_pwm_ch1` through `g_motor_pwm_ch4` (declared in main.c)
- PWM output is enabled at init; duty is controlled by `PWM_SetDuty()` calls from the motor arbitration layer

## Debug Logging (SEGGER RTT)

All debug output goes through SEGGER RTT, **not** UART. View with JLink RTT Viewer or similar.

**Multi-channel + color-coded logging** (`RTT/rtt_log.h`):

| Channel | Tag | Color | Purpose |
|---------|-----|-------|---------|
| LOG_CH_MAIN (0) | MAIN | Cyan | Main program flow |
| LOG_CH_USB (1) | USB | Green | USB module |
| LOG_CH_SENSOR (2) | SENSOR | Cyan | Sensor readings |
| LOG_CH_MOTOR (3) | MOTOR | Cyan | Motor control |
| LOG_CH_COMM (4) | COMM | Cyan | Communication stack |
| LOG_CH_UI (5) | UI | Cyan | User interface |

**Per-channel macros** follow the pattern: `MAIN_D()`, `MAIN_I()`, `MAIN_W()`, `MAIN_E()` (Debug/Info/Warn/Error). Same for `COMM_D()`, `MOTOR_D()`, etc.

Legacy macros `COMM_DBG()`, `HAL_DEBUG()` used in older code map to the COMM channel macros above.

## Simulation Mode

Enabled by default (`ENABLE_SIMULATION_MODE=1` in `App_Motor_Project.c`). The `g_sim` struct holds simulated hardware signals:
- Power output states (forward/reverse relays)
- Hall sensor limits (forward/reverse limit switches)
- IO button states
- ADC values (voltage, current)

The main loop calls `Sim_Update()` which detects changes on `g_sim` and publishes corresponding EventBus events. This allows full motor arbitration testing **without physical hardware** — the EventBus subscribers react identically to real and simulated events.

To test a specific scenario: modify `g_sim` fields (e.g., set `g_sim.adc_current` above the overcurrent threshold) and observe the fault/fault-clear cycle via RTT logs.

## Testing via Modbus

No unit test framework exists. All testing is done through Modbus RTU commands:

```bash
# Edit config at top of modbus_test_cmds.py, then:
py modbus_test_cmds.py
```

The script generates hex command frames for all Modbus operations. Copy the output hex to your RS485 master tool.

**Quick command reference** (node_id=1):

| Operation | Hex Frame |
|-----------|-----------|
| Read speed (0x2730) | `01 03 27 30 00 01 CRC` |
| Write START | `01 06 27 20 00 01 CRC` |
| Write FWD | `01 06 27 20 00 11 CRC` |
| Write STOP | `01 06 27 20 00 02 CRC` |
| Clear all faults | `01 06 27 40 00 00 CRC` |

## Known Code Issues (from Security Review)

The file `安全审查报告.md` documents 9 findings. Key issues to be aware of when modifying code:

1. **`Protocol_ModbusRtu.c:63-64`** — `ModbusRTU_SendResponse` writes CRC at `raw[len]` and `raw[len+1]` without checking `len + 2 <= 256`. Currently safe (max `len = 253`), but add a guard if protocol framing changes.
2. **`Protocol_ModbusRtu.c:254`** — `ModbusRTU_ProcessFrame` does `memcpy(frame, len)` without checking `len <= 256`. Currently safe because `Comm_HAL_RecvFrame` caps at 256, but add a guard if that contract changes.
3. **`App_Comm.c:183`** — Every single-register write calls `Param_Save()` which triggers Flash sector erase. Multi-register writes (0x10) batch correctly (one save for all registers), but rapid single-write sequences cause excessive Flash wear. Consider adding a debounce timer if single-write frequency increases.

## 4-Layer Communication Stack (strict top-down dependency)

```
App_Comm.c/h          — Register callbacks, motor control commands, Flash persistence
    ↓ calls
Protocol_ModbusRtu.c/h — CRC16, function codes (0x03/0x06/0x10), exception responses
    ↓ calls
Comm_HAL.c/h           — Ring buffers, frame timeout (3.5 char times), TX queue
    ↓ calls
rs485.c/h              — Pure hardware: USART4 + PA03 direction pin + ISRs
```

- **Each layer only calls the layer directly below it** — no cross-layer access
- Protocol layer knows nothing about register meanings; it calls `on_read`/`on_write`/`on_validate` callbacks
- Comm_HAL knows nothing about Modbus; it just assembles byte streams into frames by idle timeout
- All config aggregated into `App_Comm_Config_t` in main.c

## Key Architecture Patterns

- **EventBus** (`Dev/EventBus.h`): Publish/subscribe for inter-module communication. 14 topics including `TOPIC_MANUAL_IO` (motor commands), `TOPIC_VOLTAGE_ALARM`, `TOPIC_CURRENT_ALARM`, `TOPIC_RTURN_LIMIT`. Uses deferred publish — events published before `EventBus_Enable()` are queued as a bitmask and replayed on enable. Max 4 subscribers per topic, priority-ordered (0 = highest).

- **DeviceManager** (`Dev/device_manager.h`): Uniform device registry with time-sliced update scheduling. Each device gets an `update()` callback called at its configured interval (typically 1ms, voltage bus at 10ms). Mutex-protected access. 16 devices registered (motor arbitrator, power outputs, Hall switches, IO buttons, PWM, ADC channels, sensors).

- **Param Manager** (`Utils/param_manager.h`): Register-based parameter storage with Flash persistence. Parameters live in `g_AppParam` (type `AppParamRecord_t`). Read/write via `Param_ReadByReg()`/`Param_WriteByReg()`, save via `Param_Save()`. Uses wear-leveled Flash storage across sectors 56-62 with CRC32 validation, sequence numbers, and magic headers.

- **Motor arbitration** (`Dev/dev_motor.c`): Commands go through the motor arbitrator which decides whether to allow based on mode (auto/remote/manual). Uses a `block_fwd`/`block_rev` bitmask — multiple devices can independently block a direction (e.g., overcurrent adds `DEV_ID_OVERCUR_FWD`, limit switches add `DEV_ID_RTURN_FWD`). Arbitration re-evaluates on every block/unblock.

## Fault System

Fault bits (stored in `g_RealTimeData.fault_status`, readable via Modbus register 0x2740):

| Bit | Macro | Description |
|-----|-------|-------------|
| bit0 | `FAULT_BIT_OVERVOLTAGE` | Overvoltage |
| bit2 | `FAULT_BIT_OVERCURRENT` | Overcurrent |
| bit4 | `FAULT_BIT_OVERTEMP` | Overtemp |
| bit6 | `FAULT_BIT_UNDERVOLTAGE` | Undervoltage |
| bit5 | `FAULT_BIT_STALL` | Motor stall |

- `App_FaultHandler` subscribes to `TOPIC_VOLTAGE_ALARM` and `TOPIC_CURRENT_ALARM`, sets/clears fault bits in realtime data
- Overcurrent triggers **dual blocking**: dev_motor blocks forward via `DEV_ID_OVERCUR_FWD`, dev_rturn also blocks via `TOPIC_RTURN_LIMIT` for redundancy
- **Auto-clear mode**: faults clear automatically when the alarm condition resolves
- **Manual-clear mode**: faults persist until cleared via Modbus write to `REG_FAULT_STATUS` (0x2740), which calls `FaultHandler_ClearFault()`

## Control Commands (REG_CTRL_CMD = 0x2720)

Bits written via Modbus function 0x06 (single write only):

| Bit | Value | Description |
|-----|-------|-------------|
| bit0 | 0x0001 | START — enable RS485 control |
| bit1 | 0x0002 | STOP — disable RS485 control, stop motor |
| bit2 | 0x0004 | ESTOP — emergency stop (motor stops, control stays enabled) |
| bit3 | 0x0008 | RESET — `__NVIC_SystemReset()` after 200ms delay |
| bit4 | 0x0010 | FWD — forward (requires START first, uses `g_AppParam.target_speed`) |
| bit5 | 0x0020 | REV — reverse (requires START first, uses `g_AppParam.target_speed`) |

Typical sequence: START (0x0001) -> FWD (0x0011) -> STOP (0x0002)

## Important Constraints

- Flash erase/write cycles are limited (~10K-100K). Each `Param_Save()` triggers a sector erase. Avoid calling it per-register in multi-register writes (0x10) — the batch write path calls `Param_Save()` once for the entire batch
- Interrupt safety: Comm_HAL uses `__disable_irq()`/`__enable_irq()` around ring buffer reads. Keep critical sections short
- `ModbusRTU_ProcessFrame` expects `len <= 256`. Frame buffer is 256 bytes. Modbus RTU max frame is 256 bytes so this is safe
- RS485 direction pin polarity is configurable via `dir_polarity` (0 = high-TX/low-RX, 1 = low-TX/high-RX)
- Realtime data (`g_RealTimeData`) is RAM-only, **not persisted to Flash** — lost on power cycle
- Multi-register writes (0x10) reject batches that include `REG_CTRL_CMD` or `REG_FAULT_STATUS` — those must use single writes (0x06)

## Documentation

- `T_v.4.36.15_20260428/通信栈架构说明.md` — Full 4-layer communication stack explanation (Chinese)
- `T_v.4.36.15_20260428/电流控制逻辑说明.md` — Over-current detection flow, dual blocking, fault recovery (Chinese)
- `T_v.4.36.15_20260428/实时数据使用说明.md` — Real-time data register map and usage (Chinese)
- `T_v.4.36.15_20260428/modbus_test_cmds.py` — Generates Modbus RTU hex command frames for read (0x03), write (0x06), multi-write (0x10), fault clear, and control commands. Edit the config at the top of the script and run `py modbus_test_cmds.py`
