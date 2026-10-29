# GPT Software Building Block

## Overview

This Software Building Block (SBB) demonstrates typical use of the Renesas General PWM Timer (GPT) HAL APIs in three operating modes:

- **Periodic**: change the period used to drive the user LED.
- **PWM**: change the PWM duty cycle used to adjust the user LED intensity.
- **One-Shot**: start a one-shot timer and report its expiration on the serial terminal.

This document covers only:

- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The reference application provides a serial-terminal menu that lets the user select one of these three modes.

## What use case does this SBB demonstrate?

The SBB demonstrates how an application can reuse a small GPT wrapper to configure and exercise Periodic, PWM, and One-Shot timer behavior.

### Periodic mode

The application accepts a period in milliseconds, derives the GPT source frequency, applies the configured source divider, converts the requested period into timer counts, validates the count against the supported timer width, opens and starts the Periodic GPT instance, and updates its period.

The reference application describes the visible result as a change in the frequency of the user LED.

### PWM mode

The application accepts a duty cycle from **0 to 100 percent**, opens and starts the PWM GPT instance, obtains the configured timer period, converts the requested percentage into duty-cycle counts, and applies the duty cycle to the configured GPT output pin.

The reference application describes the visible result as a change in the intensity of the user LED.

### One-Shot mode

The application opens and starts the One-Shot GPT instance and waits for the timer callback to report `TIMER_EVENT_CYCLE_END`. When the event is received, the application reports that the timer expired.

A software timeout protects the reference application from waiting indefinitely if the expected callback is not received.

## Hardware requirements

- 1 x FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1
- 1 x USB cable for programming and debugging

The supplied README does not specify an additional jumper wire for these four boards for GPT Periodic/PWM/One-Shot operation.

### FPB-RA2E3

To use J-Link On-Board VCOM for the serial terminal, close:

- **E25**
- **E28**
- **E46**
- **E47**

### FPB-RA4E2

No additional board-specific hardware connection is identified for this board in the supplied README.

### FPB-RA6E2

No additional board-specific hardware connection is identified for this board in the supplied README.

### FPB-RA8E1

No additional board-specific hardware connection is identified for this board in the supplied README.

## What does the SBB core code do?

The reusable SBB core is implemented in:

- `gpt_timer.c`
- `gpt_timer.h`

It wraps the GPT operations needed by the demonstration and provides board-dependent constants used by the application.

The SBB core:

- Opens a supplied GPT timer instance.
- Tracks the currently active GPT control instance.
- Starts a supplied GPT timer.
- Sets the Periodic timer period.
- Calculates and sets PWM duty-cycle counts.
- Selects the applicable GPT output pin for the supported board configuration.
- Defines board-specific maximum timer counts and displayed period ranges.
- Closes a GPT timer instance when it is open.
- Handles the One-Shot cycle-end callback.
- Exposes a flag indicating that the One-Shot timer has expired.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`gpt_timer.c` and `gpt_timer.h` are the reusable GPT-facing layer.

They contain the wrappers around `R_GPT_Open()`, `R_GPT_Start()`, `R_GPT_PeriodSet()`, `R_GPT_InfoGet()`, `R_GPT_DutyCycleSet()`, and `R_GPT_Close()`, as well as the One-Shot callback and board/configuration constants used by the demonstration.

### Demo/reference application code

`gpt_example_app.c` and `gpt_example_app.h` implement the interactive demonstration. This layer is responsible for:

- Initializing the UART console.
- Reading and displaying the FSP version.
- Displaying the three-mode menu.
- Reading and converting terminal input.
- Calculating Periodic-mode timer counts from milliseconds.
- Validating the requested period against the configured maximum counter value.
- Validating PWM duty cycle input.
- Selecting and sequencing the Periodic, PWM, and One-Shot demonstrations.
- Waiting for the One-Shot callback.
- Displaying operation and error information.

Its single application entry point is:

```c
void gpt_example_app_run(void);
```

The source states that this function is called from `hal_entry.c`.

## Which public functions should I look at first?

### `init_gpt_timer()`

```c
fsp_err_t init_gpt_timer(timer_ctrl_t * const p_timer_ctl,
                         timer_cfg_t const * const p_timer_cfg);
```

Stores the supplied control instance as the active GPT and calls `R_GPT_Open()` when the timer is not already open.

### `start_gpt_timer()`

```c
fsp_err_t start_gpt_timer(timer_ctrl_t * const p_timer_ctl);
```

Starts the selected GPT instance with `R_GPT_Start()`.

### `set_period_gpt_timer()`

```c
fsp_err_t set_period_gpt_timer(timer_ctrl_t * const p_ctrl,
                               uint32_t const period_counts);
```

Updates a timer period using `R_GPT_PeriodSet()`. The reference application uses this function in Periodic mode after converting the requested millisecond period to timer counts.

### `set_timer_duty_cycle()`

```c
fsp_err_t set_timer_duty_cycle(uint8_t duty_cycle_percent);
```

