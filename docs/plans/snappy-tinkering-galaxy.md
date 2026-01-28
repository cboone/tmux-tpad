# Plan: Add `persist` option for session persistence

## Summary

Add a `persist` configuration option that keeps tpad sessions alive after the configured command exits, instead of destroying them with `; exit`.

## Current behavior

When a `cmd` is configured (e.g., `lazygit`), line 110 of `tpad.tmux` sends `"$cmd; exit"` to the session. When the command exits, `exit` destroys the session and the popup closes. Next invocation creates a fresh session.

## Proposed behavior

- New boolean config: `@tpad-<instance>-persist` (default: `false`)
- When `true`: send only `$cmd` (no `; exit`). After the command exits, the user gets a shell prompt and the session stays alive. Reopening the popup reattaches to the existing session.
- When `false` or unset: current behavior preserved.

## Changes

### 1. `tpad.tmux` — `configure_session()` (lines 103–112)

The only code change. Add a conditional around the `; exit` suffix:

```bash
configure_session() {
  local instance="$1"
  local session_id="$2"
  apply_session_config "$instance" "$session_id"

  local cmd="$(get_config "$instance" cmd)"
  if [[ -n "$cmd" ]]; then
    local persist="$(get_config "$instance" persist)"
    if [[ "$persist" == "true" ]]; then
      tmux send-keys -t "$session_id" "$cmd" C-m
    else
      tmux send-keys -t "$session_id" "$cmd; exit" C-m
    fi
  fi
}
```

No changes needed to `DEFAULTS`, `get_config`, `toggle_popup`, `create_session_if_needed`, or any other function. The `persist` key follows the same empty-string-is-falsy pattern as `per-dir`.

### 2. `README.md` — Documentation

- Add `persist` row to the Behavior Options table (alphabetically after `per-dir`)
- Add `persist` to the lazygit example configuration
- Update usage note: "The popup will close automatically when the command exits (unless `persist` is enabled)"
- Check off "Session persistence options" in the roadmap

## Edge cases

| Scenario | Behavior |
|---|---|
| No `cmd`, `persist` true | No effect — session is already a plain shell that persists naturally |
| `cmd` set, `persist` false/unset | Current behavior: `$cmd; exit` destroys session on command exit |
| `cmd` set, `persist` true | Command runs, shell prompt after exit, session stays alive |
| `per-dir` + `persist` | Works naturally — `create_session_if_needed` skips creation if session exists (line 95), so the command is not re-sent on reattach |

## Files to modify

- `/Users/hpg/Development/tmux-tpad__worktrees/feature-persist-sessions/tpad.tmux` — `configure_session()` on lines 103–112
- `/Users/hpg/Development/tmux-tpad__worktrees/feature-persist-sessions/README.md` — documentation updates

## Verification

Manual testing in tmux:

1. **Backward compat**: `cmd` without `persist` — popup closes when command exits (session destroyed)
2. **Persist with cmd**: `persist true` + `cmd "echo hello"` — popup stays open with shell prompt after echo
3. **Persist with interactive cmd**: `persist true` + `cmd "htop"` — quit htop, shell prompt appears, popup stays open
4. **Reattach**: toggle off/on with persist — same session reattached, no command re-execution
5. **Per-dir + persist**: different repos get different persistent sessions, reattach works per-repo
