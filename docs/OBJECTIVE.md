# MacBook Pro Screen Switcher

## Objective

Create a one-click Dock app that toggles the macOS **Displays > "Optimize for"** setting between:

- **Built-in display** (e.g. "Gav MacBook")
- **External display** (e.g. "LG TV SSCR 2")

This removes the need to navigate through System Settings > Displays every time.

## How It Works

macOS has an "Optimize for" dropdown in **System Settings > Displays** when displays are mirrored. This controls which display drives the resolution/refresh-rate for the mirrored set. Changing it manually requires: System Settings > Displays > click the "Optimize for" dropdown > select the other display. This project automates that into a single click.

## Technical Approach

### Core tool: `displayplacer`

[displayplacer](https://github.com/jakehilborn/displayplacer) is an open-source macOS CLI utility (installable via Homebrew) that configures multi-display resolutions and arrangements from the command line.

Key behavior for mirrored displays:
- **The first `screenId` in a mirroring set becomes the "Optimize for" screen** in System Settings.
- Mirror sets are expressed with `+` between screen IDs: `id:<screenA>+<screenB>`
- Swapping the order of the IDs swaps which display is "optimized for".

> **⚠ Unverified assumption:** The "first ID = optimize-for target" behavior is sourced from community posts and issue threads, not official displayplacer documentation. This must be experimentally confirmed before building on it. **Step 1 of implementation is to install displayplacer and verify this behavior hands-on.**

#### Why displayplacer over AppleScript UI automation?

An [AppleScript approach](https://github.com/fayaz-dev/toggle-mirror-display) exists that drives the System Settings UI directly, eliminating the third-party dependency. However:

| | displayplacer (CLI) | AppleScript (UI automation) |
|---|---|---|
| **Reliability** | Talks to macOS display APIs directly | Fragile — breaks when Apple redesigns System Settings UI (which happens frequently) |
| **Speed** | Instant | Slow — must open/navigate System Settings |
| **Accessibility permissions** | Not required | Required — must grant Accessibility access |
| **Dependency** | Third-party Homebrew package | None (built into macOS) |
| **macOS version sensitivity** | Low — display APIs are stable | High — UI layout changes across versions |

**Decision:** Use displayplacer. The API-level approach is more robust across macOS updates, faster to execute, and doesn't require Accessibility permissions. The Homebrew dependency is a one-time install.

### Implementation plan

1. **Install displayplacer** via `brew install displayplacer`
2. **Verify the core assumption** — confirm that swapping screen ID order in a mirror set actually changes the "Optimize for" target in System Settings. Document the exact working command.
3. **Create a toggle shell script** (`toggle-display.sh`) that:
   - Runs `displayplacer list` to read the current mirror configuration
   - **Dynamically discovers screen IDs** by matching on display type (built-in vs external) or display name — never hardcodes IDs, since they can change after reboots, cable reconnections, or macOS updates
   - Detects which screen is currently first (i.e. the "Optimize for" target)
   - **Captures the full current display mode** (resolution, refresh rate, color depth, scaling) so it can be replayed exactly — not just the screen ID order
   - Swaps the order so the other screen becomes the "Optimize for" target
   - Applies the new config with the complete displayplacer command, e.g.:
     ```
     displayplacer "id:<newFirst>+<newSecond> res:<width>x<height> hz:<refresh> color_depth:<depth> enabled:true scaling:<mode> origin:(0,0)"
     ```
   - **Handles the external display being disconnected** — if only the built-in display is detected, shows a macOS notification ("External display not found") and exits cleanly instead of failing silently
4. **Wrap as a Dock-launchable `.app` bundle** (see section below)
5. **Add a macOS notification** (`osascript -e 'display notification ...'`) to confirm which display is now the "Optimize for" target

### App bundle approach

A minimal `.app` bundle (no Automator or Shortcuts dependency) is the most portable and future-proof approach:

```
ScreenSwitcher.app/
  Contents/
    Info.plist          # App metadata (CFBundleExecutable, icon, etc.)
    MacOS/
      toggle-display    # The shell script (chmod +x), runs on launch
    Resources/
      AppIcon.icns      # Optional icon
```

This avoids Automator (effectively deprecated — Apple has shifted to Shortcuts since Monterey and Automator receives no updates) and Shortcuts (which has limited shell script support and adds unnecessary complexity). A bare `.app` bundle is a single directory that macOS treats as a clickable application — no framework dependency, no build step, and it can be dragged straight to the Dock.

**Gatekeeper / code signing:** Since this is an unsigned app, macOS will block it on first launch. The user must right-click > Open (or remove the quarantine attribute via `xattr -d com.apple.quarantine ScreenSwitcher.app`). This is a one-time step. If broader distribution is ever needed, the app can be signed with a Developer ID certificate.

### Configuration

Display names are configurable via environment variables at the top of the toggle script, with sensible defaults:

```bash
BUILTIN_DISPLAY="${BUILTIN_DISPLAY:-built-in}"    # substring match against displayplacer output
EXTERNAL_DISPLAY="${EXTERNAL_DISPLAY:-LG TV}"      # substring match against displayplacer output
```

The script matches displays by searching `displayplacer list` output for these substrings, making it work across screen ID changes and adaptable to different external displays without modifying the script itself.

### Dependencies

- macOS (tested on Sequoia / Sonoma)
- [displayplacer](https://github.com/jakehilborn/displayplacer) — `brew install displayplacer`

## Research Sources

- [displayplacer GitHub repo](https://github.com/jakehilborn/displayplacer) — the core CLI tool
- [displayplacer: switching primary display (Issue #19)](https://github.com/jakehilborn/displayplacer/issues/19)
- [Configure and switch macOS displays with displayplacer (Medium)](https://medium.com/macoclock/configure-and-switch-macos-displays-with-displayplacer-650c62c0f1bf)
- [toggle-mirror-display AppleScript (GitHub)](https://github.com/fayaz-dev/toggle-mirror-display) — alternative AppleScript approach (evaluated and rejected; see comparison above)
- [Apple Community: "Optimize for" menu bar discussion](https://discussions.apple.com/thread/254605826)
- [Jamf Community: displayplacer configurations](https://community.jamf.com/general-discussions-2/displayplacer-screen-resolutions-mirror-display-settings-and-configurations-33806)
