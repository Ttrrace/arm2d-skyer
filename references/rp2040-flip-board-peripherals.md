# RP2040 Flip Board Peripheral Integration

This reference records interfaces observed in an RP2040 Flip / Crystal Mouse example project and defines conventions for future application services. The firmware project itself is not included in this skill. Verify its current branch, schematic, `application/bsp_cfg.h`, and Keil project before using any pins, addresses, timing, or APIs below on hardware.

## Layers

```text
deivers/                 Existing chip/register drivers (retain project spelling)
application/             Board initialization, tasks, caching, application semantics
main.c                   Peripheral service scheduling
arm_2d_scene_*.c/.h       Cached snapshot consumers; no direct bus access
```

Drivers own registers, GPIO, ADC, PWM, and bus transactions. Application code owns board pins, initialization order, sampling periods, debounce, caches, and events. A main loop or scheduler drives nonblocking poll/task operations. PFB callbacks must not perform I2C, repeated ADC sampling, `sleep_ms()`, `sleep_us()`, or frequent `printf()` calls.

## Recorded Interfaces

| Peripheral | Example implementation | Bus/pins | Main entrypoints | Success convention |
| --- | --- | --- | --- | --- |
| QMI8658 six-axis IMU | `deivers/drv_QMI8658.*`, `application/qmi8658c_task.*`, `application/qmi8658_motion.*` | I2C0, GPIO0/1, `0x6B` | `qmi8658c_init()`, `qmi8658_motion_poll()` | Initialization and low-level I/O: `1` |
| Power key | `main.c::power_task()` | Key GPIO9 active low; power hold GPIO2 | `power_task()` | `fsm_rt_on_going` |
| IR TX/RX | `deivers/drv_ir.*`, `application/ir_task.*` | TX GPIO28, RX GPIO22; default 38 kHz | `drv_ir_*()`, `ir_task()` | `bool`: accepted operation or valid data |
| GL5528 light sensor | `deivers/drv_light.*`, `application/light_task.*` | GPIO26 / ADC0 | `drv_light_read()`, `light_task()` | Returned structure; task currently prints only |
| Temperature/humidity | No dedicated `.c/.h` found in the example inventory | Verify schematic | No confirmed API | Do not invent one |
| BM8563 RTC | `deivers/bm8563.*`, `application/bm8563_task.*` | Shared I2C0, `0x51` | `bm8563_*()`, `bm8563_hander_init()` | `BM8563_OK == 0` |
| PAJ7620 gesture | `deivers/drv_paj7620.*` | Shared I2C0, `0x73` | `paj7620Init()`, `paj7620ReadReg()` | `0`; `0xFF` means chip-ID mismatch |
| DET402 buzzer | `deivers/drv_buzzer.*`, `application/buzzer_task.*` | GPIO23 / PWM | `drv_buzzer_*()`, `buzzer_task()` | Start operations generally return `bool` |

SD, TinyUSB MSC/HID, and display adapters belong to separate subsystems. Consult their existing modules and relevant references.

## Proposed Service Contract

The recorded drivers use inconsistent names, returns, and data exposure. Adapt them with thin services in `application/`; do not rewrite third-party chip drivers merely to standardize them. The following names are proposed conventions, not existing APIs:

```c
typedef enum board_peripheral_state_t {
    BOARD_PERIPHERAL_UNINITIALIZED = 0,
    BOARD_PERIPHERAL_READY,
    BOARD_PERIPHERAL_STALE,
    BOARD_PERIPHERAL_ERROR,
} board_peripheral_state_t;

typedef struct board_sample_meta_t {
    uint32_t timestamp_ms;
    uint32_t sequence;
    bool valid;
} board_sample_meta_t;

bool board_xxx_init(void);
bool board_xxx_poll(uint32_t current_ms);
bool board_xxx_get_latest(board_xxx_sample_t *sample);
board_peripheral_state_t board_xxx_get_state(void);
```

