# Icon Behaviour Diagnosis

## The problem

After toggling, the Dock icon shows the **opposite** display to what the user expects.

## Current state snapshot

| Fact | Value |
|---|---|
| Current "Optimize for" target | External (LG TV) — `C886DD0D...` is first in mirror set |
| Current AppIcon.icns | `icon-monitor.icns` (1,200,874 bytes — LG C2 monitor image) |
| Dock is showing | LG C2 monitor image |

So right now: optimized for LG TV, Dock shows LG TV icon.

## Current code logic (lines 87–105 of toggle-display)

```
if FIRST_ID == EXTERNAL_ID:
    → switch to BUILT-IN
    → set icon to MACBOOK
else:
    → switch to EXTERNAL
    → set icon to MONITOR
```

In plain English:
- **After switching to built-in** → Dock shows **MacBook**
- **After switching to external (LG TV)** → Dock shows **LG TV monitor**

The icon always reflects **what you just switched to** (the current state).

## Why it feels backwards

There are two valid mental models for what the icon should mean:

### Model A: "Status indicator" (current implementation)
> The icon shows **what is currently active**.
> - LG TV icon = "You are currently optimized for the LG TV"
> - MacBook icon = "You are currently optimized for the MacBook"

### Model B: "Action button"
> The icon shows **what will happen when you click it** (the next action).
> - LG TV icon = "Click me to switch to the LG TV"
> - MacBook icon = "Click me to switch to the MacBook"

**These are opposite to each other.** If you're currently on the LG TV:
- Model A shows: LG TV icon (this is what's active)
- Model B shows: MacBook icon (this is what you'll switch to)

## Which model does the user expect?

Based on the feedback ("I want the icon to display the picture of the LG TV when the LG TV button has been clicked" + "it's still backwards"), the expected behaviour needs clarification:

### Interpretation 1: "When I click and it switches to LG TV, show me the LG TV"
→ This is **Model A** (status indicator) — the current implementation.
→ If this is right but it still feels wrong, the bug may be elsewhere (e.g., icon cache, wrong icon file, timing issue with killall Dock).

### Interpretation 2: "The icon IS the button — LG TV icon means 'click to go to LG TV'"
→ This is **Model B** (action button).
→ The fix would be to swap the icon assignments:
  - After switching to built-in → show **LG TV** icon (click again = go to LG TV)
  - After switching to external → show **MacBook** icon (click again = go to MacBook)

## Decision needed

**Which model do you want?**

| | When optimized for LG TV | When optimized for MacBook |
|---|---|---|
| **Model A** (status — "I am on...") | Shows LG TV icon | Shows MacBook icon |
| **Model B** (action — "Click to go to...") | Shows MacBook icon | Shows LG TV icon |

Please indicate **A** or **B** and I will update the code accordingly.

## Possible non-logic causes

If Model A is correct but it still looks wrong, these could also be at play:

1. **Dock icon cache** — `killall Dock` might not always flush the icon cache reliably. The Dock may be showing a stale icon from a previous run.
2. **Timing** — `killall Dock` runs before the display actually finishes switching, so you might briefly see the old icon.
3. **Icon file mixup** — it's possible `icon-monitor.icns` and `icon-macbook.icns` got swapped during one of the regeneration passes. Worth visually confirming each file matches its name.
