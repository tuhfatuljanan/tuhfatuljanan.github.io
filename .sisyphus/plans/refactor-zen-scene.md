# Refactor zen scene implementation

## Goal
Refactor `assets/js/zen_scene.js` to implement the user-requested optimizations without changing the overall scene concept:

1. Replace per-frame pine mesh scanning with cached references.
2. Add lifecycle cleanup for resize/RAF/WebGL resources.
3. Reuse geometries and materials where practical.
4. Extract scattered magic numbers into configuration objects.
5. Tighten cloud and mountain transparency behavior.

## Files in scope
- `assets/js/zen_scene.js` — primary implementation target.
- `zen.html` — only if a tiny complementary CSS cleanup is needed; otherwise avoid touching it.

## Current code hotspots
- Pine animation currently scans `treeGroup.children` every frame in `animate()` (`assets/js/zen_scene.js`, around lines 364-374).
- Scene starts a permanent RAF loop and a resize listener with no explicit teardown (`assets/js/zen_scene.js`, around lines 390-403).
- Repeated geometry creation exists in pine layers, grass, rocks, mountains, and stars (`assets/js/zen_scene.js`, around lines 89-345).
- Magic numbers are scattered across scene setup, object creation, and animation sections.
- Mountains use transparent material and clouds use transparent layered materials, which can cause avoidable sorting/depth artifacts.

## Planned changes

### 1. Introduce config objects
Create top-level config objects near the start of `assets/js/zen_scene.js`, for example:
- `SCENE_CONFIG` — viewport, fog, camera, light, animation constants.
- `TREE_CONFIG` — trunk dimensions, pine layer data, branch scaling and offsets.
- `GROUND_CONFIG` — stone positions, grass counts/ranges, mountain shapes.
- `SKY_CONFIG` — star counts, cloud counts, cloud drift values.
- `THEME_CONFIG` — dark/day colors and per-theme material parameters.

This keeps current values but centralizes them for maintainability.

### 2. Cache pine mesh references during creation
Refactor pine creation so each created cone mesh is registered at creation time:
- Build a `pineLayerMeshes` array (for example `Array.from({ length: pineLayers.length }, () => [])`).
- Main layer cone and its small child cones should be pushed into the correct layer bucket when created.
- `animate()` should iterate `pineLayerMeshes[index]` directly and update rotations, instead of scanning `treeGroup.children` and checking geometry type / y proximity every frame.

This is the primary animation optimization.

### 3. Reuse geometries/materials pragmatically
Implement low-risk reuse rather than over-engineering:
- Use shared base geometries where scale is sufficient:
  - unit cone geometry for pine pieces and grass,
  - unit sphere geometry for stars/cloud puffs,
  - shared dodecahedron geometry for rocks,
  - shared cone/cylinder geometries for mountains where scaling fits.
- Use mesh scaling to vary size instead of creating a fresh geometry per object wherever safe.
- Reuse shared materials for stars and existing repeated mesh families.

Do **not** force `InstancedMesh` in this refactor unless a specific section becomes simpler with it; public guidance suggests shared geometry/material is sufficient ROI for this scene size.

### 4. Add lifecycle cleanup
Add explicit cleanup primitives:
- Track `animationFrameId` returned by `requestAnimationFrame`.
- Track a `disposed` flag to stop the loop safely.
- Add a `disposeScene()` / `cleanup()` function that:
  - removes the resize listener,
  - cancels the pending RAF,
  - disposes shared geometries/materials,
  - disposes renderer,
  - removes the renderer DOM element if still mounted.
- Attach cleanup to page lifecycle (`pagehide`, and optionally `beforeunload`).

This addresses current missing teardown while respecting shared resource disposal rules.

### 5. Tighten transparency behavior
Apply lower-risk transparency improvements:
- Mountains: remove `transparent: true` entirely and render them as opaque fog-tinted geometry unless code inspection during edit shows a strong reason to preserve transparency.
- Clouds: keep transparency, but set `depthWrite = false`; keep `depthTest = true` and retain layered materials.

This follows common Three.js guidance for soft clouds while reducing sorting artifacts and avoiding unnecessary mountain transparency.

### 6. Keep scene behavior stable
Preserve current visible composition as much as possible:
- same tree silhouette and object placement,
- same day/night content split,
- same camera orbit idea,
- same cloud drift feel.

The refactor goal is internal improvement + small transparency cleanup, not a visual redesign.

## Risks and mitigations
- **Risk:** scaling shared geometries changes mesh proportions subtly.
  - **Mitigation:** use unit geometries with explicit `mesh.scale` values matched to current dimensions.
- **Risk:** cleanup disposes shared resources too aggressively.
  - **Mitigation:** only dispose top-level shared geometry/material registries once, inside the single scene cleanup.
- **Risk:** transparency change alters mountain look too much.
  - **Mitigation:** keep mountain color/fog relationship close to current appearance even if transparency is removed.

## Verification
1. Run `node --check assets/js/zen_scene.js`.
2. Attempt `bundle exec jekyll build` from repo root; if unavailable, record the environment limitation.
3. Confirm by code inspection that:
   - no per-frame pine child scan remains,
   - cleanup path exists for listener + RAF + renderer/resources,
   - repeated geometry creation is reduced via shared geometry setup,
   - config objects exist and are used,
   - mountain/cloud transparency settings were tightened as planned.
