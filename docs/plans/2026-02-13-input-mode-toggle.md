# `input_mode` Toggle Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the boolean `is_button` config option with an `input_mode` enum (`switch`/`pulse`/`toggle`) so toggle-button hardware (like Shelly Plus 1) can stop movement by pulsing the same direction button.

**Architecture:** Add `input_mode` to the YAML schema, resolve legacy `is_button` with deprecation warnings in `devices_from_config`, replace `self._is_button` with `self._input_mode` throughout the class, and update `_async_handle_command` to handle toggle-mode stop by pulsing the last-used direction button.

**Tech Stack:** Home Assistant custom component (Python), voluptuous for schema validation.

**Design doc:** `docs/plans/2026-02-13-input-mode-toggle-design.md`

---

### Task 1: Add `input_mode` config constant and schema

**Files:**
- Modify: `custom_components/cover_time_based/cover.py:36-88`

**Step 1: Add the new config constant**

After line 52 (`CONF_IS_BUTTON = "is_button"`), add:

```python
CONF_INPUT_MODE = "input_mode"
INPUT_MODE_SWITCH = "switch"
INPUT_MODE_PULSE = "pulse"
INPUT_MODE_TOGGLE = "toggle"
```

**Step 2: Update SWITCH_COVER_SCHEMA**

Replace the `SWITCH_COVER_SCHEMA` (lines 75-82) with:

```python
SWITCH_COVER_SCHEMA = {
    **BASE_DEVICE_SCHEMA,
    vol.Required(CONF_OPEN_SWITCH_ENTITY_ID): cv.entity_id,
    vol.Required(CONF_CLOSE_SWITCH_ENTITY_ID): cv.entity_id,
    vol.Optional(CONF_STOP_SWITCH_ENTITY_ID, default=None): vol.Any(cv.entity_id, None),
    vol.Optional(CONF_IS_BUTTON, default=False): cv.boolean,
    vol.Optional(CONF_INPUT_MODE, default=None): vol.Any(
        vol.In([INPUT_MODE_SWITCH, INPUT_MODE_PULSE, INPUT_MODE_TOGGLE]), None
    ),
    **TRAVEL_TIME_SCHEMA,
}
```

**Step 3: Commit**

```bash
git add custom_components/cover_time_based/cover.py
git commit -m "feat: add input_mode config constant and schema"
```

---

### Task 2: Update `devices_from_config` to resolve `input_mode`

**Files:**
- Modify: `custom_components/cover_time_based/cover.py:139-220`

**Step 1: Replace `is_button` extraction with `input_mode` resolution**

Replace line 196:
```python
        is_button = config.pop(CONF_IS_BUTTON) if CONF_IS_BUTTON in config else False
```

With:
```python
        is_button = config.pop(CONF_IS_BUTTON) if CONF_IS_BUTTON in config else False
        input_mode = config.pop(CONF_INPUT_MODE, None) if CONF_INPUT_MODE in config else None

        if input_mode is not None and is_button:
            _LOGGER.warning(
                "Device '%s': both 'is_button' and 'input_mode' are set. "
                "'input_mode: %s' takes precedence. Please remove 'is_button'.",
                device_id, input_mode,
            )
        elif is_button:
            input_mode = INPUT_MODE_PULSE
            _LOGGER.warning(
                "Device '%s': 'is_button' is deprecated. "
                "Use 'input_mode: pulse' instead.",
                device_id,
            )
        elif input_mode is None:
            input_mode = INPUT_MODE_SWITCH
```

**Step 2: Update the CoverTimeBased constructor call**

Replace line 217 (`is_button,`) with `input_mode,` in the constructor call:

```python
        device = CoverTimeBased(
            device_id,
            name,
            travel_moves_with_tilt,
            travel_time_down,
            travel_time_up,
            tilt_time_down,
            tilt_time_up,
            travel_delay_at_end,
            min_movement_time,
            travel_startup_delay,
            tilt_startup_delay,
            open_switch_entity_id,
            close_switch_entity_id,
            stop_switch_entity_id,
            input_mode,
            cover_entity_id,
        )
```

