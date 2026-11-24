---
name: isometric-diamond-grid
description: >
  Standardizes Project-Rouge's 2:1 isometric diamond-shaped tile grid math, coordinate
  transforms (gridToScreen / screenToGrid), painter's algorithm zIndex layering, PixiJS v8
  camera/input handling, crisp pixel rendering, 8-way A* pathfinding without diagonal corner
  cutting, and symmetric shadowcasting FOV. Load before implementing or modifying map tiles,
  coordinates, camera, picking, pathfinding, or fog of war.
---

# Isometric Diamond Grid (Project-Rouge)

Authoritative spatial & rendering contract for Project-Rouge's isometric diamond-shaped tile map in **PixiJS v8**. Always read [.agents/skills/pixijs/SKILL.md](../pixijs/SKILL.md) alongside this skill before writing PixiJS rendering code, and [.agents/skills/roguelike/references/run-seeds-fov.md](../roguelike/references/run-seeds-fov.md) (§3) for FOV algorithms.

---

## 1. Grid Convention & Tile Geometry

- **Projection:** Standard **2:1 isometric diamond**.
  - `TILE_WIDTH = 64`
  - `TILE_HEIGHT = 32`
  - `HALF_W = 32` (`TILE_WIDTH / 2`)
  - `HALF_H = 16` (`TILE_HEIGHT / 2`)
  - `ELEVATION_PX = 16` (vertical pixel offset per elevation step `z = 1`)
- **Simulation Coordinates:** Integer grid coordinates `(gx, gy, z)` are the single source of truth for gameplay logic, pathfinding, combat reach, and FOV. Never store screen pixels as simulation state.
- **Axes & Origin:**
  - `(0, 0)` is the **top (North)** corner of the diamond grid.
  - `+gx` advances **down-right (Southeast)**.
  - `+gy` advances **down-left (Southwest)**.
  - `(gx + gy)` increases straight **down (South)** toward the camera.
- **Anchor Point:** `(sx, sy)` MUST represent the **exact center of the tile's top diamond face**. Set sprite/container anchors cleanly (e.g., `(0.5, 0.5)` for flat floor diamonds, or bottom-center `(0.5, 1.0)` aligned to diamond center `+ offset` for tall walls/actors) so coordinate math stays uniform.

---

## 2. Exact Coordinate Transforms

### Grid to World/Screen (`gridToScreen`)

```ts
export const TILE_WIDTH = 64;
export const TILE_HEIGHT = 32;
export const HALF_W = TILE_WIDTH / 2;   // 32
export const HALF_H = TILE_HEIGHT / 2;  // 16
export const ELEVATION_PX = 16;

export interface GridCoord {
  gx: number;
  gy: number;
  z?: number;
}

export interface ScreenPoint {
  sx: number;
  sy: number;
}

export function gridToScreen(gx: number, gy: number, z = 0): ScreenPoint {
  return {
    sx: (gx - gy) * HALF_W,
    sy: (gx + gy) * HALF_H - z * ELEVATION_PX,
  };
}
```

### World/Screen to Grid (`screenToGrid`)

Because `(sx, sy)` is anchored at the diamond center, a tile's screen-space diamond maps to the unit square `[gx - 0.5, gx + 0.5) × [gy - 0.5, gy + 0.5)` in fractional grid space:

```ts
export function screenToGrid(worldX: number, worldY: number): {
  gx: number;
  gy: number;
  fx: number;
  fy: number;
} {
  const fx = (worldX / HALF_W + worldY / HALF_H) / 2;
  const fy = (worldY / HALF_H - worldX / HALF_W) / 2;
  return {
    gx: Math.round(fx),
    gy: Math.round(fy),
    fx,
    fy,
  };
}
```

- **Flat maps (`z = 0`):** `Math.round(fx)` and `Math.round(fy)` perform **exact diamond hit-testing** with no floating-point branch jitter.
- **Elevated maps (`z > 0`):** Iterate candidate elevations or visible tiles in **reverse painter's order** (highest `(gx + gy)` and highest `z` first, front-to-back). For a given tile elevation `z`, evaluate `screenToGrid(worldX, worldY + z * ELEVATION_PX)` and accept the first tile whose grid coordinate matches `(gx, gy)` at elevation `z` (and optionally test its front vertical skirt polygon if side-clicking raised blocks is enabled).

---

## 3. Painter's Algorithm & PixiJS v8 Layering

Organize the world `Container` into **5 explicit display layers** (in bottom-to-top render order):

1. `terrainLayer` — base floor diamonds, chasms, water, stairs.
2. `overlayLayer` — movement range highlights, A* path preview, AoE templates, attack/cover indicators, hover cursor diamond.
3. `entityLayer` — walls, pillars, doors, chests, props, actors/tokens. **Enable `entityLayer.sortableChildren = true`.**
4. `vfxLayer` — floating damage numbers, spell projectiles, hit flashes, telegraph markers.
5. `fogLayer` — explored-dimmed shroud and unseen shadow tiles.

### Deterministic `zIndex` Formula