- `init()` initializes/probes once and returns `true` if available.
- `poll()` is repeatable without long blocking waits; it returns `true` only for a new sample/event.
- `get_latest()` copies the latest complete cached snapshot without hardware access; returns `false` without valid data.
- Each sample carries a millisecond timestamp, increasing sequence number, and validity. Consumers use `sequence` to detect changes.
- Do not relabel old data as new after failures. Use `STALE` after a service-defined timeout and `ERROR` for bus/chip errors.
- Reject null pointers unless an existing driver contract explicitly permits them.
- ISRs collect data, set flags, or advance short state machines; they do not perform I2C, formatted output, or Arm-2D drawing.

Use explicit units:

| Quantity | Recommended representation |
| --- | --- |
| Acceleration | Label raw values `raw`; prefer `accel_mg[3]` or explicit Q15 |
| Angular velocity | `gyro_mdps[3]`, or document existing floating-point dps |
| Ambient temperature | `temperature_centi_c` (Celsius x100) |
| Relative humidity | `humidity_centi_percent` (%RH x100) |
| Illuminance | `lux_x100` / `wLuxX100` |
| Voltage | `millivolt` / `wMilliVolt` |
| Time | `timestamp_ms`; RTC calendar uses `struct tm` |
| Duration | Explicit `_ms` or `_us` suffix |

RP2040 has no hardware FPU. Prefer cached integer/Q15 values in high-frequency UI paths instead of converting floats every frame.

## Shared I2C

Recorded configuration: `i2c0`, SDA GPIO0, SCL GPIO1; `qmi8658c_init()` defaults to 400 kHz.

`iic0_read_bytes()` and `iic0_write_bytes()` are implemented inside `deivers/drv_QMI8658.c`; RTC and PAJ7620 depend on them. Removing that driver therefore also breaks their transport.

When reorganizing, move shared transport to a board module such as `application/board_sensor_i2c.c/.h` owning bus/pin initialization, consistent read/write returns, timeouts/retries/optional recovery, and multicore/multitask locking. Do not let every sensor reinitialize the same bus/pins. Complete each transaction before another even in a bare-metal loop. Do not initiate I2C from GPIO/ADC/PWM ISRs.

## QMI8658 IMU

Recorded interfaces:

```c
uint8_t qmi8658c_init(void);                 /* 1 success, 0 failure */
int QMI8658A_ReadData(int16_t data[6]);      /* AX, AY, AZ, GX, GY, GZ */
void QMI8658A_Get_G_DPS(float data[6]);      /* accel: g; gyro: dps */
void qmi8658_motion_init(void);
bool qmi8658_motion_poll(uint32_t current_ms);
bool qmi8658_motion_get_position(int16_t *x, int16_t *y);
bool qmi8658_motion_get_gyro_offset_xyz(int16_t *x, int16_t *y, int16_t *z);
bool qmi8658_motion_get_fire_motion(qmi8658_fire_motion_t *motion);
bool qmi8658_motion_consume_z_tap(void);
bool qmi8658_motion_is_ready(void);
```

Initialize with `qmi8658c_init() != 0u`; call `qmi8658_motion_init()` only on success, then drive `qmi8658_motion_poll(get_system_ms())` in the main loop. The recorded implementation samples every 20 ms and returns `true` for successfully read new data. Wait for `qmi8658_motion_is_ready()` after calibration before using mapped positions/offsets.

`QMI8658A_Get_G_DPS()` ignores underlying read failures in the recorded code; for reliable handling, read with `QMI8658A_ReadData()` and convert only on success. Raw order is acceleration XYZ then gyro XYZ. Scenes read cached `qmi8658_motion_*get*` results. Change axis/sign mapping in `qmi8658_motion.h` when orientation changes, rather than swapping axes separately in scenes.

## Power Key

Recorded state machine: GPIO9 input pull-up active low, GPIO2 high to hold power, 20 ms debounce, 1000 ms power-off hold. `power_task()` runs continuously in the main loop to handle startup source, release confirmation, subsequent hold, and power shutdown.

