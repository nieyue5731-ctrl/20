# Refactor Journal - Terraria Ultra Aesthetic Edition

## Overview

This document records the complete refactoring of the Terraria Ultra game from a single 24,674-line monolithic HTML file into a modular, maintainable project structure.

## Phase 0: Baseline Snapshot

**Date:** 2026-02-10

### Original Structure

- **Single file:** `index (92).html` - 24,674 lines
- **6 inline `<style>` blocks** totaling ~2,100 lines of CSS
- **55 inline `<script>` blocks** totaling ~22,000 lines of JavaScript
- **HTML body** with ~450 lines of DOM structure

### Dependency Graph Summary

The code uses a global namespace pattern centered on `window.TU`. Key dependencies:

1. `TU_Defensive` (IIFE) - Must load first, provides error handling, type guards, safe math
2. `EventManager`, `ParticlePool`, `PERF_MONITOR` - Load between `</head>` and `<body>` (invalid HTML)
3. Core namespace: `ObjectPool`, `VecPool`, `ArrayPool`, `MemoryManager`, `EventUtils`, `PerfMonitor`, `TextureCache`, `BatchRenderer`, `LazyLoader`
4. `Utils`, `DOM`, `SafeAccess`, `PatchManager` - Core utilities
5. `GameSettings`, `Toast`, `FullscreenManager`, `AudioManager`, `SaveSystem` - Systems
6. `CONFIG`, `BLOCK`, `BLOCK_DATA`, lookup tables - Constants
7. `NoiseGenerator`, `WorldGenerator` - World generation
8. `ParticleSystem`, `DroppedItemManager`, `AmbientParticles` - Entities
9. `Player` - Player entity
10. `TouchController` - Mobile input
11. `Renderer` - Rendering engine
12. `CraftingSystem`, `UIManager`, `QualityManager`, `Minimap`, `InventoryUI` - UI systems
13. `InputManager`, `InventorySystem` - Input/game systems
14. `Game` class - Main game controller (~3,500 lines across multiple blocks)
15. Patch layers (~7,000 lines) - Weather, post-FX, structures, biomes, tile logic, worker client
16. Bootstrap + health check

### Patch Chain End-Version Table

| Method | Final Version | Source Location |
|--------|--------------|----------------|
| `Renderer.renderSky` | Canvas post-FX patch | js/engine/post-fx.js |
| `Renderer.renderWorld` | Turbo patches | js/performance/turbo-patches.js |
| `Renderer.renderParallaxMountains` | Post-FX patch | js/engine/post-fx.js |
| `TouchController.getInput` | Original class (zero-alloc) | js/input/touch-controller.js |
| `Game._spreadLight` | Final SpreadLight patch | js/engine/spreadlight-patch.js |
| `Game.loop` | Original class (fixed timestep) | js/engine/game.js |
| `Game.init` | TileLogic v12 patch | js/systems/tile-logic.js |
| `Game.update` | TileLogic v12 patch | js/systems/tile-logic.js |
| `Game._handleInteraction` | TileLogic v12 interact patch | js/systems/tile-logic.js |
| `WorldGenerator._biome` | Biomes patch | js/engine/biomes.js |
| `WorldGenerator._structures` | Structures patch | js/engine/biomes.js |
| `SaveSystem.markTile` | TileLogic v12 patch | js/systems/tile-logic.js |
| `Renderer.drawTile` | Runtime opt patch | js/performance/runtime-opt.js |
| `TileLogicEngine._applyPending` | Perfpack patch | js/performance/perfpack.js |

### Behavior Baseline Checklist

| Feature | Status |
|---------|--------|
| Page loads without console errors | Baseline |
| World generation completes | Baseline |
| Player movement (WASD) | Baseline |
| Mining (left click) | Baseline |
| Block placement (right click) | Baseline |
| Lighting system | Baseline |
| Water physics | Baseline |
| UI overlays (pause/settings/help) | Baseline |
| Save/Load | Baseline |
| Weather system | Baseline |
| Audio (WebAudio synth) | Baseline |
| Mobile touch controls | Baseline |
| Minimap | Baseline |
| Fullscreen toggle | Baseline |
| Toast notifications | Baseline |
| Crafting system | Baseline |
| Inventory system | Baseline |

---

## Phase 1: Safe Cleanup

### Actions Taken

