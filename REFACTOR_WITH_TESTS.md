# EX-Turntable Refactor with Tests

This document describes the plan to refactor EX-Turntable so that:

1. Hardware libraries (AccelStepper, Arduino GPIO, Wire/I2C, Serial, EEPROM) are
   decoupled from the application logic behind small interfaces.
2. The application logic (homing, calibration, movement, phase switching, command
   parsing, idle-disable) is extracted into a testable class hierarchy that has no
   dependency on `Arduino.h` or any hardware.
3. The code can be natively unit tested with Google Test via PlatformIO's
   `env:native` (no Arduino hardware required), covering both TURNTABLE and
   TRAVERSER modes from a single test binary.
4. User configuration continues to be made **entirely through `config.h` /
   `config.example.h` / `config.traverser.h` using `#define`**, exactly as today.
   Users must never need to understand C++ syntax or touch the new internal
   structs/classes.

It is written in enough detail that another agent or a human can implement it.

---

## 1. Current state of the codebase (as of the `refactor-with-tests` branch)

All application code lives at the repo root:

| File | Purpose | Notes |
|------|---------|-------|
| `EX-Turntable.ino` | `setup()` / `loop()` | 118 lines; calls the function modules |
| `TurntableFunctions.cpp/h` | homing, calibration, move, phase, LED, sensor debounce | ~520 lines; all logic is file-scope globals + free functions |
| `IOFunctions.cpp/h` | Wire/I2C, serial command parsing, config display | ~440 lines; also contains reset (`avr/wdt.h` / `ESP.restart`) |
| `EEPROMFunctions.cpp/h` | EEPROM storage of calibrated step count | ~90 lines; TTEX magic bytes + version |
| `AccelStepper.cpp/h` | vendored AccelStepper library (in-repo, not a lib_dep) | modify only via the adapter; it must still compile on AVR and native |
| `defines.h` | selects `config.h`, provides `#ifndef` defaults and `#error` validation | keep its role unchanged |
| `standard_steppers.h` | `STEPPER_DRIVER` macros (ULN2003 variants, A4988) | driver selection for users |
| `config.example.h` / `config.traverser.h` | user-facing `#define` config | unchanged workflow |
| `version.h` | version string | unchanged |
| `platformio.ini` | AVR envs only (nano, uno, esp32dev) | add native test env |

Build config today: `src_dir = .`, `include_dir = .`, `framework = arduino`
for the AVR envs. CI copies `config.example.h` → `config.h` then `pio run`
(see `.github/workflows/build-test-workflow.yml`).

The `stepper` object is a global `AccelStepper` constructed by the
`STEPPER_DRIVER` macro in `TurntableFunctions.cpp`:

```cpp
AccelStepper stepper = STEPPER_DRIVER;
```

### 1.1 Known quirks that MUST be preserved (behavior-preserving refactor)

- `SELECTED_DRIVER` and `A4988_DRIVER` are referenced at
  `TurntableFunctions.cpp:100` (`#if SELECTED_DRIVER == A4988_DRIVER`) but are
  **never defined anywhere** in the repo. Undefined identifiers in a preprocessor
  expression evaluate to `0`, so `#if 0 == 0` is **always true** and the block
  (`setEnablePin(STEPPER_ENABLE_PIN)` + `setPinsInverted(...)`) runs for every
  driver type. Preserve this behavior in `main.cpp`/`AccelStepperAdapter` and add
  a code comment flagging the dead `#if`.
- `PHASE_SWITCH_ANGLE` validation: `processAutoPhaseSwitch()` in
  `TurntableFunctions.cpp` contains a buried compile-time clamp:

  ```cpp
  #if PHASE_SWITCH_ANGLE + 180 >= 360
  #undef PHASE_SWITCH_ANGLE
  #define PHASE_SWITCH_ANGLE 45
  #endif
  ```

  Move this clamp into `defines.h` (next to the other defaults/validation) so the
  runtime behavior is identical but there is no macro mutation hidden inside a
  `.cpp` file.
- `processLED()` in `TurntableFunctions.cpp` does `uint16_t currentMillis =
  millis();` (truncation bug) while `ledMillis` is `unsigned long`. Fix by making
  `currentMillis` `unsigned long`. This is a deliberate, documented bugfix in an
  otherwise behavior-preserving refactor.
- Reset command `R` uses `avr/wdt.h` (`#ifndef ESP32`) or `ESP.restart()`. This is
  hardware-only and belongs in `main.cpp` (or the hardware layer), guarded so the
  native test build still compiles.
- `config.h` is expected to be at the repo root (CI copies it there). AVR builds
  must continue to resolve `#include "config.h"` from `defines.h` (which will move
  into `src/`).

---

## 2. Two PRs that are part of this work

Both PRs are by `mwinters-stuff`, were analyzed in full, and are **already merged
into this branch** (`refactor-with-tests`) — **not** into `main` or `devel`:

