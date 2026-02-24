# MacBook Pro Screen Switcher

## Objective

Create a one-click Dock app that toggles the macOS **Displays > "Optimize for"** setting between:

- **Gav MacBook** (built-in display)
- **LG TV SSCR 2** (external display)

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

### Implementation plan

1. **Install displayplacer** via `brew install displayplacer`
2. **Discover screen IDs** by running `displayplacer list` — identify the persistent IDs for "Gav MacBook" and "LG TV SSCR 2"
3. **Create a toggle shell script** (`toggle-display.sh`) that:
   - Runs `displayplacer list` to read the current mirror configuration
   - Detects which screen is currently first (i.e. the "Optimize for" target)
   - Swaps the order so the other screen becomes the "Optimize for" target
   - Applies the new config with `displayplacer "id:<newFirst>+<newSecond> ..."`
4. **Wrap in an Automator Application** (or a lightweight `.app` bundle) so it can be dragged to the Dock and launched with a single click
5. **Optional**: Add a macOS notification (`osascript -e 'display notification ...'`) to confirm which display is now active

### Dependencies

- macOS (tested on Sequoia / Sonoma)
- [displayplacer](https://github.com/jakehilborn/displayplacer) — `brew install displayplacer`
- Automator (built into macOS)

## Research Sources

- [displayplacer GitHub repo](https://github.com/jakehilborn/displayplacer) — the core CLI tool
- [displayplacer: switching primary display (Issue #19)](https://github.com/jakehilborn/displayplacer/issues/19)
- [Configure and switch macOS displays with displayplacer (Medium)](https://medium.com/macoclock/configure-and-switch-macos-displays-with-displayplacer-650c62c0f1bf)
- [toggle-mirror-display AppleScript (GitHub)](https://github.com/fayaz-dev/toggle-mirror-display) — alternative AppleScript approach
- [Apple Community: "Optimize for" menu bar discussion](https://discussions.apple.com/thread/254605826)
- [Jamf Community: displayplacer configurations](https://community.jamf.com/general-discussions-2/displayplacer-screen-resolutions-mirror-display-settings-and-configurations-33806)
