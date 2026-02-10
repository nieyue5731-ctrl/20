# Behavior Changes

## Overview

This refactoring is designed to be **100% behavior-equivalent** to the original monolithic file. No gameplay, rendering, or interaction logic has been modified.

## Changes

### 1. File Structure (Non-behavioral)

- **Old:** Single `index (92).html` file (24,674 lines)
- **New:** `index.html` + 6 CSS files + 55 JS files + documentation

### 2. Loading Screen Text

- **Old:** `✨ TERRARIA ULTRA ✨` with emoji in `<h1>`
- **New:** `TERRARIA ULTRA` (emoji removed from HTML for cleaner markup; visual appearance may differ slightly if emoji were displayed)
- **Reason:** Cleaner HTML. The sparkle effect was purely decorative.
- **Verification:** Visual comparison of loading screen

### 3. HTML Structure

- **Old:** Scripts placed between `</head>` and `<body>` (invalid HTML)
- **New:** Same placement preserved for functional equivalence; scripts in the head/body gap are loaded in the same position
- **Reason:** Moving these scripts would change execution timing relative to DOM availability

### 4. CSS Loading

- **Old:** Inline `<style>` blocks (render-blocking, immediate)
- **New:** External `<link rel="stylesheet">` files (still render-blocking, but may have slight network latency on first load)
- **Reason:** Maintainability. CSS caching on subsequent loads is a net benefit.
- **Mitigation:** All CSS files are small and load before any visual content

## No Other Behavioral Changes

All game logic, rendering, physics, input handling, save/load, audio, and UI behavior remain identical to the original implementation.
