# Plan: Configurable Binding Table

Allow users to specify which tmux key table a tpad binding is created in via a new `table` option, enabling root-table bindings, mouse events, and custom tables. Mouse event names are auto-detected and default to the `root` table.

## Code Change: `bind_key()` in `tpad.tmux` (lines 153-160)

Current:

```bash
bind_key() {
  local instance="$1"
  local key="$(get_config "$instance" bind)"
  [[ -z "$key" ]] && return

  tmux bind-key "$key" run-shell "$TPAD_SCRIPT toggle $instance"
  tmux bind-key -T "tpad_$instance" "$key" run-shell "$TPAD_SCRIPT toggle $instance"
}
```

New:

```bash
bind_key() {
  local instance="$1"
  local key="$(get_config "$instance" bind)"
  [[ -z "$key" ]] && return

  local table="$(get_config "$instance" table)"

  # Mouse events always require the root table
  if [[ -z "$table" ]]; then
    case "$key" in
      Mouse*|DoubleClick*|TripleClick*|WheelUp*|WheelDown*) table="root" ;;
    esac
  fi

  if [[ -n "$table" ]]; then
    tmux bind-key -T "$table" "$key" run-shell "$TPAD_SCRIPT toggle $instance"
  else
    tmux bind-key "$key" run-shell "$TPAD_SCRIPT toggle $instance"
  fi

  tmux bind-key -T "tpad_$instance" "$key" run-shell "$TPAD_SCRIPT toggle $instance"
}
```

**Design notes:**
- Auto-detection only applies when `table` is not explicitly set, so the user can always override.
- The internal popup binding (`tpad_<instance>` table) is unchanged — it controls toggle-close inside the popup and is independent of how the popup is opened.
- This code runs once at plugin init time; no performance concern.

## Documentation: `README.md`

1. Add `table` row to the **Behavior Options** table (after line 65):

   ```
   | table   |         | Key table for the binding (e.g., `root`). Auto-detected for mouse events. |
   ```

2. Add a **mouse/root-table binding example** to the Example Configuration section:

   ```tmux
   # Right-click to open a scratchpad (table auto-detected as root)
   set -g @tpad-scratch-bind    "MouseDown3Pane"

   # Keyboard shortcut without prefix key (explicit root table)
   set -g @tpad-quick-bind      "C-Space"
   set -g @tpad-quick-table     "root"
   ```

## Why No Other Changes Are Needed

- `get_config()` already handles arbitrary keys and returns empty string for unset options.
- `DEFAULTS` does not need a `table` entry (empty = prefix table, or auto-detected for mouse).
- Instance discovery (`awk` in `initialize_instances`) already correctly deduplicates across all `@tpad-<instance>-*` options.

## Verification

1. **Backward compat**: Reload plugin with no `table` option set and a keyboard binding. Confirm prefix + key still opens popup.
2. **Mouse auto-detect**: Set `@tpad-scratch-bind "MouseDown3Pane"` with no `table` option. Confirm right-click opens popup (auto-detected as root).
3. **Root table keyboard**: Set `@tpad-test-table "root"` and `@tpad-test-bind "C-Space"`. Confirm C-Space opens popup without prefix.
4. **Explicit override**: Set both `@tpad-test-bind "MouseDown3Pane"` and `@tpad-test-table "some-custom-table"`. Confirm the explicit table is used, not `root`.
5. **Toggle close**: In all cases above, confirm pressing the same key/mouse inside the popup closes it.
6. **Per-dir + table**: Combine `per-dir` with a root-table binding to ensure they compose correctly.