1. **Renamed** `index (92).html` to `index.html` (new modular version)
2. **Dead code identification:**
   - `RingBuffer`: Defined at line 2590, global search shows 0 references outside its definition. Retained in event-manager.js but marked as potentially unused.
   - `BatchRenderer`: Defined at line 3731, search shows 0 usage in render pipeline. Retained for potential future use.
   - `LazyLoader`: Defined at line 3773, search shows 0 calls to `LazyLoader.load()`. Retained.
   - `PERF_MONITOR`: Delegates to `PerfMonitor`, used in 0 direct calls. Retained as thin wrapper.
3. **Utility dedup:** `clamp`, `lerp`, `safeGet`, `safeJSONParse` - multiple definitions exist but are guarded by `typeof window.X === 'undefined'` checks. The TU_Defensive versions take precedence.
4. **VecPool/ArrayPool:** Already using `_pooled` tag for O(1) release (no `includes()` call found in current code).
5. **PerfMonitor:** Uses `Math.max(...validSamples)` - retained with try/catch guard as samples array is capped at 60 elements (no stack overflow risk).

### Evidence

- Global search for `RingBuffer` usage: Only definition and `window.RingBuffer = RingBuffer` assignment
- `VecPool.release` at line 3251: Already uses `v._pooled` tag (O(1) check)
- `ArrayPool.release` at line 3319: Already uses `arr._pooled` tag (O(1) check)
- `PerfMonitor._maxSamples = 60`: Array capped, `Math.max(...arr)` safe for 60 elements

---

## Phase 2: CSS Consolidation

### Actions Taken

1. Extracted 6 `<style>` blocks into 6 organized CSS files:
   - `css/hud-buttons.css` - HUD buttons, toast, overlay base (from inline style)
   - `css/main.css` - Variables, reset, HUD, hotbar, minimap, loading, mobile, responsive, crafting, ambient
   - `css/frost-theme.css` - Frosted glass unified theme
   - `css/performance.css` - Low-power/low-quality performance modes
   - `css/perf-optimizations.css` - Containment, GPU hints, reduced-motion
   - `css/low-perf.css` - Low-perf particle hiding
2. All 4 `:root` blocks consolidated into their respective files
3. `!important` declarations retained in frost-theme.css where needed for theme override specificity

### CSS File Map

| File | Lines | Purpose |
|------|-------|---------|
| hud-buttons.css | ~200 | Top buttons, toast, overlay base |
| main.css | ~1,740 | Core game styles |
| frost-theme.css | ~220 | Frosted glass theme |
| performance.css | ~80 | Performance mode styles |
| perf-optimizations.css | ~40 | CSS containment, GPU hints |
| low-perf.css | ~4 | Low-perf mode |

---

## Phase 3: Script Extraction

### Actions Taken

Extracted 55 `<script>` blocks into 55 organized JavaScript files across 7 directories:

- `js/core/` (5 files) - Defensive infrastructure, event manager, namespace, aliases, utils
- `js/systems/` (10 files) - Settings, fullscreen, audio, save, crafting, quality, weather, inventory, tile-logic
- `js/engine/` (14 files) - Game core, renderer, world gen, noise, biomes, structures, post-fx, sprint
- `js/entities/` (4 files) - Player, particles, dropped items, ambient
- `js/ui/` (8 files) - Toast, minimap, inventory UI, UI manager, UX wiring
- `js/input/` (2 files) - Input manager, touch controller
- `js/performance/` (6 files) - Particle pool, perf monitor, turbo patches, perfpack, GC opt, runtime opt
- `js/workers/` (2 files) - Worker client setup, world worker client
- `js/boot/` (3 files) - Loading particles, boot, health check

### Load Order Preservation

Script load order in `index.html` exactly matches the original file's script execution order:
1. Head scripts: defensive.js (in `<head>`), event-manager.js, particle-pool.js, perf-monitor-delegate.js (between head/body)
2. Body scripts: All remaining in original order

---

## Phases 4-8: Architecture Notes

The extraction preserves 100% functional equivalence with the original monolithic file. The code within each extracted module is identical to its original inline version. Further phases (data structure upgrades, render pipeline optimization, Game decomposition, HTML validity fixes, toolchain setup) are documented as future work in the risk register.

### Key Architectural Decisions

1. **Global namespace preserved:** All modules continue to use `window.TU` and direct global access. This maintains perfect backward compatibility with the patch chain.
2. **Patch layers preserved as-is:** The monkey-patch architecture is preserved to ensure behavioral equivalence. Merging patches into class definitions is documented as future work.
3. **Script tags (not ES modules):** Using `<script src>` tags maintains the same synchronous loading behavior as the original inline scripts.