Gets the PWM GPT's current `period_counts`, calculates the requested percentage as timer counts, and applies it with `R_GPT_DutyCycleSet()` to `TIMER_PIN`.

### `deinit_gpt_timer()`

```c
void deinit_gpt_timer(timer_ctrl_t * const p_timer_ctl);
```

Closes the selected GPT instance if it is open.

### `user_gpt_one_shot_callback()`

```c
void user_gpt_one_shot_callback(timer_callback_args_t * p_args);
```

Sets `g_one_shot_expired` when the callback receives `TIMER_EVENT_CYCLE_END`.

## What configuration assumptions were made for the reference boards?

### Generated GPT instances

The reference application uses generated configurations for three GPT uses:

- `g_timer_periodic_ctrl` / `g_timer_periodic_cfg`
- `g_timer_pwm_ctrl` / `g_timer_pwm_cfg`
- `g_timer_one_shot_mode_ctrl` / `g_timer_one_shot_mode_cfg`

These configurations provide the timer modes, source divider, output configuration, callbacks, and other GPT settings consumed by the hand-written source.

### GPT output pin

For FPB-RA2E3, FPB-RA4E2, and FPB-RA6E2, `gpt_timer.h` selects:

```c
GPT_IO_PIN_GTIOCA
```

The FPB-RA8E1 is not included in that `TIMER_PIN` conditional and therefore uses:

```c
GPT_IO_PIN_GTIOCB
```

### Timer width used by the reference application

For FPB-RA4E2 and FPB-RA6E2, the header defines the maximum period count as:

```c
0xFFFF
```

For FPB-RA2E3 and FPB-RA8E1, it defines:

```c
0xFFFFFFFF
```

The reference application compares its calculated Periodic-mode count against this value before programming the timer.

### Period ranges from the supplied configuration

The supplied README and header define these Periodic-mode ranges:

| Board | Configured period range |
|---|---:|
| FPB-RA2E3 | 0 to 89478 ms |
| FPB-RA4E2 | 0 to 671 ms |
| FPB-RA6E2 | 0 to 671 ms |
| FPB-RA8E1 | 0 to 35791 ms |

These are documented as ranges for the supplied example configuration rather than universal GPT hardware limits.

### PWM range

The SBB defines:

```c
#define GPT_MAX_PERCENT (100U)
```

The reference application therefore accepts PWM duty-cycle input from **0 through 100 percent**.

### Console input

The input buffer size is 16 bytes. The supplied README states that user-entered data should not exceed **15 bytes** and that operation is not guaranteed for non-integer inputs such as floating-point values, characters, or special characters.

## What must I change for my board or application?

### GPT instances and modes

The three generated GPT configurations must match the intended Periodic, PWM, and One-Shot uses. If a different GPT channel or mode configuration is selected, update the generated FSP project configuration and ensure the hand-written code refers to the appropriate generated objects.

### PWM output pin

`TIMER_PIN` is board-dependent. If the PWM output moves to a different GPT output, validate the generated pin configuration and the `TIMER_PIN` selection used by `R_GPT_DutyCycleSet()`.

### Clock source and divider

Periodic-mode timing is calculated from the GPT source frequency and `g_timer_periodic_cfg.source_div`.

The reference application obtains the source frequency from GPTCLK when the BSP indicates GPTCLK is present and not bypassed. Otherwise it obtains PCLKD using `R_FSP_SystemClockHzGet()`. It then applies the configured GPT source divider before converting milliseconds to counts.

If system clocks, GPT source clocks, or the GPT source divider change, revalidate the resulting period range and count calculation.

### Timer width and permitted range

The supplied header applies different maximum counter values for the documented boards. If the timer channel/configuration changes, verify that the maximum period-count assumption and user-facing period range remain valid for the selected configuration.

### PWM period

`set_timer_duty_cycle()` derives duty-cycle counts from the PWM timer's current `period_counts`, obtained through `R_GPT_InfoGet()`. If the underlying PWM period changes, the duty count changes accordingly.

### One-Shot timing

The One-Shot period is provided by `g_timer_one_shot_mode_cfg`; the supplied hand-written source does not change that period at runtime. Change the generated One-Shot configuration if a different one-shot interval is needed.

### Memory

The supplied files do not provide a measured RAM or ROM footprint for the SBB.

### Task priorities and execution model

The supplied implementation does not define an RTOS or task priorities. The application runs a foreground loop and, during One-Shot operation, polls `g_one_shot_expired` until the callback sets it or a software timeout expires.

If integrated into an RTOS-based system, scheduling and synchronization behavior must be defined by that application.

### Console middleware

The interactive menu and standard I/O are reference-application functionality. The underlying GPT wrapper functions do not depend on the menu itself. An integrating application can invoke the reusable GPT operations through its own control path.

## Dependencies

### Renesas FSP GPT HAL

The reusable SBB directly uses:

- `R_GPT_Open()`
- `R_GPT_Start()`
- `R_GPT_InfoGet()`
- `R_GPT_DutyCycleSet()`
- `R_GPT_PeriodSet()`
- `R_GPT_Close()`

