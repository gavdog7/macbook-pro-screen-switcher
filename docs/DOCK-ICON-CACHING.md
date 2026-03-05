# Dock Icon Caching: Problem & Solution

## The problem

When changing a `.app` bundle's icon programmatically and restarting the Dock via `killall Dock`, the Dock continues to show the **old icon**. This makes it appear that the icon swap didn't work, even though the correct `.icns` file is on disk.

## Root cause

macOS has **three independent icon caches** that all need to be invalidated:

| Cache | Location | Purpose |
|---|---|---|
| **LaunchServices database** | System-managed (accessed via `lsregister`) | Maps bundle IDs to icon metadata |
| **IconServices cache** | `/Library/Caches/com.apple.iconservices.store`, `/var/folders/.../com.apple.iconservices/` | Renders and caches icon images at various sizes |
| **Dock bookmark data** | `~/Library/Preferences/com.apple.dock.plist` → `persistent-apps[n]` | Stores a binary bookmark blob per pinned app that includes cached icon data |

The critical one is the **Dock bookmark data**. Each entry in the Dock's `persistent-apps` array contains a `book` field with a binary bookmark. When the Dock restarts after `killall Dock`, it resolves these bookmarks — which can return **cached icon data** rather than re-reading from the `.app` bundle on disk.

## What doesn't work

| Approach | Why it fails |
|---|---|
| `cp new.icns AppIcon.icns && killall Dock` | Dock re-reads its cached bookmark, not the file on disk |
| `touch Bundle.app && killall Dock` | Bookmark resolution ignores mtime changes |
| Clearing `/var/folders/.../com.apple.dock.iconcache` | This path doesn't exist on modern macOS (Sequoia) |
| `NSWorkspace.setIcon` via JXA alone | Sets the extended attribute correctly, but the Dock still uses its bookmark cache |
| `lsregister -f` alone | Updates LaunchServices but not the Dock's own cache |

## What works: Delete and re-create the Dock plist entry

The only reliable approach is to **destroy the Dock's cached bookmark** by deleting the app's entry from the `persistent-apps` array in the Dock plist, then re-creating it at the same position. This forces the Dock to build a fresh bookmark with a fresh icon lookup when it restarts.

### Implementation

```bash
update_dock_icon() {
    local icon_src="$1"
    local dock_plist="$HOME/Library/Preferences/com.apple.dock.plist"
    local app_url="file://${BUNDLE_DIR}/"

    # A. Set the icon via multiple mechanisms for correctness
    cp "$icon_src" "$RESOURCES_DIR/AppIcon.icns"                     # CFBundleIconFile
    osascript -l JavaScript -e "                                      # Extended attribute
        ObjC.import('AppKit');
        var img = \$.NSImage.alloc.initWithContentsOfFile('${icon_src}');
        \$.NSWorkspace.sharedWorkspace.setIconForFileOptions(img, '${BUNDLE_DIR}', 0);
    " 2>/dev/null
    touch "$BUNDLE_DIR" "$BUNDLE_DIR/Contents/Info.plist"            # Update timestamps
    lsregister -f "$BUNDLE_DIR"                                       # Re-register with LaunchServices

    # B. Delete and re-create the Dock entry at the same position
    local count i found_index=-1
    count=$(/usr/libexec/PlistBuddy -c "Print persistent-apps" "$dock_plist" | grep -c "^    Dict")
    for (( i=0; i<count; i++ )); do
        local url
        url=$(/usr/libexec/PlistBuddy -c "Print persistent-apps:${i}:tile-data:file-data:_CFURLString" "$dock_plist")
        if [[ "$url" == "$app_url" ]]; then
            found_index=$i
            break
        fi
    done

    if [[ $found_index -ge 0 ]]; then
        /usr/libexec/PlistBuddy -c "Delete persistent-apps:${found_index}" "$dock_plist"
        /usr/libexec/PlistBuddy \
            -c "Add persistent-apps:${found_index} dict" \
            -c "Add persistent-apps:${found_index}:tile-type string file-tile" \
            -c "Add persistent-apps:${found_index}:tile-data dict" \
            -c "Add persistent-apps:${found_index}:tile-data:file-data dict" \
            -c "Add persistent-apps:${found_index}:tile-data:file-data:_CFURLString string ${app_url}" \
            -c "Add persistent-apps:${found_index}:tile-data:file-data:_CFURLStringType integer 15" \
            "$dock_plist"
    fi

    # C. Restart Dock (reads fresh plist, does fresh icon lookup)
    killall Dock
}
```

### Why each step matters

| Step | What it does | Why it's needed |
|---|---|---|
| `cp` to AppIcon.icns | Updates the file that `CFBundleIconFile` points to | Correctness for anything reading Info.plist |
| `NSWorkspace.setIcon` | Sets icon as extended attribute on the `.app` folder | This is what Finder/Dock/Spotlight **prefer** to read — same mechanism as pasting an icon in Get Info |
| `touch` | Updates modification timestamps | Signals "this bundle changed" |
| `lsregister -f` | Re-registers with LaunchServices | Updates the LS database entry |
| **PlistBuddy Delete + Add** | Destroys the cached bookmark blob | **This is the critical step** — forces the Dock to build a new bookmark with a fresh icon lookup |
| `killall Dock` | Restarts the Dock process | Reads the new plist with the fresh entry |

## Tested on

- macOS Sequoia (Darwin 25.1.0)
- Unsigned shell-script `.app` bundle
- Dock item pinned at position 0

## References

- [Granola: "So, you think it's easy to change an app icon?"](https://www.granola.ai/blog/so-you-think-its-easy-to-change-an-app-icon)
- [Tauri issue #2985: Cannot change app dock icon dynamically](https://github.com/tauri-apps/tauri/issues/2985)
- [Apple Developer Forums: macOS app icon not updating](https://developer.apple.com/forums/thread/676723)
- [GitHub Gist: Clear the icon cache on Mac](https://gist.github.com/ismyrnow/e92c6010cda9325b2d8811387a05f224)