**Step 3: Commit**

```bash
git add custom_components/cover_time_based/cover.py
git commit -m "feat: resolve is_button to input_mode with deprecation warnings"
```

---

### Task 3: Update constructor to use `input_mode`

**Files:**
- Modify: `custom_components/cover_time_based/cover.py:239-277`

**Step 1: Replace `is_button` parameter and field**

In the `__init__` method, replace the parameter name `is_button` (line 255) with `input_mode`, and replace the field assignment `self._is_button = is_button` (line 274) with `self._input_mode = input_mode`:

Change line 255 from:
```python
        is_button,
```
to:
```python
        input_mode,
```

Change line 274 from:
```python
        self._is_button = is_button
```
to:
```python
        self._input_mode = input_mode
```

**Step 2: Commit**

```bash
git add custom_components/cover_time_based/cover.py
git commit -m "refactor: replace is_button with input_mode in constructor"
```

---

### Task 4: Update `_async_handle_command` for all three modes

**Files:**
- Modify: `custom_components/cover_time_based/cover.py:1106-1231`

This is the core change. Replace the entire `_async_handle_command` method with the version below. Key changes:
- `self._is_button` checks become `self._input_mode in (INPUT_MODE_PULSE, INPUT_MODE_TOGGLE)`
- The `SERVICE_STOP_COVER` branch gets new toggle logic: pulse the last-used direction button

**Step 1: Replace `_async_handle_command`**

Replace the entire method (lines 1106-1231) with:

```python
    async def _async_handle_command(self, command, *args):
        if command == SERVICE_CLOSE_COVER:
            cmd = "DOWN"
            self._state = False
            if self._cover_entity_id is not None:
                await self.hass.services.async_call(
                    "cover",
                    "close_cover",
                    {"entity_id": self._cover_entity_id},
                    False,
                )
            else:
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_off",
                    {"entity_id": self._open_switch_entity_id},
                    False,
                )
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_on",
                    {"entity_id": self._close_switch_entity_id},
                    False,
                )
                if self._stop_switch_entity_id is not None:
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._stop_switch_entity_id},
                        False,
                    )

                if self._input_mode in (INPUT_MODE_PULSE, INPUT_MODE_TOGGLE):
                    await sleep(1)

                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._close_switch_entity_id},
                        False,
                    )

        elif command == SERVICE_OPEN_COVER:
            cmd = "UP"
            self._state = True
            if self._cover_entity_id is not None:
                await self.hass.services.async_call(
                    "cover",
                    "open_cover",
                    {"entity_id": self._cover_entity_id},
                    False,
                )
            else:
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_off",
                    {"entity_id": self._close_switch_entity_id},
                    False,
                )
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_on",
                    {"entity_id": self._open_switch_entity_id},
                    False,
                )
                if self._stop_switch_entity_id is not None:
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._stop_switch_entity_id},
                        False,
                    )
                if self._input_mode in (INPUT_MODE_PULSE, INPUT_MODE_TOGGLE):
                    await sleep(1)

                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._open_switch_entity_id},
                        False,
                    )

        elif command == SERVICE_STOP_COVER:
            cmd = "STOP"
            self._state = True
            if self._cover_entity_id is not None:
                await self.hass.services.async_call(
                    "cover",
                    "stop_cover",
                    {"entity_id": self._cover_entity_id},
                    False,
                )
            elif self._input_mode == INPUT_MODE_TOGGLE:
                # Toggle mode: pulse the last-used direction button to stop
                if self._last_command == SERVICE_CLOSE_COVER:
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_on",
                        {"entity_id": self._close_switch_entity_id},
                        False,
                    )
                    await sleep(1)
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._close_switch_entity_id},
                        False,
                    )
                elif self._last_command == SERVICE_OPEN_COVER:
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_on",
                        {"entity_id": self._open_switch_entity_id},
                        False,
                    )
                    await sleep(1)
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_off",
                        {"entity_id": self._open_switch_entity_id},
                        False,
                    )
                else:
                    _LOGGER.debug(
                        "_async_handle_command :: STOP in toggle mode with no last command, skipping"
                    )
            else:
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_off",
                    {"entity_id": self._close_switch_entity_id},
                    False,
                )
                await self.hass.services.async_call(
                    "homeassistant",
                    "turn_off",
                    {"entity_id": self._open_switch_entity_id},
                    False,
                )
                if self._stop_switch_entity_id is not None:
                    await self.hass.services.async_call(
                        "homeassistant",
                        "turn_on",
                        {"entity_id": self._stop_switch_entity_id},
                        False,
                    )

                    if self._input_mode == INPUT_MODE_PULSE:
                        await sleep(1)

                        await self.hass.services.async_call(
                            "homeassistant",
                            "turn_off",
                            {"entity_id": self._stop_switch_entity_id},
                            False,
                        )

        _LOGGER.debug("_async_handle_command :: %s", cmd)

        self.async_write_ha_state()
```

