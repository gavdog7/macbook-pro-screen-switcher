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
- **Status:** NOT STARTED
- **Evidence:** —

## Step 4: Wrap as .app bundle
- **Status:** NOT STARTED
- **Evidence:** —

## Step 5: Add macOS notification
- **Status:** NOT STARTED
- **Evidence:** —
