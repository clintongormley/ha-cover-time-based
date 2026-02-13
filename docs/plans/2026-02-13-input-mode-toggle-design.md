# Design: `input_mode` with Toggle Support

## Problem

The `is_button` option supports hardware that uses momentary pulses (press to start, separate stop button to stop). However, some hardware — like Shelly Plus 1 relays configured as buttons — uses toggle behavior: pressing the same direction button again stops movement, and pressing the opposite direction immediately reverses.

With the current `is_button: true`, the stop command turns off both switches — but they're already off after the initial pulse. The stop is effectively a no-op, so the cover runs until it hits its mechanical endstop.

## Solution

Replace the boolean `is_button` with an `input_mode` enum that supports three behaviors:

| `input_mode` | Behavior | Replaces |
|---|---|---|
| `switch` | Turn on to start, turn off to stop (latching) | `is_button: false` (default) |
| `pulse` | Momentary pulse to start, separate stop pulse | `is_button: true` |
| `toggle` | Pulse same direction button to start and stop | **new** |

### Toggle mode behavior

- **Open/Close commands**: Same as `pulse` — pulse the direction button (turn on, wait 1s, turn off).
- **Stop command**: Pulse the **last-used direction button** (the one that started the movement). This tells the hardware to stop.
- **Direction change**: The existing stop-then-start pattern works. Stop pulses the current direction button (motor stops), then the new direction is pulsed (motor starts other way). This is preferred for motor health over immediate reversal.

### Stop command logic for toggle mode

```
last_command == CLOSE → pulse close_switch (1s)
last_command == OPEN  → pulse open_switch (1s)
last_command == None  → no-op (already stopped)
```

## Configuration

```yaml
cover:
  - platform: cover_time_based
    devices:
      terrace_pergola:
        name: Terrace Pergola
        open_switch_entity_id: switch.terrace_pergola_open
        close_switch_entity_id: switch.terrace_pergola_close
        input_mode: toggle
        travelling_time_down: 110
        travelling_time_up: 110
```

### Migration from `is_button`

- If `is_button` is present and `input_mode` is not: map `true` → `pulse`, `false` → `switch`
- Log a deprecation warning when `is_button` is encountered
- If both are specified, `input_mode` takes precedence and a warning is logged
- `is_button` remains in the schema for backward compatibility

## Changes

All changes are in `cover.py`:

1. **Schema**: Add `input_mode` as `vol.In(["switch", "pulse", "toggle"])` with default `switch`. Keep `is_button` as optional.
2. **`devices_from_config`**: Resolve `is_button` → `input_mode` mapping with deprecation warnings. Pass `input_mode` to constructor instead of `is_button`.
3. **Constructor**: Replace `self._is_button` with `self._input_mode`.
4. **`_async_handle_command`**: Replace `if self._is_button:` checks with `if self._input_mode in ("pulse", "toggle"):` for open/close, and add toggle-specific stop logic that pulses the last-used direction button.

## Scope exclusions

- Pulse duration remains hardcoded at 1 second (can be addressed in a separate PR per issue #26).
- Suppressing stop at endpoints for covers with mechanical endstops (issue #14) is a separate concern.
- `input_mode` only applies to switch-based configurations, not `cover_entity_id` configurations.