### Renesas FSP/BSP clock APIs

Periodic-mode count calculation in the reference application uses:

- `R_FSP_ClockDividerGet()` and `R_BSP_SourceClockHzGet()` when the applicable GPTCLK feature path is enabled.
- `R_FSP_SystemClockHzGet(FSP_PRIV_CLOCK_PCLKD)` otherwise.

### Generated FSP configuration

The SBB includes `hal_data.h` and depends on the generated Periodic, PWM, and One-Shot GPT configuration/control objects.

### Console/reference application

The demo layer uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `fgets()`
- `atoi()`
- `strlen()`
- `memset()`
- `strcspn()`
- `R_FSP_VersionGet()`

### Hardware resources

The reference implementation requires the GPT peripherals and configured GPT output used by the supplied board project. The Periodic and PWM modes use the user LED according to the supplied example description, while One-Shot completion is reported on the terminal.

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied application demonstrates:

- Selecting Periodic, PWM, or One-Shot GPT operation from a terminal menu.
- Changing the Periodic timer period from terminal input.
- Calculating timer counts from the configured source clock and source divider.
- Rejecting a Periodic-mode request when the calculated count exceeds `GPT_MAX_PERIOD_COUNT`.
- Setting PWM duty cycle from 0 through 100 percent.
- Calculating PWM duty-cycle counts from the current configured period.
- Starting a One-Shot timer and detecting its cycle-end callback.
- Detecting the absence of the expected One-Shot callback through a software timeout.
- Closing the currently active GPT before switching to another demonstration mode.

### Reference application ranges

The supplied configurations expose different period ranges on the four requested boards:

- FPB-RA2E3: **0 to 89478 ms**
- FPB-RA4E2: **0 to 671 ms**
- FPB-RA6E2: **0 to 671 ms**
- FPB-RA8E1: **0 to 35791 ms**

The PWM demonstration accepts **0 to 100 percent** duty cycle.

### Performance measurements

The supplied files do not report measured timer accuracy, PWM frequency accuracy, jitter, interrupt latency, CPU utilization, memory footprint, or One-Shot latency.

The documented millisecond ranges are properties of the supplied configuration, not measured GPT performance limits. This SBB should therefore be treated as a functional GPT reference rather than a performance benchmark.

## What files are safe to modify and what is generated, preset, or demo-only?

| File | Classification | Guidance |
|---|---|---|
| `gpt_timer.c` | Reusable SBB implementation | Modify when intentionally changing GPT wrappers, PWM calculation, active-timer handling, or One-Shot callback behavior. |
| `gpt_timer.h` | Reusable SBB interface and board/configuration presets | Modify carefully when changing public APIs, timer limits, output-pin selection, or displayed period ranges. |
| `gpt_example_app.c` | Demo/reference application | Appropriate place for changing menu behavior, input handling, demonstration sequencing, period-count calculation policy, or presentation. |
| `gpt_example_app.h` | Demo/reference application interface | Contains the demonstration entry point, menu, banner, and informational strings. |
| `readme.txt` | Existing example documentation | Source for supported-board configuration ranges, hardware notes, input restrictions, and terminal settings. |

The source also depends on generated FSP configuration through `hal_data.h`. The supplied file set does not enumerate those generated files individually, so this document does not classify unprovided generated files by name.

## What happens if I modify the source and later adopt a newer SBB release?

Changes to `gpt_timer.c` or `gpt_timer.h` become local modifications to the reusable SBB layer. A newer SBB release can modify the same interfaces, board conditionals, calculations, or callback logic, requiring those changes to be reconciled.

To reduce upgrade conflicts:

- Keep menu behavior, terminal parsing, and demonstration messages in `gpt_example_app.*`.
- Prefer using generated FSP configuration for timer channels, operating modes, periods, dividers, callbacks, and pin configuration when those settings belong to the generated project.
- Keep necessary SBB-core modifications focused and document why they are required.
- Keep generated FSP configuration separate from hand-maintained SBB source.
- When adopting a newer SBB release, compare the new `gpt_timer.c` and `gpt_timer.h` against the locally modified versions before carrying changes forward.
- Revalidate the board's timer width assumption, `TIMER_PIN`, supported period range, clock calculation, PWM behavior, One-Shot callback, and generated GPT configurations after an update.

The supplied files do not define an automated source-migration or compatibility mechanism for later SBB releases. Compatibility after an update must therefore be validated by the integrating project.

## Recommended reading order

1. **`gpt_timer.h`** to understand the reusable API, board-specific output selection, maximum timer count, and configured period ranges.
2. **`gpt_timer.c`** to understand GPT initialization, start, Periodic/PWM updates, cleanup, and One-Shot callback behavior.
3. **`gpt_example_app.c`** to see how Periodic, PWM, and One-Shot operations are selected and exercised.
4. **`gpt_example_app.h`** to see the demo entry point and terminal menu/information strings.
5. **`readme.txt`** for the original hardware, period-range, input, and serial-terminal information.