Compute depth deterministically inside `entityLayer` (and optionally `terrainLayer` when tiles overlap):

```ts
export const LAYER_BIAS = {
  FLOOR_PROP: 0,
  ACTOR: 2,
  WALL_PILLAR: 4,
  OVERHEAD: 6,
} as const;

export function computeIsoZIndex(
  gx: number,
  gy: number,
  z = 0,
  layerBias: number = LAYER_BIAS.ACTOR
): number {
  return Math.round((gx + gy) * 100) + z * 10 + layerBias;
}
```

- **Tweened Movement:** When an actor walks smoothly from `(gx0, gy0)` to `(gx1, gy1)`, update its `zIndex` every frame using interpolated fractional coordinates `(interpX + interpY) * 100 + z * 10 + LAYER_BIAS.ACTOR` so actors cleanly pass behind and in front of walls mid-step.

---

## 4. PixiJS v8 Input, Camera & Pixel Crispness

- **Single Stage/Viewport Pointer Handler:** Never attach individual `pointertap`/`pointermove` listeners to hundreds of tile sprites. Attach one pointer listener on the stage/viewport container and transform pointer coordinates into world space using `worldContainer.toLocal(event.global)`:
  ```ts
  const local = worldContainer.toLocal(event.global);
  const { gx, gy } = screenToGrid(local.x, local.y);
  ```
- **Camera Pan & Cursor-Centered Zoom:**
  - Clamp camera zoom to `[0.5, 2.5]` (e.g., `0.5` min overview, `2.5` max tactical close-up).
  - Zoom around the cursor position by recording `before = worldContainer.toLocal(event.global)`, updating `worldContainer.scale`, reading `after = worldContainer.toLocal(event.global)`, and shifting `worldContainer.position` by `(after - before) * scale`.
- **React HUD Overlay:** Place the React UI overlay above the PixiJS `<canvas>` with `pointer-events: none` on the root HUD wrapper and `pointer-events: auto` only on interactive panels/buttons so map clicks pass through unobstructed.
- **Crisp Pixel Mode:** When rendering pixel-art tiles or sprites, set texture `scaleMode = 'nearest'` and `roundPixels = true` on the PixiJS renderer/containers; keep sprite dimensions aligned to integer pixels to prevent sub-pixel shimmer during pan/zoom.

---

## 5. Pathfinding & Symmetric Field of View

### 8-Way A* Pathfinding (D&D 5.5E Alignment)

- **Scale:** `1 tile = 5 ft`.
- **Movement Cost:**
  - Normal tile = `1` (`5 ft`), difficult terrain = `2` (`10 ft`).
  - Use Chebyshev distance `Math.max(Math.abs(dx), Math.abs(dy))` as the admissible heuristic (matching D&D 5e/5.5e grid diagonal movement where diagonals cost 5 ft; use `1.5` alternating or weighted cost only if explicitly configured).
- **No Diagonal Corner-Cutting:** A diagonal move from `(gx, gy)` to `(gx + dx, gy + dy)` (where `dx !== 0 && dy !== 0`) is **forbidden** if **either** orthogonal neighbor `(gx + dx, gy)` or `(gx, gy + dy)` blocks movement (wall, closed door, or impassable obstacle).

### Symmetric Shadowcasting FOV

- Follow **Albert Ford's Symmetric Shadowcasting** as documented in [.agents/skills/roguelike/references/run-seeds-fov.md](../roguelike/references/run-seeds-fov.md) (§3):
  - **Floor tiles** are visible only when their **center** lies inside the active slope range.
  - **Wall tiles** are visible if **any part** of the tile lies inside the slope range.
  - Use exact rational/integer slope fractions (avoid raw floating-point drift on edges).
- **Distance Metrics:**
  - Use **Chebyshev distance** `Math.max(Math.abs(dx), Math.abs(dy))` for D&D melee reach, movement range, and square AoE bounds.
  - Use **Euclidean circular check** `dx * dx + dy * dy <= r * r` for round light sources and FOV radius (`6–9` tiles default).
- **Visibility States & Caching:** Maintain per-tile states `unseen | explored | visible`. Recompute FOV **only** when a viewer moves or a door/wall changes state — never recompute every render frame.

---

## 6. Verification Checklist (Required Unit Tests)

Before considering grid/FOV changes complete, run unit tests verifying:
1. **Round-trip Grid Math:** `screenToGrid(gridToScreen(gx, gy))` returns exact `(gx, gy)` for every cell in a `30 × 30` grid, plus sub-pixel offsets inside all 4 diamond quadrants (`±0.2` tile offsets).
2. **FOV Symmetry:** On randomly generated rooms/corridors, for every pair of floor tiles `(A, B)`, `A` sees `B` if and only if `B` sees `A`.
3. **No Diagonal Corner-Cutting:** A* refuses diagonal steps adjacent to solid wall corners and routes orthogonally around the corner instead.
4. **Depth Sort Monotonicity:** For any adjacent tiles `(gx, gy)` vs `(gx + 1, gy)` or `(gx, gy + 1)` at equal `z`, `computeIsoZIndex` strictly increases toward the camera.
