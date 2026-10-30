# AGT (Periodic, One-shot)

## Overview

This Software Building Block (SBB) demonstrates the Renesas Asynchronous General Purpose Timer (AGT) in **One-Shot** and **Periodic** modes on the following reference boards:

- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The reference application accepts timer periods from a serial terminal. AGT channel 0 operates in One-Shot mode. When that timer expires, its callback starts AGT channel 1 in Periodic mode. The periodic timer then expires at the user-selected interval and toggles an on-board LED.

## Use case demonstrated by this SBB

The SBB demonstrates how two AGT instances can be combined to sequence timer-driven behavior:

1. Configure one AGT instance for One-Shot operation.
2. Configure a second AGT instance for Periodic operation.
3. Set both timer periods from user-selected millisecond values.
4. Start the One-Shot timer.
5. Start the Periodic timer from the One-Shot timer callback when the One-Shot period expires.
6. Toggle an on-board LED from the Periodic timer callback.
7. Query and stop active timers from the application.

The reference application accepts One-Shot and Periodic period values from **1 to 2000 ms**.

## Supported reference boards

This documentation covers only:

| Board | Reference application notes |
|---|---|
| FPB-RA2E3 | Close **E25, E28, E46, and E47** to use J-Link OB VCOM for the serial terminal. |
| FPB-RA4E2 | No additional board-specific hardware connection is identified in the supplied `readme.txt`. |
| FPB-RA6E2 | No additional board-specific hardware connection is identified in the supplied `readme.txt`. |
| FPB-RA8E1 | Open jumpers **J6 and J9** to use the on-board debug functionality. |

For all four boards, connect the board's USB debug port to the host PC using the appropriate USB cable for programming and debugging.

## How the demonstration works

At startup, the reference application:

1. Initializes the UART console.
2. Displays the SBB and FSP version information.
3. Opens both AGT timer instances.
4. Requests the One-Shot timer period.
5. Requests the Periodic timer period.
6. Obtains each timer's configured clock frequency using `agt_info_get()`.
7. Converts each millisecond period to raw timer counts using:

```text
period_counts = (time_period_ms * clock_frequency) / 1000
```

8. Programs the calculated periods into the two AGT instances.
9. Waits for terminal input to start or stop timer operation.

When timer operation is started, the application starts the One-Shot AGT instance. When its cycle ends, `one_shot_timer_callback()` starts the Periodic AGT instance unless that timer is already running. Each cycle-end event from the Periodic timer causes `periodic_timer_callback()` to toggle the first BSP-defined on-board LED.

## SBB core code

The reusable SBB implementation is contained in:

- `agt_ep.c`
- `agt_ep.h`

The SBB core wraps the FSP AGT operations required by the application and contains the timer callbacks used to sequence the One-Shot and Periodic timers.

Its responsibilities are:

- Open the One-Shot and Periodic AGT instances.
- Start an AGT instance.
- Retrieve AGT configuration/status information.
- Change an AGT period.
- Stop an AGT instance.
- Close both AGT instances.
- Start the Periodic timer when the One-Shot timer expires.
- Toggle the board LED when the Periodic timer expires.
- Report callback failures to the application through `g_error_flag`.

## Reusable SBB code versus demo/reference application

### Reusable SBB code

`agt_ep.c` and `agt_ep.h` form the reusable timer-facing layer.

They expose functions that wrap the underlying FSP `R_AGT` interface and contain the callbacks that implement the demonstrated timer relationship.

### Demo/reference application code

`agt_example_app.c` and `agt_example_app.h` implement the interactive reference application.

The reference application is responsible for:

- UART console initialization.
- Displaying SBB and FSP version information.
- Accepting timer periods from the terminal.
- Validating the demo's `1..2000 ms` input range.
- Converting milliseconds to timer counts using the timer's configured clock frequency.
- Starting or stopping timer operation in response to terminal input.
- Displaying timer state information.
- Handling errors and entering a breakpoint trap when an operation fails.

The reference application exposes a single entry point:

```c
void agt_example_app_run(void);
```

The source states that this function is called by `hal_entry.c`.

The UART user interface, input parsing, printed messages, interactive start/stop behavior, and `1..2000 ms` input policy belong to the reference application rather than the general AGT wrapper API.

## Public functions to look at first

### `agt_init()`

```c
fsp_err_t agt_init(void);
```

Opens the configured One-Shot timer (`g_timer_one_shot`) and Periodic timer (`g_timer_periodic`). This is the logical starting point for understanding how the two FSP AGT instances are brought into use.

### `agt_period_set()`

```c
fsp_err_t agt_period_set(timer_ctrl_t * const p_ctrl,
                         uint32_t const period_counts);
```