- **PR #112 "disable stepper after a set number of seconds"** — adds
  `DISABLE_OUTPUT_TIMEOUT` (seconds). When `DISABLE_OUTPUTS_IDLE` is defined and
  the stepper transitions running→stopped, outputs are disabled immediately (as
  today) or, if `DISABLE_OUTPUT_TIMEOUT` is set, after that many seconds of
  idling. Original implementation added
  `scheduleStepperDisable()`/`processStepperDisable()` with file-scope statics in
  `IOFunctions.cpp`.
- **PR #113 "if stepper is on sensor at start, rotate away then home"** — adds
  `ROTATE_BEFORE_HOME`. When enabled, if the home sensor is active at the start of
  homing, move `sanitySteps` away, wait for the sensor to deactivate, then stop and
  begin normal homing. Original implementation used `static` locals inside
  `moveHome()`.

**Merge status:** both PRs were merged into `refactor-with-tests` as `--no-ff`
merge commits (verify with `git log --first-parent`):
- PR #112 "disable stepper after a set number of seconds" → commit `202f070`
- PR #113 "if stepper is on sensor at start, rotate away then home" → commit `acaece4`

The merged tree builds cleanly with both `config.example.h` and
`config.traverser.h` (nano + uno, verified before this refactor began).

**Porting strategy:** the refactor ports both features into the new architecture as
first-class state machines (see §5.1 rotate-before-home and §5.7 idle-disable).
Because the refactor rewrites/deletes the exact files these PRs touched
(`TurntableFunctions.cpp`, `IOFunctions.cpp`, `EX-Turntable.ino`,
`config.example.h`), their changes are absorbed during the port — do not attempt to
re-merge the PRs against the refactored tree. When porting, reference the merged
commits above so the contributor's intent and implementation remain attributable
and verifiable.

**Decisions already made:**
- `DISABLE_OUTPUT_TIMEOUT` is **opt-in**: keep it commented out in
  `config.example.h` (default behavior remains: disable immediately on idle). Note
  that PR #112, as merged, currently has `#define DISABLE_OUTPUT_TIMEOUT 60`
  active in `config.example.h` — the refactor must revert this to a commented-out,
  opt-in entry.
- `ROTATE_BEFORE_HOME` is **opt-in** and matches PR #113 semantics: rotate clear
  only on the **first homing after boot** (a flag that is never reset), even if the
  sensor is re-engaged later. Preserve the PR's behavior where, if the sensor never
  clears during the rotate-away move, the homing sequence simply waits (do not add
  new failure handling); flag this known limitation in a code comment.

---

## 3. Reusable test infrastructure from DCCEXProtocol

The repo `https://github.com/DCC-EX/DCCEXProtocol` (MIT-style CI-proven
`env:native` + Googletest setup) contains mocks that can be copied, extended, and
used directly:

| File (in DCCEXProtocol) | Reuse |
|--------------------------|-------|
| `test/mocks/Arduino.h` | Mock `HIGH/LOW`, pin modes, `F(str)`, `byte`; gmock singleton `MockArduino` with `mockPinMode`/`mockDigitalWrite`; mockable `millis()`/`micros()` backed by `_currentMillis`/`_currentMicros` with `advanceMillis()`/`advanceMicros()`/`resetMillis()`/`resetMicros()`; `digitalRead`, `delay`, `itoa`/`ltoa`/`utoa`. |
| `test/mocks/Print.h` | `__FlashStringHelper` typedef (makes `F("...")` compile in mocks) + a `Print` class with `print`/`println` overloads. |
| `test/mocks/Stream.h` | `Stream : Print` with separate input/output buffers: `stream << "<M 100 0>"` feeds input; `getOutput()` asserts output; `clearInput()`/`clearOutput()` reset. |
| `test/test_main.cpp` | Standard gtest `main()`. |
| `platformio.ini` `[env:native_test]` | `platform = native`, `lib_deps = googletest`, `test_framework = googletest`, `test_build_src = yes`, `-I./test/mocks`, `-DNATIVE_TESTING`, ASan/UBSan + gcov flags. |
| `.github/workflows/tests.yml` | CI running `pio test -e native_test` with `--gtest-shuffle --gtest_repeat=5`. |
| `extra_flags.py`, `generate_test_coverage.py` | Coverage tooling. |
| `test/setup/*` harness pattern | `TEST_F` fixtures with `SetUp`/`TearDown` (reset `millis()`, clear buffers). |

**Licensing:** the DCCEXProtocol `Arduino.h`, `Print.h`, `Stream.h` mocks are
GPLv3 — directly compatible with EX-Turntable's GPLv3. Do **not** copy
`TestHarnessBase.hpp` (it is CC-BY-SA 4.0); write our own equivalent (~40 lines)
following the same pattern.

**Gaps to fill in the copied mocks:**
1. Add `delayMicroseconds()` to the mock `Arduino.h` (vendored
   `AccelStepper.cpp:418` needs it).
2. Add `print(int n, int base)` / `println(int n, int base)` overloads for
   `Serial.println(I2C_ADDRESS, HEX)` and add `print(float)` / `println(float)`
   for debug output of `stepper.maxSpeed()` / `stepper.acceleration()`.
