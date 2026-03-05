# Implementation Progress

## Step 1: Install displayplacer
- **Status:** COMPLETE
- **Evidence:** `brew install displayplacer` succeeded (v1.4.0). `displayplacer list` returns both displays:
  - External: `C886DD0D-6852-429A-9EE3-2EBC45EA6C75` (72 inch external screen, 3840x2160 @100Hz)
  - Built-in: `37D8832A-2D66-02CA-B9F7-8F30A301B230` (MacBook built in screen, 3840x2160 @120Hz)
  - Displays are currently mirrored with external first in ID order.

## Step 2: Verify core assumption (mirror ID order = "Optimize for" target)
- **Status:** COMPLETE
- **Evidence:** Confirmed by human reviewer. With external display first (`id:C886DD0D...+37D8832A...`), System Settings showed "Optimize for: LG TV SSCR2". After swapping to built-in first (`id:37D8832A...+C886DD0D...` using `mode:132`), System Settings changed to "Optimize for: Gav-MacBook". displayplacer `--help` also documents this: "The first screenId in a mirroring set will be the 'Optimize for' screen."
- **Key finding:** When swapping to built-in-first, must use `mode:` syntax rather than `res:` with `scaling:off`, because the built-in display's native modes don't list `scaling:off` explicitly. The `mode:` approach is more reliable.
- **Key finding:** Swapping the "optimize for" target also changes the available resolution set — the "optimize for" display drives the resolution options for the mirror set.

## Step 3: Create toggle-display.sh script
- **Status:** COMPLETE
- **Evidence:** Script created at `toggle-display.sh`. Tests:
  1. Toggle external→built-in: built-in became first in mirror set (confirmed via `displayplacer list`)
  2. Toggle built-in→external: external became first, restored 3840x2160 @100Hz
  3. Round-trip: both directions work cleanly with exit 0
  4. Error handling: `EXTERNAL_DISPLAY="nonexistent-display"` → exits 1 with notification
- **Bugs fixed during development:**
  - Removed `set -e` — `grep -qi` non-match exits non-zero, killing the script
  - Changed default `BUILTIN_MATCH` from `built-in` to `built.in` (regex) — displayplacer reports "built in" (no hyphen)

## Step 4: Wrap as .app bundle
- **Status:** NOT STARTED
- **Evidence:** —

## Step 5: Add macOS notification
- **Status:** COMPLETE (integrated into Step 3)
- **Evidence:** Notifications are built into the toggle script via `osascript -e 'display notification ...'`. Shows "Now optimized for: Built-in Display" or "Now optimized for: External Display" on success, and descriptive error messages on failure.