Updates the period of the selected AGT instance using `R_AGT_PeriodSet()`.

### `agt_start()`

```c
fsp_err_t agt_start(timer_ctrl_t * const p_ctrl);
```

Starts the selected timer using `R_AGT_Start()`.

### `agt_status_get()`

```c
fsp_err_t agt_status_get(timer_ctrl_t * const p_ctrl,
                         timer_status_t * const p_status);
```

Retrieves the current timer state and counter value.

### `agt_info_get()`

```c
fsp_err_t agt_info_get(timer_ctrl_t * const p_ctrl,
                       timer_info_t * const p_info);
```

Retrieves timer information. The reference application uses the returned `clock_frequency` to convert the requested millisecond period into timer counts.

### `agt_stop()`

```c
fsp_err_t agt_stop(timer_ctrl_t * const p_ctrl);
```

Stops the selected timer.

### `agt_deinit()`

```c
void agt_deinit(void);
```

Closes the One-Shot and Periodic AGT instances if they are open.

### Timer callbacks

Two callbacks implement the demonstrated behavior:

```c
void one_shot_timer_callback(timer_callback_args_t * p_args);
void periodic_timer_callback(timer_callback_args_t * p_args);
```

`one_shot_timer_callback()` reacts to `TIMER_EVENT_CYCLE_END`, checks whether the Periodic timer is already counting, and starts it when necessary.

`periodic_timer_callback()` reacts to `TIMER_EVENT_CYCLE_END` and toggles the first LED defined by the BSP LED configuration through `R_IOPORT_PinWrite()`.

## Reference configuration assumptions

The supplied code assumes two generated AGT instances:

| Purpose | Generated control object | Generated configuration object |
|---|---|---|
| One-Shot timer | `g_timer_one_shot_ctrl` | `g_timer_one_shot_cfg` |
| Periodic timer | `g_timer_periodic_ctrl` | `g_timer_periodic_cfg` |

The following additional assumptions are visible in the supplied implementation:

- The One-Shot and Periodic behaviors are established by the generated FSP configurations supplied to `R_AGT_Open()`.
- Timer periods are programmed as raw counts after reading `clock_frequency` from each timer with `R_AGT_InfoGet()`.
- The reference application's accepted period is greater than `0` and no greater than `2000` ms.
- The Periodic callback uses the first LED in `g_bsp_leds.p_leds[]`.
- Callback processing depends on AGT cycle-end events.
- The reference application uses a UART-backed standard I/O console.
- The terminal-input buffer is 16 bytes; the supplied README states that input should not exceed 15 bytes.

The supplied files do not state the exact AGT source clock, divider, interrupt priority, or timer-channel configuration for each of the four reference boards. Those settings therefore need to be verified in the generated FSP configuration of the relevant board project.

## What must change when adapting the SBB?

The reusable API does not hard-code a board-specific timer frequency. Instead, the reference application queries each timer's actual `clock_frequency` and derives raw counts from it. When integrating the SBB into another board or application, the generated FSP configuration and application assumptions below must be reviewed.

### Timer configuration

The project must provide the generated control/configuration objects expected by the SBB and configure one timer for One-Shot operation and one for Periodic operation if the demonstrated behavior is to be retained.

The supplied implementation specifically references:

- `g_timer_one_shot_ctrl`
- `g_timer_one_shot_cfg`
- `g_timer_periodic_ctrl`
- `g_timer_periodic_cfg`

### Clocks and timing

The example obtains each AGT instance's clock frequency at runtime and calculates raw counts from the requested milliseconds. If the AGT clock source or divider changes, the value returned through `R_AGT_InfoGet()` is therefore significant to the period calculation.

The supplied files do not specify clock-tree settings for FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1. Refer to the generated project configuration when validating those settings.

### LED and pins

The Periodic callback writes to:

```c
g_bsp_leds.p_leds[RESET_FLAG]
```

where `RESET_FLAG` is `0`. The target BSP therefore needs an appropriate LED definition if this demonstration behavior is retained. If the target application does not use an LED, the Periodic callback behavior should be replaced by the application's required periodic action.

### UART and middleware

The reference application depends on `console/uart_console.h` and standard I/O for its interactive demonstration. If the SBB is integrated without this demo, the terminal user interface does not need to be retained unless the application requires it.

For FPB-RA2E3, the supplied README explicitly requires E25, E28, E46, and E47 to be closed when J-Link OB VCOM is used for the serial terminal.

### Memory and task priorities

The supplied source and README do not specify memory requirements, stack/heap sizing, an RTOS, or task priorities. No board-specific memory or scheduling requirement can therefore be stated from these files. These aspects must be evaluated in the context of the application that integrates the SBB.