3. Add small shims `test/mocks/Wire.h` and `test/mocks/EEPROM.h` so that
   `test_build_src = yes` can compile the whole `src/` tree on the host (see §7).

---

## 4. Target architecture

The logic core and interfaces are **Arduino-free**: they include no `Arduino.h`
and use only standard C++ types (`stdint.h`, `std::string`, `std::array`, etc.).
Only `main.cpp` and the hardware adapters include `Arduino.h` (or the mocked
versions during native builds).

```
src/
  main.cpp                        # replaces EX-Turntable.ino (PlatformIO compiles .cpp with Arduino framework)
  defines.h                       # moved from repo root; role unchanged (config selection, defaults, #error)
  standard_steppers.h             # moved from repo root; STEPPER_DRIVER macros unchanged
  hardware/
    IStepper.h                    # pure-virtual interface (see §4.1)
    AccelStepperAdapter.h/.cpp    # wraps vendored AccelStepper
    SensorPair.h/.cpp             # debounced home + limit sensors over GPIO -> ISensorPair
    OutputController.h/.cpp       # relays (phase), LED, accessory, extra outputs -> IOutputController
    EEPROMStore.h/.cpp            # wraps <EEPROM.h> -> IEEPROMStore
    SerialLogger.h/.cpp           # wraps Serial -> ILogger
  controller/
    TurntableConfig.h             # internal runtime config struct (see §6)
    ConfigLoader.h/.cpp           # translates #defines -> TurntableConfig (see §6)
    TurntableController.h/.cpp    # all state machines (see §5)
    CommandParser.h/.cpp          # serial frame + I2C byte-protocol decode (see §8)
test/
  mocks/                          # extended copies from DCCEXProtocol + Wire.h/EEPROM.h shims
    Arduino.h
    Print.h
    Stream.h
    Wire.h
    EEPROM.h
    MockStepper.h                 # gmock mock of IStepper
    MockSensorPair.h              # gmock mock of ISensorPair
    MockOutputController.h        # gmock mock of IOutputController
    MockEEPROMStore.h             # gmock mock of IEEPROMStore
    MockLogger.h                  # gmock mock of ILogger
  setup/
    TestHarnessBase.hpp           # our own gmock fixture base (not copied from DCCEXProtocol)
  unit/
    test_move.cpp
    test_homing.cpp
    test_calibration.cpp
    test_phase_switch.cpp
    test_command_parser.cpp
    test_eeprom.cpp
    test_led.cpp
    test_idle_disable.cpp
    test_config_loader.cpp
platformio.ini                    # + env:native_test (see §7)
.github/workflows/tests.yml       # new CI job for native tests
extra_flags.py                    # copied/adapted from DCCEXProtocol
generate_test_coverage.py         # copied/adapted from DCCEXProtocol
```

### 4.1 `IStepper` interface

Pure virtual class with exactly the methods the application uses today (17):

```cpp
class IStepper {
public:
  virtual ~IStepper() = default;
  virtual void moveTo(long absolute) = 0;
  virtual void move(long relative) = 0;
  virtual bool run() = 0;
  virtual void stop() = 0;
  virtual void setMaxSpeed(float speed) = 0;
  virtual float maxSpeed() = 0;
  virtual void setAcceleration(float acceleration) = 0;
  virtual float acceleration() = 0;
  virtual long distanceToGo() = 0;
  virtual long targetPosition() = 0;
  virtual long currentPosition() = 0;
  virtual void setCurrentPosition(long position) = 0;
  virtual void enableOutputs() = 0;
  virtual void disableOutputs() = 0;
  virtual void setEnablePin(uint8_t enablePin) = 0;
  virtual void setPinsInverted(bool directionInvert, bool stepInvert, bool enableInvert) = 0;
  virtual bool isRunning() = 0;
};
```

`AccelStepperAdapter` implements this by composition over the vendored
`AccelStepper` (not inheritance — only `enableOutputs`/`disableOutputs` are
virtual on AccelStepper; the rest are not, and composition gives a clean seam).

### 4.2 Other interfaces

```cpp
class ISensorPair {           // debounced home + limit sensing
public:
  virtual ~ISensorPair() = default;
  virtual bool homeActive() = 0;
  virtual bool limitActive() = 0;
};

class IOutputController {     // relay phase, LED, accessory, extra outputs
public:
  virtual ~IOutputController() = default;
  virtual void setPhase(uint8_t phase) = 0;
  virtual void setLED(bool on) = 0;
  virtual void setAccessory(bool on) = 0;
  virtual void setExtra(uint8_t output, bool on) = 0;   // RT_EX_TURNTABLE extras
};

class IEEPROMStore {          // calibrated step-count persistence
public:
  virtual ~IEEPROMStore() = default;
  virtual long getSteps() = 0;
  virtual void writeSteps(long steps) = 0;
  virtual void clear() = 0;
};

class ILogger {               // all text output
public:
  virtual ~ILogger() = default;
  virtual void print(const char *msg) = 0;
  virtual void print(const __FlashStringHelper *msg) = 0;
  virtual void print(int value) = 0;
  virtual void print(long value) = 0;
  virtual void print(unsigned long value) = 0;
  virtual void print(float value) = 0;
  virtual void print(int value, int base) = 0;
  virtual void println() = 0;
  virtual void println(const char *msg) = 0;
  virtual void println(const __FlashStringHelper *msg) = 0;
  virtual void println(int value) = 0;
  virtual void println(long value) = 0;
  virtual void println(unsigned long value) = 0;
  virtual void println(float value) = 0;
  virtual void println(int value, int base) = 0;
};
```