Do not read GPIO9 independently from a scene, which duplicates debounce and competes for events. For UI press/release/short/long events, first extract a service such as `application/power_key_service.c/.h` and expose cached events. The same service must own both power hold and key semantics. Declare active level, pull configuration, debounce, long-press threshold, and latching behavior for additional keys.

## IR TX/RX

```c
void drv_ir_init(uint32_t carrier_hz, uint16_t duty_permille);
bool drv_ir_send_byte_start(uint8_t data);
bool drv_ir_send_task(void);
bool drv_ir_is_sending(void);
void drv_ir_reset_rx(void);
bool drv_ir_receive_snapshot(drv_ir_rx_capture_t *capture);
bool drv_ir_decode_capture(const drv_ir_rx_capture_t *capture, uint8_t *data);
```

`drv_ir_send_byte_start()` loads a sequence and starts its first step. Frequent main-loop `drv_ir_send_task()` calls advance microsecond marks/spaces; transmission is not automatically completed by a background alarm. Call intervals near or above the shortest pulse width distort the waveform.

`ir_task_init()` and `ir_task(period_ms)` implement a loopback self-test with periodic bytes and PASS/FAIL logs. Product services should wrap `drv_ir_*()` according to the protocol. GPIO IRQ accumulates edges; the main loop obtains a complete snapshot and decodes it. Scenes consume decoded events, not capture arrays.

## GL5528 Light Sensor

```c
void drv_light_init(void);
drv_light_sample_t drv_light_read(uint16_t sample_count);
```

The sample contains raw ADC, millivolts, LDR resistance, and `wLuxX100`. The recorded `light_task()` reads/prints every 500 ms without a scene getter. Default reads average 32 samples separated by 50 us (about 1.6 ms minimum), and illuminance conversion uses floating-point `pow`.

Sample at low frequency in an application service and cache integer `lux_x100`; do not call the driver from a PFB handler. `DRV_LIGHT_GL5528_R10_OHM` and `DRV_LIGHT_GL5528_GAMMA` are estimates requiring known-light calibration for absolute illuminance. `wLdrOhm == UINT32_MAX` represents near-open-circuit behavior, not an ordinary resistance value.

## Temperature and Humidity

The original inventory found no dedicated AHT/SHT/DHT or other environmental driver in `deivers/`, `application/`, or Keil entries. QMI8658 internal temperature is not an ambient temperature/humidity sensor.

Before adding support, confirm the actual chip, 7-bit I2C address or single-wire pin, supply, pull-ups, bus speed, typical/maximum conversion times, CRC/checksum, range, and power-on stabilization.

Proposed snapshot:

```c
typedef struct board_environment_sample_t {
    board_sample_meta_t meta;
    int32_t temperature_centi_c;
    uint32_t humidity_centi_percent;
} board_environment_sample_t;
```

Implement start-conversion/wait/read sequences as a nonblocking state machine across polls. Avoid long sleeps in scenes or high-frequency main-loop paths.

## BM8563 RTC

```c
bm8563_err_t bm8563_init(const bm8563_t *rtc);
bm8563_err_t bm8563_read(const bm8563_t *rtc, struct tm *time);
bm8563_err_t bm8563_write(const bm8563_t *rtc, const struct tm *time);
bm8563_err_t bm8563_ioctl(const bm8563_t *rtc, int16_t command, void *buffer);
```

`BM8563_OK` is zero. `BM8563_ERR_LOW_VOLTAGE` signals unreliable backup power/time. POSIX `struct tm` uses `tm_year = year - 1900` (2026 is `126`), `tm_mon = 0..11` (March is `2`), and `tm_wday = 0..6`.

The recorded bring-up wrapper `bm8563_hander_init()` writes a fixed time on every boot, uses `tm_year = 26` (1926 in POSIX semantics), ignores init/write errors, and always returns zero. The `hander` spelling is historical.

