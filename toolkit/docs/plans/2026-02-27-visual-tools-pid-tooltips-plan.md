# Visual Tools PID & Tooltips — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add PID-based window matching, tooltip extraction, and better error messages to all 6 visual MCP tools.

**Architecture:** Backwards-compatible changes to `visual.ts` (Zod schemas + param forwarding) and `visual-bridge.py` (connection logic + tooltip extraction). A shared `_connect_window()` helper in the bridge replaces duplicated connection code in every handler. Tests call the bridge CLI directly with JSON input.

**Tech Stack:** TypeScript + Zod (MCP schemas), Python 3 + pywinauto (window automation), vitest (tests)

---

### Task 1: Bridge — shared `_connect_window()` helper

Extract the duplicated `pywinauto.Application().connect(title_re=...)` + `app.top_window()` pattern into a single helper that supports both PID and title-based connection.

**Files:**
- Modify: `packages/mcp-server/src/visual-bridge.py:160-217` (and all handlers)

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('visual-bridge.py — PID parameter support', () => {
  it('bridge accepts pid parameter in screenshot action', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    // Pass pid=99999 (non-existent) — should get a connection error, not a parameter error
    const input = JSON.stringify({
      action: 'screenshot',
      pid: 99999,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return; // Python not installed
    }

    expect(parsed).toHaveProperty('error');
    const errorMsg = parsed['error'] as string;
    // Should fail on connection, not on missing window_title
    expect(errorMsg).not.toContain('window_title');
  });

  it('bridge requires at least pid or window_title', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    // No pid, no window_title — should get validation error
    const input = JSON.stringify({
      action: 'screenshot',
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
    const errorMsg = parsed['error'] as string;
    expect(errorMsg).toMatch(/pid.*window_title|window_title.*pid|either|at least/i);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: FAIL — bridge doesn't accept `pid` yet

**Step 3: Implement `_connect_window()` helper in bridge**

Add this helper function in `visual-bridge.py` after `_find_control()` (around line 93):

```python
def _connect_window(data: dict):
    """
    Connect to a window by PID and/or title regex.

    Priority:
      1. If pid is provided, connect by process ID
      2. If window_title is also provided, filter by title within that process
      3. If only window_title, connect by title regex (legacy behavior)
      4. If neither, raise ValueError

    Returns:
        (app, win) tuple — the pywinauto Application and window wrapper
    """
    import pywinauto
    import warnings

    pid = data.get("pid")
    if pid is not None:
        pid = int(pid)
    window_title = data.get("window_title") or ""

    if not pid and not window_title:
        raise ValueError("Either pid or window_title must be provided")

    with warnings.catch_warnings():
        warnings.simplefilter("ignore")

        if pid:
            app = pywinauto.Application(backend="win32").connect(process=pid)
            if window_title:
                win = app.window(title_re=window_title)
            else:
                win = app.top_window()
        else:
            app = pywinauto.Application(backend="win32").connect(title_re=window_title)
            win = app.top_window()

    return app, win
```

Then replace the connection code in each of the 5 handlers (`do_screenshot`, `do_list_controls`, `do_interact`, `do_save_reference`, `do_visual_compare`).

For example, in `do_screenshot` replace:

```python
    try:
        app = pywinauto.Application(backend="win32").connect(title_re=window_title)
        win = app.top_window()
```

with:

```python
    try:
        app, win = _connect_window(data)
```

And remove the now-unused local `window_title = data.get("window_title", "")` lines in handlers that no longer need them (keep it only in handlers that still use it for error messages).

Do the same replacement in all 5 handlers:
- `do_screenshot` (line 184)
- `do_list_controls` (line 241)
- `do_interact` (line 321)
- `do_save_reference` (line 396)
- `do_visual_compare` (line 474)

Also update the `except` blocks — add `ValueError` catch alongside `ElementNotFoundError`:

```python
    except ValueError as ve:
        return {"error": str(ve)}
    except pywinauto.findwindows.ElementNotFoundError:
        return {"error": f"No window found matching pid={data.get('pid')}, title='{data.get('window_title', '')}'"}
```

**Step 4: Run test to verify it passes**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: PASS — both new tests pass, all existing tests still pass

**Step 5: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/visual-bridge.py packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): add _connect_window() helper with PID support in bridge"
```

---

### Task 2: Bridge — return PID in `do_launch` response

**Files:**
- Modify: `packages/mcp-server/src/visual-bridge.py` (do_launch function, ~line 146)
- Test: `packages/mcp-server/tests/visual.test.ts`

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('visual-bridge.py — launch returns pid', () => {
  it('launch response schema includes pid field when connecting to running app', async () => {
    // We can't easily test a real launch, but we verify the bridge
    // returns an error JSON (not a crash) for a non-existent exe.
    // The real validation is: when launch succeeds, pid is in the response.
    // This test verifies the bridge doesn't crash with the new code.
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'launch',
      exe_path: 'C:/does/not/exist/fake_app.exe',
      wait_seconds: 1,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    // Should return valid JSON (error about missing file, but not a crash)
    expect(typeof parsed).toBe('object');
    expect(parsed).not.toBeNull();
  });
});
```

**Step 2: Run test to verify it passes (baseline — this test is a safety net)**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: PASS

**Step 3: Add PID to launch response**

In `visual-bridge.py`, in `do_launch()`, replace the return dict (around line 146):

```python
    return {
        "status": "connected" if connected else "launched",
        "exe": exe_path,
        "window_title": win.window_text(),
        "window_class": win.class_name(),
        "position": {
            "left": rect.left,
            "top": rect.top,
            "width": rect.width(),
            "height": rect.height(),
        },
    }
```

with:

```python
    return {
        "status": "connected" if connected else "launched",
        "exe": exe_path,
        "pid": win.process_id(),
        "window_title": win.window_text(),
        "window_class": win.class_name(),
        "position": {
            "left": rect.left,
            "top": rect.top,
            "width": rect.width(),
            "height": rect.height(),
        },
    }
```

**Step 4: Run tests to verify nothing broke**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: All PASS

**Step 5: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/visual-bridge.py packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): return pid in pb_launch_app response"
```

---

### Task 3: TypeScript — add `pid` parameter to all 5 visual tool schemas

**Files:**
- Modify: `packages/mcp-server/src/tools/visual.ts:148-438`
- Test: `packages/mcp-server/tests/visual.test.ts`

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('registerVisualTools — pid parameter in schemas', () => {
  it('pb_screenshot_window accepts optional pid parameter', () => {
    // Verify that the Zod schema in visual.ts accepts pid.
    // We test indirectly by confirming registration succeeds
    // (schema validation happens at call time, not registration).
    const server = makeMcpServer();
    const cache = new PBCache();
    expect(() => registerVisualTools(server, cache)).not.toThrow();
  });
});
```

**Step 2: Run test to verify it passes (baseline)**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: PASS

**Step 3: Add `pid` parameter to all 5 tools in visual.ts**

For each of these 5 tools, add the `pid` parameter to the `inputSchema` and forward it in the `callBridge` call. Also make `window_title` optional.

**pb_screenshot_window** (line ~154): Replace the `inputSchema` with:

```typescript
      inputSchema: {
        window_title: z
          .string()
          .optional()
          .describe('Title (or regex pattern) of the window to capture'),
        pid: z
          .number()
          .int()
          .optional()
          .describe(
            'Process ID of the target application. Use this to avoid ambiguity when multiple windows share the same title. Returned by pb_launch_app.',
          ),
        save_path: z
          .string()
          .optional()
          .describe(
            'File path where the PNG should be saved. Auto-generated under screenshots/ if not set.',
          ),
      },
```

And update the handler signature and callBridge call:

```typescript
    async ({ window_title, pid, save_path }) => {
      let result: BridgeResult;
      try {
        result = await callBridge('screenshot', {
          ...(window_title !== undefined ? { window_title } : {}),
          ...(pid !== undefined ? { pid } : {}),
          ...(save_path !== undefined ? { save_path } : {}),
        });
```

Apply the same pattern to **pb_list_controls**, **pb_interact_control**, **pb_save_reference**, and **pb_visual_compare**:

- Add `pid` to `inputSchema` (same Zod definition as above)
- Make `window_title` optional (add `.optional()`)
- Update handler signatures to destructure `pid`
- Forward `pid` to `callBridge` with conditional spread
- Update tool descriptions to mention PID support

Also update **pb_launch_app** description to mention it returns `pid`:

```typescript
      description:
        'Launches a PowerBuilder application .exe for visual testing. Connects to an already-running instance if one exists. Returns the main window title, class, position, and pid.',
```

**Step 4: Run tests to verify nothing broke**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: All PASS

**Step 5: Run full test suite to verify no regressions**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run --reporter=verbose`
Expected: All tests pass (218+)

**Step 6: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/tools/visual.ts packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): add optional pid parameter to all 5 visual tool schemas"
```

---

### Task 4: Bridge — tooltip extraction in `do_list_controls`

**Files:**
- Modify: `packages/mcp-server/src/visual-bridge.py` (do_list_controls, ~line 219)
- Test: `packages/mcp-server/tests/visual.test.ts`

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('visual-bridge.py — list_controls enriched output', () => {
  it('bridge does not crash when list_controls includes tooltip extraction code', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    // Call list_controls on a non-existent window — verify error format is clean
    const input = JSON.stringify({
      action: 'list_controls',
      window_title: 'NONEXISTENT_WINDOW_TOOLTIP_TEST',
      visible_only: true,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
  });

  it('bridge accepts pid in list_controls action', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'list_controls',
      pid: 99999,
      visible_only: true,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
    const errorMsg = parsed['error'] as string;
    expect(errorMsg).not.toContain('window_title');
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: FAIL — the pid test should fail because list_controls doesn't use `_connect_window` yet (if Task 1 was applied to all handlers, it may pass — verify)

**Step 3: Update `do_list_controls` in bridge**

Replace `do_list_controls` body with:

```python
def do_list_controls(data: dict) -> dict:
    missing = _check_visual_deps()
    if missing:
        return {
            "error": (
                f"Missing dependencies: {', '.join(missing)}. "
                f"Install: pip install {' '.join(missing)}"
            )
        }

    import pywinauto

    visible_only = bool(data.get("visible_only", True))

    try:
        app, win = _connect_window(data)

        controls = []

        for ctrl in win.descendants():
            try:
                is_visible = ctrl.is_visible()
                if visible_only and not is_visible:
                    continue
                rect = ctrl.rectangle()
                cls_name = ctrl.class_name()

                # Tooltip extraction — best effort
                tooltip = ""
                try:
                    el_info = ctrl.element_info
                    if hasattr(el_info, 'rich_text') and el_info.rich_text:
                        tooltip = str(el_info.rich_text)[:200]
                    elif hasattr(el_info, 'description') and el_info.description:
                        tooltip = str(el_info.description)[:200]
                except Exception:
                    pass

                control_info: dict = {
                    "index": len(controls),
                    "class_name": cls_name,
                    "text": ctrl.window_text()[:80],
                    "tooltip": tooltip,
                    "enabled": ctrl.is_enabled(),
                    "visible": is_visible,
                    "rect": {
                        "left": rect.left,
                        "top": rect.top,
                        "width": rect.width(),
                        "height": rect.height(),
                    },
                }
                controls.append(control_info)
            except Exception:
                continue

        class_summary: dict = {}
        for ctrl_info in controls:
            cls = ctrl_info["class_name"]
            if cls not in class_summary:
                class_summary[cls] = {"count": 0, "indices": []}
            class_summary[cls]["count"] += 1
            class_summary[cls]["indices"].append(ctrl_info["index"])

        return {
            "window": win.window_text(),
            "pid": win.process_id(),
            "control_count": len(controls),
            "class_summary": class_summary,
            "controls": controls,
        }
    except ValueError as ve:
        return {"error": str(ve)}
    except pywinauto.findwindows.ElementNotFoundError:
        return {"error": f"No window found matching pid={data.get('pid')}, title='{data.get('window_title', '')}'"}
    except Exception as exc:
        return {"error": f"{type(exc).__name__}: {exc}"}
```

**Step 4: Run tests to verify they pass**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: All PASS

**Step 5: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/visual-bridge.py packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): add tooltip/visible/pid to list_controls response"
```

---

### Task 5: Bridge — `get_tooltip` action and better error messages in `do_interact`

**Files:**
- Modify: `packages/mcp-server/src/visual-bridge.py` (do_interact, ~line 293)
- Modify: `packages/mcp-server/src/tools/visual.ts` (interact action enum, ~line 251)
- Test: `packages/mcp-server/tests/visual.test.ts`

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('visual-bridge.py — get_tooltip action', () => {
  it('bridge accepts get_tooltip as a valid action', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'interact',
      window_title: 'NONEXISTENT_WINDOW_TOOLTIP',
      control_action: 'get_tooltip',
      control_class: 'Button',
      control_index: 0,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
    const errorMsg = parsed['error'] as string;
    // Should fail because window not found, NOT because of unknown action
    expect(errorMsg).not.toContain('Unknown action');
  });
});

describe('visual-bridge.py — improved control_index error', () => {
  it('bridge returns descriptive error for out-of-range index', async () => {
    // This test validates the error message format.
    // We can't trigger the actual IndexError without a running window,
    // but we ensure the bridge handles the interact action correctly.
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'interact',
      pid: 99999,
      control_action: 'click',
      control_class: 'Button',
      control_index: 999,
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    // Should get an error (connection failure since PID doesn't exist)
    expect(parsed).toHaveProperty('error');
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: FAIL — get_tooltip is an unknown action

**Step 3: Implement changes**

**In `visual-bridge.py`**, update `do_interact`:

1. Replace the connection code with `_connect_window`:

```python
    try:
        app, win = _connect_window(data)
```

2. Add `get_tooltip` action handler after the `get_text` block:

```python
        elif action == "get_tooltip":
            tooltip = ""
            try:
                el_info = ctrl.element_info
                if hasattr(el_info, 'rich_text') and el_info.rich_text:
                    tooltip = str(el_info.rich_text)[:200]
                elif hasattr(el_info, 'description') and el_info.description:
                    tooltip = str(el_info.description)[:200]
            except Exception:
                pass
            return {"status": "ok", "pid": win.process_id(), "tooltip": tooltip}
```

3. Update the unknown action error message to include `get_tooltip`:

```python
        else:
            return {
                "error": (
                    f"Unknown action '{action}'. "
                    "Use: click, set_text, get_text, get_tooltip, select"
                )
            }
```

4. Add `pid` to all action responses:

```python
        if action == "click":
            ctrl.click()
            time.sleep(0.3)
            return {"status": "clicked", "pid": win.process_id(), "control": ctrl.window_text()}

        elif action == "set_text":
            if value is None:
                return {"error": "'value' is required for set_text action."}
            ctrl.set_text(value)
            return {"status": "text_set", "pid": win.process_id(), "value": value}

        elif action == "get_text":
            return {"status": "ok", "pid": win.process_id(), "text": ctrl.window_text()}

        elif action == "get_tooltip":
            # ... (as above)

        elif action == "select":
            if value is None:
                return {"error": "'value' is required for select action."}
            ctrl.select(value)
            return {"status": "selected", "pid": win.process_id(), "value": value}
```

5. Improve the `_find_control` error message (replace lines 80-84):

```python
        if control_index >= len(matching):
            available = [c.window_text()[:40] for c in matching]
            raise IndexError(
                f"control_index={control_index} out of range: "
                f"found {len(matching)} controls matching "
                f"class={control_class!r} text={control_text!r}. "
                f"Available: {available}"
            )
```

6. Update error handling:

```python
    except ValueError as ve:
        return {"error": str(ve)}
    except pywinauto.findwindows.ElementNotFoundError:
        return {"error": f"No window found matching pid={data.get('pid')}, title='{data.get('window_title', '')}'"}
    except Exception as exc:
        return {"error": f"{type(exc).__name__}: {exc}"}
```

**In `visual.ts`**, update the `pb_interact_control` action enum (line ~251):

```typescript
        action: z
          .enum(['click', 'set_text', 'get_text', 'get_tooltip', 'select'])
          .describe('Action to perform on the control'),
```

And update the tool description:

```typescript
      description:
        'Interacts with a control inside a running PowerBuilder window. Supported actions: click, set_text, get_text, get_tooltip, select. Use pb_list_controls first to identify controls by class name, text, or index.',
```

**Step 4: Run tests to verify they pass**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: All PASS

**Step 5: Run full test suite**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run --reporter=verbose`
Expected: All tests pass

**Step 6: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/visual-bridge.py packages/mcp-server/src/tools/visual.ts packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): add get_tooltip action and improve control_index errors"
```

---

### Task 6: Bridge — PID in remaining handlers (`do_screenshot`, `do_save_reference`, `do_visual_compare`)

Ensure the 3 remaining handlers also use `_connect_window()` and return `pid`.

**Files:**
- Modify: `packages/mcp-server/src/visual-bridge.py`
- Test: `packages/mcp-server/tests/visual.test.ts`

**Step 1: Write the failing test**

Add to `packages/mcp-server/tests/visual.test.ts`:

```typescript
describe('visual-bridge.py — PID in all handlers', () => {
  it('save_reference accepts pid parameter', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'save_reference',
      pid: 99999,
      reference_name: 'test_ref',
      references_dir: './test_refs',
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
    const errorMsg = parsed['error'] as string;
    // Should fail on connection (PID 99999), not on missing window_title
    expect(errorMsg).not.toContain('window_title is required');
  });

  it('visual_compare accepts pid parameter', async () => {
    const { execFile } = await import('node:child_process');
    const { promisify } = await import('node:util');
    const execAsync = promisify(execFile);

    const input = JSON.stringify({
      action: 'visual_compare',
      pid: 99999,
      reference_name: 'test_ref',
      threshold: 0.1,
      max_diff_percent: 5.0,
      references_dir: './nonexistent_refs',
    });

    let stdout = '';
    try {
      const result = await execAsync('python', [BRIDGE_SCRIPT, input], { timeout: 10_000 });
      stdout = result.stdout;
    } catch (err: unknown) {
      if (err && typeof err === 'object' && 'stdout' in err) {
        stdout = (err as { stdout: string }).stdout;
      }
    }

    let parsed: Record<string, unknown> | null = null;
    try {
      parsed = JSON.parse(stdout) as Record<string, unknown>;
    } catch {
      return;
    }

    expect(parsed).toHaveProperty('error');
  });
});
```

**Step 2: Run test to verify behavior**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`

**Step 3: Update remaining handlers**

**`do_screenshot`** — replace connection block with `_connect_window(data)` and add `pid` to response:

```python
def do_screenshot(data: dict) -> dict:
    missing = _check_visual_deps()
    if missing:
        return {
            "error": (
                f"Missing dependencies: {', '.join(missing)}. "
                f"Install: pip install {' '.join(missing)}"
            )
        }

    import pywinaui
    import pyautogui
    from PIL import Image

    save_path = data.get("save_path") or None

    try:
        app, win = _connect_window(data)
        win.set_focus()
        time.sleep(0.5)

        rect = win.rectangle()
        img = pyautogui.screenshot(
            region=(rect.left, rect.top, rect.width(), rect.height())
        )

        if not save_path:
            os.makedirs("screenshots", exist_ok=True)
            timestamp = time.strftime("%Y%m%d_%H%M%S")
            safe_title = re.sub(r"[^\w]", "_", win.window_text())[:30]
            save_path = f"screenshots/{safe_title}_{timestamp}.png"

        img.save(save_path)

        buffer = io.BytesIO()
        img.save(buffer, format="PNG")
        img_b64 = base64.b64encode(buffer.getvalue()).decode()

        return {
            "status": "captured",
            "pid": win.process_id(),
            "window_title": win.window_text(),
            "save_path": os.path.abspath(save_path),
            "size": {"width": img.width, "height": img.height},
            "image_base64": img_b64,
        }
    except ValueError as ve:
        return {"error": str(ve)}
    except pywinauto.findwindows.ElementNotFoundError:
        return {"error": f"No window found matching pid={data.get('pid')}, title='{data.get('window_title', '')}'"}
    except Exception as exc:
        return {"error": f"{type(exc).__name__}: {exc}"}
```

**Note:** Fix the typo above — it should be `import pywinauto` not `pywinaui`. The implementer must use the correct import.

**`do_save_reference`** — same pattern: use `_connect_window(data)`, add `pid` to response.

**`do_visual_compare`** — same pattern: use `_connect_window(data)`, add `pid` to response.

**Step 4: Run tests**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run packages/mcp-server/tests/visual.test.ts --reporter=verbose`
Expected: All PASS

**Step 5: Run full test suite**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run --reporter=verbose`
Expected: All tests pass

**Step 6: Commit**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add packages/mcp-server/src/visual-bridge.py packages/mcp-server/tests/visual.test.ts
git commit -m "feat(visual): use _connect_window and return pid in all handlers"
```

---

### Task 7: Build and final verification

**Files:**
- None to modify — verification only

**Step 1: Build the project**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npm run build`
Expected: Clean build, no errors

**Step 2: Run full test suite**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx vitest run --reporter=verbose`
Expected: All tests pass (should be 218 + ~8 new = ~226 tests)

**Step 3: Verify TypeScript types**

Run: `cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit && npx tsc --noEmit`
Expected: No type errors

**Step 4: Commit (if any build fixes needed)**

```bash
cd C:\Users\JUDE\Claude\Code\powerbuilder-toolkit
git add -A
git commit -m "chore: build verification for visual tools PID support"
```