> `F("...")` strings inside the controller must become plain literals via the
> logger interface (the controller is Arduino-free). For AVR, `SerialLogger`
> wraps them with `F()` internally or simply prints them (they are in PROGMEM as
> string literals in the .rodata; for a small project plain literals are fine).

---

## 5. `TurntableController` — the state machines

One class owning all turntable behavior. Constructor:

```cpp
class TurntableController {
public:
  TurntableController(IStepper &stepper,
                      ISensorPair &sensors,
                      IOutputController &outputs,
                      IEEPROMStore &eeprom,
                      ILogger &logger,
                      const TurntableConfig &config);
  void begin();                 // was startupConfiguration() + setupStepperDriver()
  void loop();                  // was the non-sensor-testing body of loop() in the .ino
  // command entry points used by CommandParser:
  void moveToPosition(long steps, uint8_t phaseSwitch);
  void initiateHoming();
  void initiateCalibration();
  void setLEDActivity(uint8_t activity);
  void setAccessory(bool on);
  void setExtra(uint8_t activity);
  bool stepperRunning() const;  // for requestEvent/status
  long fullTurnSteps() const;   // for display
  // calibration state:
  bool calibrating() const;
  uint8_t homed() const;
private:
  // state machines + member state replacing the file-scope globals:
  //   homed, calibrating, calibrationPhase, lastStep, lastTarget, fullTurnSteps,
  //   halfTurnSteps, phaseSwitchStartSteps, phaseSwitchStopSteps, lastHomeDebounce,
  //   lastLimitDebounce, lastHomeSensorState, lastLimitSensorState, ledState,
  //   ledOutput, ledMillis, invertDirection/Step/Enable (from config), plus:
  //   idleDisablePending, idleDisableStartMillis (PR #112)
  //   rotateBeforeHomeDone (PR #113)
};
```

### 5.1 Homing — `moveHome()` (preserve exact behavior)

Current logic (TurntableFunctions.cpp:177-217), to be preserved 1:1:
1. `setPhase(0)`.
2. If home sensor active → `stepper.stop()`, (if `disableOutputsIdle`) `disableOutputs()`,
   `setCurrentPosition(0)`, `lastStep = 0`, `homed = 1`, log "Turntable homed successfully".
3. Else if `!stepper.isRunning()`:
   - if `targetPosition() == lastTarget` → `setCurrentPosition(0)`, `lastStep = 0`,
     `homed = 2`, log failure (random home).
   - else → `enableOutputs()`, `move(sanitySteps)`, `lastTarget = targetPosition()`,
     log "Homing started".

**PR #113 integration (only when `config.rotateBeforeHome` and this is the first
homing since boot):** before the above, if sensor is active and not moving, log
"Home sensor active at startup, rotating clear before homing", `enableOutputs()`,
`move(sanitySteps)`, set clearing state, return. While clearing: if sensor still
active return (keep rotating); when the sensor deactivates: `stepper.stop()`,
`lastTarget = sanitySteps`, log "Sensor cleared, starting homing", and clear the
rotating flag so normal homing proceeds. The "rotated once" flag is never reset
after boot (matches PR as-submitted).

### 5.2 Calibration — `calibration()` (preserve exact behavior)

Two compile-time variants become runtime branches on `config.mode`:

- **TURNTABLE (2-phase):**
  - phase 2 && home active && `currentPosition() > homeSensitivity` → stop,
    (disable if `disableOutputsIdle`), `fullTurnSteps = |currentPosition()|`,
    `halfTurnSteps = fullTurnSteps/2`, recompute auto phase switch, `calibrating=false`,
    `calibrationPhase=0`, `eeprom.writeSteps(fullTurnSteps)`, log completion,
    `setCurrentPosition(currentPosition())`, `homed=0`, `lastTarget=sanitySteps`,
    display config.
  - phase 1 && `lastStep == sanitySteps` && home active && `currentPosition() > homeSensitivity`
    → stop, `setCurrentPosition(0)`, phase 2, `enableOutputs()`, `moveTo(sanitySteps)`,
    `lastStep = sanitySteps`.
  - phase 0 && `!isRunning()` && `homed == 1` → phase 1, `enableOutputs()`,
    `moveTo(sanitySteps)`, `lastStep = sanitySteps`.
  - phase 1 or 2 && `!isRunning()` && `currentPosition() == sanitySteps` → log
    failure, (disable if `disableOutputsIdle`), `calibrating=false`, phase 0.