**Step 2: Verify no remaining references to `self._is_button`**

Search for `_is_button` in the file — there should be zero occurrences.

**Step 3: Commit**

```bash
git add custom_components/cover_time_based/cover.py
git commit -m "feat: implement toggle stop logic in _async_handle_command"
```

---

### Task 5: Update README documentation

**Files:**
- Modify: `README.md:123`

**Step 1: Replace the `is_button` row in the config table**

Replace line 123:
```
| is_button              | boolean      | _Optional_ (`cover_entity_id` not supported)    | Treats the switches as buttons, only pressing them for 1s                 | False   |
```

With:
```
| input_mode             | string       | _Optional_ (`cover_entity_id` not supported)    | How switches are controlled: `switch` (latching), `pulse` (momentary with separate stop), `toggle` (momentary, same button stops) | switch  |
| is_button              | boolean      | _Deprecated_                                     | Use `input_mode: pulse` instead. Kept for backward compatibility          | False   |
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README with input_mode option, deprecate is_button"
```

---

### Task 6: Lint, type-check, and deploy

**Step 1: Run ruff**

```bash
cd /workspaces/ha-cover-time-based
ruff check .
ruff format .
```

Fix any issues found.

**Step 2: Run pyright**

```bash
cd /workspaces/ha-cover-time-based
npx pyright
```

Fix any type errors that can be fixed.

**Step 3: Commit any formatting fixes**

```bash
git add -A
git commit -m "chore: fix lint and formatting"
```

**Step 4: Deploy to HA for manual testing**

```bash
rm -Rf /workspaces/homeassistant-core/config/custom_components/fado && cp -r /workspaces/ha-cover-time-based/custom_components/cover_time_based /workspaces/homeassistant-core/config/custom_components/
```

Note: This copies the cover_time_based component, not fado. Adjust the destination path if the HA instance expects a different location.

---

### Task 7: Create PR

**Step 1: Push branch and create PR**

```bash
git push -u origin feat/input-mode-toggle
gh pr create --title "feat: add input_mode with toggle support for Shelly-style buttons" --body "$(cat <<'EOF'
## Summary
- Replaces boolean `is_button` with `input_mode` enum: `switch` (default), `pulse` (was `is_button: true`), `toggle` (new)
- Toggle mode: stop command pulses the last-used direction button, fixing Shelly Plus 1 and similar hardware where the same button press starts and stops movement
- `is_button` remains in schema for backward compatibility with deprecation warnings

## Test plan
- [ ] Verify existing `is_button: true` configs still work (mapped to `input_mode: pulse`)
- [ ] Verify `is_button: false` configs still work (mapped to `input_mode: switch`)
- [ ] Verify deprecation warning is logged when `is_button` is used
- [ ] Test `input_mode: toggle` with Shelly Plus 1: open to partial position, verify it stops
- [ ] Test `input_mode: toggle` direction change: open then close mid-travel
- [ ] Test `input_mode: toggle` manual stop button in HA UI

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```