For production, separate proposed `board_rtc_init()`, `board_rtc_get_latest()`, and explicit `board_rtc_set_time()`. Write time only after a user setting, initial invalid state, or explicit defaults restoration. Preserve RTC time through ordinary resets.

## PAJ7620 Gestures

Only a chip driver was recorded; `main.c` included its header without initialization or polling.

```c
uint8_t paj7620Init(void);                  /* 0 success */
uint8_t paj7620ReadReg(uint8_t reg, uint8_t qty, uint8_t data[]);
void paj7620SelectBank(bank_e bank);
```

Initialization assumes shared I2C is ready. Gesture flags are bank 0 `PAJ7620_ADDR_GES_PS_DET_FLAG_0/1`. A service should convert register bits to stable enums/latched events and handle simultaneous bits, repetitions, and expiry. Scenes do not read registers.

## Buzzer

```c
void drv_buzzer_init(void);
void drv_buzzer_set_tone(uint16_t freq_hz, uint16_t duty_permille);
bool drv_buzzer_score_start(const drv_buzzer_score_t *score);
bool drv_buzzer_score_task(void);
bool drv_buzzer_pcm_start(const drv_buzzer_pcm_t *pcm);
void drv_buzzer_pcm_stop(void);
```

The recorded `buzzer_task()` loops a demonstration score. For clicks/alerts, send short commands to a service owning PWM/PCM state. Do not start, stop, or poll PCM from draw handlers.

## Initial Scheduling Guidance

These are starting points, subject to the chip datasheet:

| Service | Call/sample cadence |
| --- | --- |
| Power key | Every loop; timestamp-based debounce |
| QMI8658 motion | Poll every loop; recorded internal 20 ms sample period |
| IR TX | As frequently as possible while transmitting |
| IR RX | Check complete snapshots every loop |
| PAJ7620 | 20-100 ms, adjusted for mode and interaction |
| Light | 200-1000 ms |
| Temperature/humidity | 500-2000 ms, respecting conversion time |
| RTC | Typically 1000 ms for a seconds display; alarms/timers as needed |
| Buzzer score | Every loop; PCM advanced by IRQ |

Use unsigned elapsed differences or explicit signed deadline comparisons so uint32 millisecond counter wraparound remains correct.

## Scene Integration

```text
main loop poll
  -> application service updates complete snapshot + sequence
    -> scene frame-start reads once
      -> scene private state
        -> PFB draw handler renders private state only
```

Initialize required services before scenes, but let the UI start when a device fails and display unavailable/stale states. Read snapshots at frame start or a separate update callback, once per frame rather than per PFB tile. On sequence changes, update scene fields and mark only changed dirty regions. Use a consistent snapshot throughout the frame. Depose releases scene-owned resources; shared services retain board-level lifetimes. Latch or queue key/gesture/IR events to avoid loss when event rates exceed scene FPS.

## Recorded Gaps and Change Strategy

The example lacks a unified service layer: I2C is coupled to QMI8658, the key is private to `main.c`, light only prints, environmental drivers are absent, RTC bring-up overwrites time and drops errors, PAJ7620 lacks application integration, and success conventions mix `1`, `0`, and `bool`.

For a new UI/peripheral request, add the smallest thin wrapper needed. If broader unification is requested, migrate shared I2C, then one peripheral service, main scheduling, and scene snapshot consumption while retaining a buildable state after each step.

## Verification

- Check declarations, implementations, Keil entries, and calls with `rg`.
- Verify 7-bit I2C addresses and one-time bus initialization.
- Check each API's success convention, key levels/debounce/hold behavior, and preservation of RTC time at normal boot.
- Confirm an environmental chip exists before writing a chip-specific API.
- Keep hardware I/O, blocking waits, and frequent logs out of scene drawing.
- Update `project/mdk/template.uvprojx` for new `.c/.h` files.
- Build with the project's Keil convention and report errors/warnings plus Code/RO/RW/ZI changes.