- **TRAVERSER (3-phase):** identical except:
  - phase 2 && limit active → stop, `setCurrentPosition(currentPosition())`,
    log "Phase 3, counting limit steps...", `moveTo(0)`, `lastStep=0`, phase 3.
  - phase 3 && limit not active → stop, (disable if `disableOutputsIdle`),
    `fullTurnSteps = |currentPosition()|`, ... (same completion as turntable phase 2).
  - phase 1 uses `moveTo(-sanitySteps)`, `lastStep = sanitySteps`.
  - phase 0: if home active log "already homed", else `enableOutputs()`,
    `moveTo(sanitySteps)`.

Note the original phase-2 completion condition differs between modes (turntable
checks `homeSensitivity`, traverser has an extra `else if` to move off the limit).
Preserve exactly.

### 5.3 Move — `moveToPosition(steps, phaseSwitch)` (preserve exact math)

1. If `steps != lastStep`:
   - Compute `moveSteps`:
     - TRAVERSER: `moveSteps = lastStep - steps`.
     - TURNTABLE + `ROTATE_FORWARD_ONLY`: `moveSteps = steps - lastStep; if < 0 += fullTurnSteps`.
     - TURNTABLE + `ROTATE_REVERSE_ONLY`: `moveSteps = steps - lastStep; if > 0 -= fullTurnSteps`.
     - TURNTABLE default (shortest): if `steps-lastStep > halfTurnSteps` →
       `steps - fullTurnSteps - lastStep`; else if `< -halfTurnSteps` →
       `fullTurnSteps - lastStep + steps`; else `steps - lastStep`.
   - If `PHASE_SWITCHING AUTO` (runtime `config.phaseSwitchingAuto`): compute
     `phaseSwitch = (steps >= 0 && steps < phaseSwitchStartSteps) || (steps <= fullTurnSteps && steps >= phaseSwitchStopSteps) ? 0 : 1;`
   - `setPhase(phaseSwitch)`, `lastStep = steps`, `enableOutputs()`, `move(moveSteps)`,
     `lastTarget = targetPosition()`.
2. If `steps == lastStep`, do nothing.

### 5.4 Auto phase switch — `processAutoPhaseSwitch()`

`phaseSwitchStartSteps = fullTurnSteps / 360 * PHASE_SWITCH_ANGLE;`
`phaseSwitchStopSteps  = fullTurnSteps / 360 * (PHASE_SWITCH_ANGLE + 180);`
(integer arithmetic). The angle clamp moves to `defines.h` / `ConfigLoader` (see §1.1).

### 5.5 Sensor debounce — `getHomeState()` / `getLimitState()`

Preserve: read raw state; if it differs from last and
`(millis() - lastDebounce) > DEBOUNCE_DELAY`, update last + timestamp; return last.
These move into `SensorPair` (hardware layer). `DEBOUNCE_DELAY` comes from config.

### 5.6 LED — `processLED()` (in `loop()`)

State 4 = on, 5 = slow blink, 6 = fast blink, 7 = off; else hold. Use
`LED_SLOW`/`LED_FAST` from config. Fix the `uint16_t currentMillis = millis()`
truncation to `unsigned long`.

### 5.7 Idle disable (PR #112) — in `loop()`

