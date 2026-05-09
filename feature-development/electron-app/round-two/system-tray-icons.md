# System Tray Icons — Design Plan

## Current: Programmatic Icons (Interim)

The initial implementation generates tray icons at runtime using Electron's `nativeImage` canvas API. These are simple colored circles indicating health state:

| State          | Color  | Meaning                                |
| -------------- | ------ | -------------------------------------- |
| Healthy        | Green  | All metrics within thresholds          |
| Warning        | Yellow | Warning-level issues detected          |
| Critical       | Red    | Critical issues or complexity exceeded |
| Paused/No repo | Gray   | Monitoring inactive                    |

### Platform behavior

- **macOS**: 22x22pt monochrome template image (adapts to light/dark menu bar automatically via `setTemplateImage(true)`)
- **Windows**: 16x16px full-color icon
- **Linux**: 22x22px full-color PNG

## Future: Designed SVG/PNG Assets

The programmatic icons should be replaced with properly designed static assets:

### Requirements

- Vipr-branded icon silhouette (not just a colored dot)
- State indicated by color accent or badge overlay, not entirely different icons
- macOS template images must be monochrome with alpha channel (system handles light/dark)
- Windows `.ico` files with multiple resolutions (16x16, 32x32, 48x48, 256x256)
- Linux PNGs at 16x16, 22x22, 32x32, 48x48

### Deliverables

```
clients/desktop/build/tray/
  macOS/
    tray-template.png       # 22x22 monochrome
    tray-template@2x.png    # 44x44 monochrome
  windows/
    tray-healthy.ico
    tray-warning.ico
    tray-critical.ico
    tray-gray.ico
  linux/
    tray-healthy.png        # 22x22
    tray-warning.png
    tray-critical.png
    tray-gray.png
```

### Migration path

When designed assets are ready:

1. Place files in `clients/desktop/build/tray/`
2. Update `TrayManager.getIconForState()` to load from file path instead of generating programmatically
3. Use `nativeImage.createFromPath()` with platform-specific paths
4. macOS: Call `image.setTemplateImage(true)` on the loaded image
5. Remove programmatic icon generation code
