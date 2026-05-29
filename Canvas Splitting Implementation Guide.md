## Context

The profiler identified `UGUI.Rendering.UpdateBatches` as the most expensive per-frame system in the Hub World, consistently costing more than the entire `Camera.Render` pass. The root cause is that static chrome (panel backgrounds, borders) and dynamic content (TMP text, animated elements) live on the same `Canvas`, causing Unity to rebuild the entire canvas geometry batch whenever any child is dirtied.

The fix is to split each scene's single root canvas into two child canvases — one that never changes, one that can dirty freely. Unity rebuilds each child canvas independently.

---

## Target Structure (Minigame Scenes)

Every minigame scene's `Minigame UI` should match this layout after splitting:

```
Minigame UI                        Canvas + CanvasScaler (1024×576) + GraphicRaycaster
├── Static Canvas                  Canvas only — no GraphicRaycaster
│   ├── Top Panel BG               Image (dark semi-transparent) — no children
│   └── Bottom Panel BG            Image (dark semi-transparent) — no children
└── Dynamic Canvas                 Canvas + GraphicRaycaster
    ├── Top Bar
    │   ├── SONA Container
    │   └── Level Title Text
    ├── [Minigame Controls Panel]
    ├── Level Complete UI
    ├── Concepts Learned Panel
    ├── Bottom Panel
    │   ├── Cancel Button
    │   └── Puzzle Solved
    └── Fade In/Out Image
```

---

## Target Structure (Hub World - Ghost Layout Pattern)

The Hub World uses a high-resolution background map and a Horizontal Layout Group for the SONA dialogue. To maintain layout integrity while optimizing rendering, we use the **Ghost Layout Pattern**.

```
Hub UI (Canvas, CanvasScaler, GraphicRaycaster)
├── Static Canvas (Canvas only)
│   ├── Hub Map Background Image          # Raycast Target: OFF
│   └── Top Bar BG Image                  # Raycast Target: OFF
│       └── SONA Visuals Container       # Position/Size Constrained to SONA Container
│           ├── SONA Avatar Image
│           └── Dialogue Bubble BG Image
└── Dynamic Canvas (Canvas, GraphicRaycaster)
    ├── Top Bar (Layout Container)
    │   └── SONA Container                # Horizontal Layout Group
    │       ├── Avatar Spacer             # Empty/Invisible - Raycast Target: OFF
    │       └── Dialogue Slot             # Empty/Invisible - Raycast Target: OFF
    │           └── SONA Dialogue Text    # TMP_Text (Dynamic)
    ├── Progress Text (TMP_Text)
    └── [Level/Minigame Buttons]
```

---

## Steps — Apply to Minigame Scenes

### 1. Create Child Canvases
Create `Static Canvas` (Canvas component, no Raycaster) and `Dynamic Canvas` (Canvas + GraphicRaycaster). `Static Canvas` must be higher in the hierarchy to render behind.

### 2. Extract Backgrounds
1. Select a panel (e.g., `Top Panel`).
2. Create an empty child named `BG`.
3. Move the `Image` component from the parent to the `BG`.
4. Set `BG` to Stretch/Stretch.
5. Reparent `BG` to the `Static Canvas`.

---

## Steps — Hub World (Ghost Layout)

### 1. Invisible Layout Spacers
1. In the `Dynamic Canvas`, keep the `SONA Container` and its Layout Group.
2. Remove/Disable the `Image` components from the `Avatar` and `Dialogue Panel` slots.
3. Ensure these objects (and the `TMP_Text`) have **Raycast Target: OFF** unless they specifically need interaction.

### 2. Mirrored Visuals
1. In the `Static Canvas`, nest `SONA Visuals` inside the `Top Bar BG Image`.
2. Add a **Position Constraint** to `SONA Visuals`, targeting the `SONA Container` in the `Dynamic Canvas`.
3. Click **"Zero"** and **"Activate"**.
4. Place the static images inside this constrained container.

---

## Verification

After completing the split, enter Play Mode and check the Profiler:
```
Baseline:   UGUI.Rendering.UpdateBatches  0.056 – 0.137 ms
Target:     UGUI.Rendering.UpdateBatches  < 0.060 ms (flat line)
```

> **Key Rule**: All images in the `Static Canvas` must have **Raycast Target unchecked** to prevent blocking interactions in the `Dynamic Canvas`.