When `config.disableOutputsIdle`:
- Track `lastRunningState`. On a running→stopped transition call the scheduler.
- `schedule()`: if `config.disableOutputTimeoutMs == 0` → `disableOutputs()`
  immediately (today's behavior). Else record `idleDisablePending = true`,
  `idleDisableStartMillis = millis()`.
- `process()` (called every loop): if pending and `isRunning()` → cancel pending.
  Else if `millis() - idleDisableStartMillis >= disableOutputTimeoutMs` →
  `disableOutputs()`, clear pending.

### 5.8 Loop body (was `EX-Turntable.ino` loop() non-sensor-testing branch)

1. TRAVERSER mode: if `limitActive()` && `!calibrating` && `isRunning()` &&
   `targetPosition() < 0` → log, `if (!homed) homed = 1;`, `stop()`,
   `setCurrentPosition(currentPosition())`.
2. If `homeActive()` && `homed` && `!calibrating` && `isRunning()` &&
   `distanceToGo() > 0` → log, `stop()`, `setCurrentPosition(0)`.
3. If `homed == 0` → `moveHome()`.
4. If `calibrating` → `calibration()`.
5. `stepper.run()`.
6. `processLED()`.
7. Idle-disable handling (§5.7).

The `.ino`'s sensor-testing branch (`sensorTesting`) and `processSerialInput()`
are handled by `main.cpp` + `CommandParser` (see §8).

---

## 6. Configuration design (`config.h` stays the only user surface)

**Requirement (hard):** users configure via `#define`s in `config.h` /
`config.example.h` / `config.traverser.h`. The internal `TurntableConfig` struct is
an implementation detail — never user-editable, never documented as a user knob.

Flow:

```
config.h (user edits #defines)
    │  #include'd by
defines.h (__has_include selection, #ifndef defaults, #error validation — unchanged role)
    │
    ▼
ConfigLoader (new; builds the struct from the defines)
    │
    ▼
TurntableConfig (internal, Arduino-free, in controller/TurntableConfig.h)
```

`TurntableConfig` fields (all with safe defaults when the define is absent —
the `#ifndef` defaults in `defines.h` keep working as today):

```cpp
struct TurntableConfig {
  bool traverser;                       // TURNTABLE_EX_MODE == TRAVERSER
  bool sensorTesting;                   // #ifdef SENSOR_TESTING
  bool homeSensorActiveHigh;            // HOME_SENSOR_ACTIVE_STATE == HIGH
  bool limitSensorActiveHigh;           // LIMIT_SENSOR_ACTIVE_STATE == HIGH
  bool relayActiveHigh;                 // RELAY_ACTIVE_STATE == HIGH
  bool phaseSwitchingAuto;              // PHASE_SWITCHING == AUTO
  uint8_t phaseSwitchAngle;             // PHASE_SWITCH_ANGLE (clamped)
  uint16_t debounceDelayMs;             // DEBOUNCE_DELAY
  uint16_t homeSensitivity;             // HOME_SENSITIVITY
  uint32_t sanitySteps;                 // SANITY_STEPS
  uint32_t fullTurnSteps;               // FULL_STEP_COUNT override, else 0 (read EEPROM)
  float maxSpeed;                       // STEPPER_MAX_SPEED
  float acceleration;                   // STEPPER_ACCELERATION
  uint8_t gearingFactor;                // STEPPER_GEARING_FACTOR
  bool rotateForwardOnly;               // #ifdef ROTATE_FORWARD_ONLY
  bool rotateReverseOnly;               // #ifdef ROTATE_REVERSE_ONLY
  uint16_t ledFastMs;                   // LED_FAST
  uint16_t ledSlowMs;                   // LED_SLOW
  bool debug;                           // #ifdef DEBUG
  bool disableOutputsIdle;              // #ifdef DISABLE_OUTPUTS_IDLE
  uint32_t disableOutputTimeoutMs;      // #ifdef DISABLE_OUTPUT_TIMEOUT (PR #112); 0 = immediate
  bool rotateBeforeHome;                // #ifdef ROTATE_BEFORE_HOME (PR #113)
  bool invertDirection;                 // #ifdef INVERT_DIRECTION
  bool invertStep;                      // #ifdef INVERT_STEP
  bool invertEnable;                    // #ifdef INVERT_ENABLE
  bool useRTBoard;                      // #ifdef USE_RT_EX_TURNTABLE
};
```

`ConfigLoader`:
- Lives in `main.cpp`-side (or `src/controller/ConfigLoader.h/.cpp`); it is the
  **only** place that reads the `#define`s. Everything below it only sees the struct.
- Implements the `#ifdef`-gated mapping above, including the `PHASE_SWITCH_ANGLE`
  clamp and the `ROTATE_FORWARD_ONLY`/`ROTATE_REVERSE_ONLY` mutual-exclusion
  checks (also still enforced by `#error` in `defines.h`).
- `validateConfig()`: assert `fullTurnSteps`/`sanitySteps` sanity, clamp gearing
  factor to 10 (today done in `receiveEvent` — move the clamp here), etc. Returns
  the struct; logs warnings via the logger if invalid.

`main.cpp` builds the hardware (real adapters), builds the config via
`ConfigLoader`, constructs `TurntableController`, sets up Wire callbacks
(`receiveEvent`/`requestEvent`), and wires serial input into `CommandParser`.

> `STEPPER_DRIVER` (macro constructing `AccelStepper`) remains the user-facing
> driver selection in `standard_steppers.h`; only the construction site in
> `main.cpp` changes (wrap in `AccelStepperAdapter`). Driver-specific setup
> (`setEnablePin`, `setPinsInverted`) is applied in `main.cpp` for A4988-style
> drivers; preserve today's always-true `#if SELECTED_DRIVER == A4988_DRIVER`
> behavior (see §1.1).

---

## 7. Build configuration (platformio.ini) and native testing

```ini
[platformio]
default_envs =
    nanoatmega328new
    uno
src_dir = src
include_dir = .

[env:nanoatmega328new]
platform = atmelavr
board = nanoatmega328new
framework = arduino
monitor_speed = 115200
monitor_echo = yes

[env:nanoatmega328]
platform = atmelavr
board = nanoatmega328
framework = arduino
monitor_speed = 115200
monitor_echo = yes

[env:uno]
platform = atmelavr
board = uno
framework = arduino
monitor_speed = 115200
monitor_echo = yes

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

[env:native_test]
platform = native
lib_deps = googletest
test_framework = googletest
test_build_src = yes
build_flags =
    -std=c++17 -g2 -Wall
    -I./test/mocks
    -I./test/setup
    -DNATIVE_TESTING
    -fsanitize=address
    -fsanitize=undefined
    -fno-omit-frame-pointer
    --coverage
    -fprofile-abs-path
test_filter = *
```

Notes:
- **AVR envs:** `src_dir = src`, `include_dir = .` so the root-level
  `config.h` (created by CI's `cp config.example.h config.h`) still resolves from
  `defines.h` (now in `src/`). The existing CI build workflow must keep passing
  unchanged.
- **Native env:** `test_build_src = yes` compiles the whole `src/` tree into the
  test binary, so `main.cpp` + every adapter must compile under the mocks. This is
  why `Wire.h`, `EEPROM.h`, and `delayMicroseconds` mocks are required (§3). The
  vendored `AccelStepper.cpp` will also compile on host (it only uses
  `digitalWrite`, `pinMode`, `micros`, `delayMicroseconds` — all mocked).
- `main.cpp` must be written so it compiles on native: guard `<avr/wdt.h>` with
  `#ifndef ESP32` and any hardware reset with `#ifdef NATIVE_TESTING` (no-op), or
  move reset entirely behind the hardware layer.
- **Single env, both modes:** `TURNTABLE_EX_MODE` becomes a runtime field
  (`config.traverser`). Tests exercise both modes in the same binary via
  `TurntableConfig` values. Do not create separate native envs per mode.

---

## 8. `CommandParser` — command decoding (preserve exact behavior)

Pure logic, no hardware. Two entry paths:

- `processSerialFrame()` — the serial `<command args>` protocol from
  `IOFunctions.cpp::processSerialInput()`:
  - States: waiting for `<`, accumulating until `>`, `\0`-terminated.
  - Commands: `C` (calibration, gated: not running), `D` (toggle debug),
    `E` (erase EEPROM, gated: not running; resets `fullTurnSteps = 0` unless
    `FULL_STEP_COUNT`), `H` (home, gated: not running), `M steps activity`
    (validate `0 <= steps <= 32767`; sets `testStepsMSB/LSB`, `testActivity`,
    `testCommandSent`, calls the I2C receive handler with 3 bytes — preserve this
    indirection!), `R` (reset — delegate to hardware, no-op under tests), `T`
    (toggle sensor testing, gated: not running; `Wire.end()` call moves to
    `main.cpp`), `V` (display config).
- `processWireReceive(received, readByte)` — the 3-byte I2C protocol from
  `receiveEvent()`:
  - If `received == 3`: read MSB/LSB/activity (or the test-command values when
    `testCommandSent`), `receivedSteps = (MSB << 8) + LSB`, clamp gearing factor to
    10, `steps = receivedSteps * gearingFactor`.
  - Dispatch: `steps <= fullTurnSteps && activity < 2 && !running && !calibrating`
    → `moveToPosition`; activity 2 → `initiateHoming`; activity 3 →
    `initiateCalibration`; 4-7 → `setLEDActivity`; 8 → accessory on; 9 → accessory
    off; 10-17 (RT board only) → `setExtra`; else ignore.
  - If `received != 3`: drain any buffered bytes (in the hardware layer).
  - `requestEvent` status byte: `isRunning() ? 1 : 0`.
- `displayConfig()` — all the `displayTTEXConfig()` serial output, now via
  `ILogger`. Preserve every line, including gearing factor, phase-switch info,
  mode, rotation direction, invert flags, speeds, sensor-testing block.

The `Stream` mock (`stream << "<M 100 0>"`, `getOutput()`) is ideal for
testing `processSerialFrame`; `processWireReceive` is tested by passing an array
of bytes directly.

---

## 9. Execution order

0. **Merged baseline (already done on this branch):** PR #112 (`202f070`) and
   PR #113 (`acaece4`) are merged into `refactor-with-tests`. Both
   `config.example.h` and `config.traverser.h` builds pass (`pio run`,
   nanoatmega328new + uno). Step 1 starts from this merged tree; do not re-merge
   the PRs.
1. **Scaffold the layout:**
   - `git mv` root sources into `src/`; convert `EX-Turntable.ino` → `src/main.cpp`
     (temporary: keep it compiling against the old modules first).
   - Update `platformio.ini` (`src_dir = src`, `include_dir = .`).
   - Verify AVR builds still pass: `pio run -e nanoatmega328new -e uno` and the CI
     `config.example.h`/`config.traverser.h` copies.
2. **Test scaffolding:**
   - Copy `test/mocks/Arduino.h`, `Print.h`, `Stream.h` from DCCEXProtocol; extend
     with `delayMicroseconds`, base/float print overloads.
   - Add `test/mocks/Wire.h` + `EEPROM.h` shims.
   - Add `test/setup/TestHarnessBase.hpp` (our own), `test/test_main.cpp`.
   - Add `[env:native_test]` + a trivial passing test to prove the toolchain.
   - Add `.github/workflows/tests.yml` (based on DCCEXProtocol's) and the coverage
     scripts.
3. **Interfaces + adapters** (in dependency order): `IStepper`/`AccelStepperAdapter`,
   `ISensorPair`/`SensorPair`, `IOutputController`/`OutputController`,
   `IEEPROMStore`/`EEPROMStore`, `ILogger`/`SerialLogger`. Keep the old modules
   compiling alongside during the transition if helpful.
4. **`TurntableConfig` + `ConfigLoader`**: define the struct; move the
   `PHASE_SWITCH_ANGLE` clamp and gearing clamp here. Delete the global
   configuration state from the old modules.
5. **`TurntableController`**: port §5 verbatim (state machines + loop body +
   PR #112 + PR #113). Replace `Serial.println` with `ILogger`.
6. **`CommandParser`**: port §8; `displayConfig` via logger.
7. **Thin `main.cpp`**: build hardware + config + controller; Wire callbacks;
   serial→parser; sensor-testing branch; reset handling (guarded for native).
8. **Delete the old modules** once nothing references them.
9. **Write the unit tests** (§10) for both modes.
10. **Regression:** `pio run` (nano/uno, both config files) and
    `pio test -e native_test` green; verify CI workflows.

---

## 10. Unit test plan

Test harness conventions (from DCCEXProtocol, adapted):
- One `TestHarnessBase` with mocks; `SetUp` builds fresh mocks + controller,
  `TearDown` calls `resetMillis()` and clears buffers.
- Use the mock `millis()`/`advanceMillis()` for all timing tests (LED blink,
  debounce, idle-disable).
- `MockStepper` (gmock of `IStepper`) drives all movement assertions
  (`EXPECT_CALL(_stepper, move(1234))`, etc.); use `ON_CALL`/`WillRepeatedly` for
  `isRunning()`/`currentPosition()`/`targetPosition()`.
- Each test file covers both `config.traverser = false` and `true` where relevant
  (parameterized fixture or explicit construction).

| File | Cases |
|------|-------|
| `test_move.cpp` | shortest-distance selection incl. exact half-turn boundary (`> halfTurnSteps`, `< -halfTurnSteps`, equal); forward-only (wrap to `fullTurnSteps`); reverse-only (wrap to `-fullTurnSteps`); traverser `moveSteps = lastStep - steps`; no-op when `steps == lastStep`; `lastStep`/`lastTarget` bookkeeping; phase switch flag computation for AUTO and MANUAL. |
| `test_homing.cpp` | fresh boot → `move(sanitySteps)` + `lastTarget` set; sensor active → `setCurrentPosition(0)`, `homed=1`, disable if idle; failure path → `homed=2` when target==lastTarget; rotate-before-home (PR #113): sensor active at first homing → `move(sanitySteps)`, sensor clears → `stop()` + `lastTarget=sanitySteps` + homing proceeds; rotate only once ever. |
| `test_calibration.cpp` | TURNTABLE phases 0→1→2→complete: `fullTurnSteps`, `halfTurnSteps`, EEPROM write, auto phase-switch recompute; failure (reaches sanity); TRAVERSER 3-phase incl. limit-stop-then-clear; `homed=0`/`lastTarget=sanitySteps` on completion. |
| `test_phase_switch.cpp` | `phaseSwitchStartSteps`/`phaseSwitchStopSteps` math; angle clamp (define too-large angle in `ConfigLoader` test). |
| `test_command_parser.cpp` | serial frames `<C/D/E/H/M/R/T/V>`; `M` validation (negative, > 32767); running gating; test-command → Wire receive indirection; Wire 3-byte protocol (MSB/LSB/activity), gearing clamp, activity dispatch 0-17; `requestEvent` status; `received != 3` drain; malformed frames. |
| `test_eeprom.cpp` | `EEPROMStore` round-trip, TTEX magic bytes, version mismatch → invalid, clear; via `MockEEPROM`-backed store or the `EEPROM.h` shim. |
| `test_led.cpp` | states 4/5/6/7 with `advanceMillis`; blink toggling at `LED_SLOW`/`LED_FAST`. |
| `test_idle_disable.cpp` | (PR #112) immediate disable when timeout 0; scheduled → fires after N seconds; re-run cancels; feature off → no calls to `disableOutputs()`. |
| `test_config_loader.cpp` | define → struct field parity for every option (this is the contract that keeps `config.h` semantics stable). |

Also add gmock tests that the interfaces are honored by the adapters
(`AccelStepperAdapter` delegates to the wrapped `AccelStepper`).

---

## 11. Licensing

- Mocks copied from DCCEXProtocol (`Arduino.h`, `Print.h`, `Stream.h`) are GPLv3 —
  compatible. Keep their headers.
- Write our own `TestHarnessBase.hpp` (do not copy the CC-BY-SA 4.0 one).
- All new files carry the repo's existing GPLv3 header style (© year, DCC-EX /
  EX-Turntable, GPLv3 text).

---

## 12. Definition of done

- `pio run -e nanoatmega328new -e uno` passes with both `config.example.h` and
  `config.traverser.h` copied to `config.h` (CI build workflow unchanged).
- `pio test -e native_test` passes with the full suite above, in both modes,
  under ASan/UBSan.
- Users configure via `config.h` exactly as before; `config.example.h` documents
  the two new optional knobs (`DISABLE_OUTPUT_TIMEOUT`, `ROTATE_BEFORE_HOME`).
- PR #112 and #113 functionality present on the new architecture with the agreed
  defaults (both opt-in) and the agreed rotate-before-home scope (first homing
  only).
- Known quirks preserved and flagged with code comments (§1.1).
