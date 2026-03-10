# Visual Tools PID & Tooltips — Design Document

**Date:** 2026-02-27
**Status:** Approved

## Problem

When the PowerBuilder IDE is open alongside the PMIX app, both windows can match the same title pattern (e.g. `PMI.*Login.*PMIGEST`). This causes `ElementAmbiguousError` on all visual MCP tools (`pb_screenshot_window`, `pb_list_controls`, `pb_interact_control`, `pb_save_reference`, `pb_visual_compare`). Additionally, toolbar buttons in PB apps often have no text labels, making them impossible to identify without tooltip extraction.

## Approach

Incremental, backwards-compatible improvements. No breaking changes — all existing parameters remain, new parameters are optional.

## Changes

### 1. `pb_launch_app` — Return PID

Add `pid` (integer) to the response JSON.

**Response (after):**
```json
{
  "status": "launched",
  "exe": "pmix.exe",
  "pid": 34772,
  "window_title": "PMI - Login PMIGEST",
  "window_class": "FNWND3",
  "position": { "left": 0, "top": 0, "width": 1920, "height": 1040 }
}
```

Python: `win.process_id()` from pywinauto.

### 2. Optional `pid` Parameter on All Visual Tools

Add optional `pid` (integer) parameter to these 5 tools:
- `pb_screenshot_window`
- `pb_list_controls`
- `pb_interact_control`
- `pb_save_reference`
- `pb_visual_compare`

**Connection logic in bridge Python:**
```python
if pid:
    app = pywinauto.Application(backend="win32").connect(process=pid)
    if window_title:
        win = app.window(title_re=window_title)
    else:
        win = app.top_window()
elif window_title:
    app = pywinauto.Application(backend="win32").connect(title_re=window_title)
    win = app.top_window()
else:
    raise ValueError("Either pid or window_title must be provided")
```

**Validation rules:**
- At least one of `pid` or `window_title` is required
- If both provided, connect by PID then filter by title (double safety)
- `window_title` becomes optional in Zod schema when `pid` is provided

**Typical workflow:**
```
1. pb_launch_app → returns pid: 34772
2. pb_screenshot_window(pid=34772)
3. pb_list_controls(pid=34772)
4. pb_interact_control(pid=34772, action="click", control_class="Button", control_text="Base")
```

### 3. `pb_list_controls` — Tooltips & Enrichment

**3a. Tooltip extraction**

Two-level strategy:
1. Try `ctrl.element_info.rich_text` or accessible properties
2. If empty, return `tooltip: ""`

No automatic hover simulation (too slow and unstable).

**3b. `visible` field per control**

Add `visible: boolean` to each control in the output, even when `visible_only=false`.

**3c. `pid` in response**

Include the PID of the matched window in the response.

**Response (after):**
```json
{
  "window": "PMI - Login PMIGEST",
  "pid": 34772,
  "control_count": 42,
  "class_summary": { ... },
  "controls": [
    {
      "index": 3,
      "class_name": "Button",
      "text": "Base",
      "tooltip": "Gestion de base",
      "enabled": true,
      "visible": true,
      "rect": { "left": 192, "top": 81, "width": 45, "height": 20 }
    }
  ]
}
```

### 4. `pb_interact_control` — New Action & Better Errors

**4a. New `get_tooltip` action**

Add `get_tooltip` to the allowed actions enum.

```json
{ "action": "get_tooltip", "pid": 34772, "control_class": "Button", "control_index": 0 }
```

Response:
```json
{ "status": "ok", "tooltip": "Gestion des societes" }
```

**4b. Better `control_index` error messages**

```python
if control_index >= len(matching):
    raise IndexError(
        f"control_index={control_index} out of range: "
        f"found {len(matching)} controls matching "
        f"class='{control_class}' text='{control_text}'. "
        f"Available: {[c.window_text()[:40] for c in matching]}"
    )
```

**4c. `pid` in response**

Include the PID in all action responses.

## Summary

| # | Tool | Change |
|---|------|--------|
| 1 | `pb_launch_app` | Returns `pid` in response |
| 2 | 5 visual tools | Optional `pid` parameter for unambiguous connection |
| 3 | `pb_list_controls` | `tooltip`, `visible`, `pid` fields in response |
| 4 | `pb_interact_control` | `get_tooltip` action, better error messages, `pid` in response |

## Files to Modify

- `packages/mcp-server/src/tools/visual.ts` — Zod schemas + tool definitions
- `packages/mcp-server/src/visual-bridge.py` — Python automation logic
- `packages/mcp-server/src/__tests__/visual.test.ts` — Tests

## Backwards Compatibility

All changes are additive. No existing parameter changes. No existing behavior changes. Tools called without `pid` work exactly as before.
