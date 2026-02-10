# Risk Register

## Active Risks

### R1: Script Load Order Sensitivity
- **Severity:** HIGH
- **Description:** The original code relies on synchronous script execution order for global variable availability. External `<script src>` tags maintain this order, but any reordering would break initialization.
- **Mitigation:** Script order in index.html exactly mirrors original file order. Load order is documented in RefactorJournal.md.
- **Status:** Mitigated

### R2: CSS Specificity Changes
- **Severity:** MEDIUM
- **Description:** Moving CSS from inline `<style>` to external files does not change specificity, but the order of CSS file loading must match the original style block order.
- **Mitigation:** CSS link order in index.html matches original style block order.
- **Status:** Mitigated

### R3: Monkey-Patch Dependencies
- **Severity:** HIGH
- **Description:** ~7,000 lines of patch code override prototype methods. These patches rely on specific execution timing and the existence of base classes.
- **Mitigation:** All patch scripts are loaded after their target classes, in the same relative order as the original.
- **Status:** Mitigated

### R4: Worker Inline String
- **Severity:** LOW
- **Description:** TileLogicEngine and WorldWorkerClient create Web Workers from inline strings using Blob URLs. This code is preserved as-is in the extracted files.
- **Mitigation:** No changes to worker creation logic.
- **Status:** No action needed

### R5: First-Load Performance
- **Severity:** LOW
- **Description:** External CSS/JS files require separate HTTP requests. On first load without cache, this could be slightly slower than the monolithic file.
- **Mitigation:** Files are served from same origin. Total payload size is unchanged. Subsequent loads benefit from caching.
- **Status:** Accepted (net positive with caching)

### R6: Global Namespace Pollution
- **Severity:** MEDIUM (existing, not introduced)
- **Description:** 30+ symbols on window. This is inherited from the original code.
- **Mitigation:** Future work: migrate to ES modules. Current refactoring preserves globals for compatibility.
- **Status:** Documented for future work

## Resolved Risks

### R7: File Rename
- **Severity:** LOW
- **Description:** Renaming `index (92).html` to `index.html` could break references.
- **Resolution:** No external references to the original filename exist in the codebase.
- **Status:** Resolved

## Future Work Risks

### R8: ES Module Migration
- **Severity:** MEDIUM
- **Description:** Converting to ES modules would eliminate global pollution but requires careful dependency management.
- **Recommendation:** Plan a phased migration starting with leaf modules (utilities, constants).

### R9: Patch Layer Consolidation
- **Severity:** HIGH
- **Description:** Merging 7,000 lines of monkey-patches into class definitions requires line-by-line equivalence verification.
- **Recommendation:** Implement with comprehensive behavioral testing.