## Dependencies

### Renesas FSP and BSP

The implementation directly uses:

- `R_AGT_Open()`
- `R_AGT_Start()`
- `R_AGT_InfoGet()`
- `R_AGT_PeriodSet()`
- `R_AGT_StatusGet()`
- `R_AGT_Stop()`
- `R_AGT_Close()`
- `R_IOPORT_PinWrite()`
- `R_FSP_VersionGet()` in the reference application
- Types and generated configuration exposed through `hal_data.h`
- BSP LED information through `g_bsp_leds`

### Console/reference application

The demo also uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- Standard C library functions including `printf()`, `fgets()`, `atoi()`, `memset()`, and `strcspn()`

### Hardware resources

The demonstrated implementation requires:

- Two configured AGT instances
- An on-board LED for the supplied Periodic callback behavior
- USB debug connection
- J-Link On-Board serial/VCOM when using the supplied terminal-based demonstration

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |
| Port | Port provided by J-Link On-Board |

For Tera Term, it additionally specifies:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

The supplied README states that terminal data should not exceed 15 bytes and that operation is not guaranteed for unsupported user-input forms such as floating-point or special-character input.

## Demonstrated behavior and performance

### Demonstrated behavior

The supplied implementation demonstrates:

- Opening two AGT instances.
- Configuring timer periods after initialization.
- Converting millisecond periods to timer counts using each timer's reported clock frequency.
- Starting and stopping timers.
- Reading timer status.
- One-Shot expiration initiating Periodic operation.
- Detecting that the Periodic timer is already running.
- Periodically toggling an on-board LED from the Periodic timer callback.
- Reporting callback errors to the foreground application through a shared error flag.

### Demonstrated input range

The reference application accepts **1 to 2000 ms** for both the One-Shot and Periodic timer periods.

### Performance measurements

The supplied source and README provide **no measured timing accuracy, interrupt latency, jitter, CPU utilization, memory footprint, maximum event rate, or other benchmark data**. The `1..2000 ms` range is an application input constraint, not a stated measurement of AGT hardware performance.

Accordingly, this SBB should be treated as a functional reference for AGT One-Shot/Periodic operation rather than a performance benchmark.

## File ownership and modification guidance

| File | Classification | Guidance |
|---|---|---|
| `agt_ep.c` | Reusable SBB implementation | Change when intentionally modifying the reusable AGT behavior or callback implementation. |
| `agt_ep.h` | Reusable SBB interface | Change when modifying the reusable interface, constants, or integration contract. |
| `agt_example_app.c` | Demo/reference application | Appropriate place for demo-specific UI, input, sequencing, and presentation changes. |
| `agt_example_app.h` | Demo/reference application interface | Contains the demo entry point, version, banner, information text, and user guide strings. |
| `readme.txt` | Existing example documentation | Source documentation for board setup, hardware requirements, and serial-terminal instructions. |

The supplied source includes `hal_data.h` and references generated FSP objects, but the supplied file set does not enumerate the generated files themselves. Generated FSP configuration should therefore remain separate from the hand-maintained reusable SBB and example application layers.

## Adopting a newer SBB release after local modifications

If `agt_ep.c` or `agt_ep.h` are changed locally, upgrading to a newer SBB may require reconciling local changes with changes made in the newer version.

To make future adoption easier:

- Keep application-specific UART interaction and user-interface policy in the reference/application layer rather than moving those concerns into `agt_ep.c`.
- Keep local changes to the reusable SBB explicit and narrowly scoped.
- Preserve the distinction between generated FSP configuration and hand-maintained SBB source.
- When adopting a newer SBB release, compare the new `agt_ep.c` and `agt_ep.h` against the locally modified versions before carrying customizations forward.
- Revalidate timer configuration, clock-derived period calculations, callbacks, interrupt behavior, board LED setup, and the application-specific integration after merging changes.

The supplied files do not define a source migration, compatibility, or automated upgrade mechanism. Compatibility with later SBB versions should therefore be validated when an update is adopted.

## Recommended reading order

For a customer evaluating the SBB:

1. **`agt_ep.h`**: understand the reusable API and integration-facing definitions.
2. **`agt_ep.c`**: see the FSP AGT wrappers and One-Shot/Periodic callbacks.
3. **`agt_example_app.c`**: see how timer periods are obtained, converted to counts, configured, and exercised interactively.
4. **`agt_example_app.h`**: see the reference application's entry point and displayed information.
5. **`readme.txt`**: review the original board and terminal setup requirements.

This separation allows the timer building block to be understood independently of the UART-driven demonstration used to exercise it.
