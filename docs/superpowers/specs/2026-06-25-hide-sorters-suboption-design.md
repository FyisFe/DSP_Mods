# Design: "Hide sorters too" sub-option for *Do not render factory entities*

Date: 2026-06-25
Mod: UXAssist
Status: Approved (design)

## Summary

Add a sub-checkbox under the existing **Do not render factory entities (except belts and
sorters)** feature. When the sub-option is enabled, sorters (inserters) are also hidden, so
that among factory entities only belts (and their cargo) remain visible.

User intent: 仅显示传送带不显示分拣器 ("only show belts, not sorters").

## Background: how the current feature works

The feature `FactoryPatch.DoNotRenderEntities` (a `PatchImpl`) hides buildings by patching the
per-model `Render` methods to return `false` (skip):

- `ObjectRenderer.Render` + `DynamicRenderer.Render` → skipped (hides generic buildings).
- `LabRenderer.Render` → skipped (hides labs).
- A postfix on `GPUInstancingManager.Render` keeps `renderEntity = true`.
- A transpiler on `RaycastLogic.GameTick` keeps inserters click-selectable while hidden.

What is **not** patched, and therefore still renders when the feature is on:

- **Belts + cargo** — rendered via `CargoTraffic.Draw(Camera)` and `CargoContainer.Draw(Camera)`
  (called from `FactoryModel.DrawInstancedBatches`), neither of which the feature touches.
- **Sorters** — rendered via `InserterRenderer.Render`. `InserterRenderer : ObjectRenderer`
  **overrides** `Render`, so the `ObjectRenderer.Render` patch does not apply to it.
- Vegetation (`VegeRenderer.Render`), veins, etc.

Hence the feature's name "(except belts and sorters)": both stay visible today.

## Goal

When the new sub-option is ON (and the parent feature is ON):

- **Hide sorters** (in addition to the buildings already hidden).
- Leave belts, cargo, vegetation, and everything else exactly as the parent feature leaves them.

End state among factory entities: only belts (+ their moving cargo) are visible.

## Approach

Add a single Harmony prefix to `InserterRenderer.Render`, living inside the existing
`DoNotRenderEntities` patch class and gated by a static flag:

```csharp
public static bool HideSorters;

[HarmonyPrefix]
[HarmonyPatch(typeof(InserterRenderer), nameof(InserterRenderer.Render))]
private static bool InserterRenderer_Render_Prefix()
{
    return !HideSorters; // true → run original (render sorters); false → skip (hide sorters)
}
```

Why this is correct and minimal:

- The `DoNotRenderEntities` patch class is only active while the parent feature is enabled, so
  this prefix has no effect when the parent is off.
- Sub-option OFF → prefix returns `true` → sorters render → identical to today's behavior.
- Sub-option ON → prefix returns `false` → sorters hidden. Belts/cargo/buildings untouched.
- Takes effect on the next rendered frame; no re-patch / refresh needed.
- The same `InserterRenderer.Render` also serves prebuild (ghost) inserter renderers, so ghost
  sorters are hidden too when the sub-option is on — consistent with hiding real sorters.

`InserterRenderer` is in the global namespace (like `ObjectRenderer`/`DynamicRenderer`/
`LabRenderer`, which `FactoryPatch.cs` already references unqualified), so no extra `using`.

### Alternatives considered (rejected)

- **Separate `PatchImpl` class for hiding sorters.** More boilerplate (own enable/disable
  lifecycle) for something that is logically a child of the parent feature and only meaningful
  while the parent is on.
- **Transpile `GPUInstancingManager.Render` to skip inserter renderer types.** More invasive and
  harder to read than a one-line prefix.

## Configuration

New entry in the `Factory` config section:

- Field: `FactoryPatch.DoNotRenderEntitiesHideSortersEnabled` (`ConfigEntry<bool>`, default `false`).
- Key: `"DoNotRenderEntitiesHideSorters"` (mirrors parent key `"DoNotRenderEntities"` +
  `"HideSorters"`, matching the `DragBuildPowerPoles` → `DragBuildPowerPolesAlternately`
  convention).
- Bound in `UXAssist.cs` next to `DoNotRenderEntitiesEnabled`.

Wiring in `FactoryPatch`:

- `Init()`: `DoNotRenderEntitiesHideSortersEnabled.SettingChanged += (_, _) =>
  DoNotRenderEntities.HideSorters = DoNotRenderEntitiesHideSortersEnabled.Value;`
- `Start()`: set initial `DoNotRenderEntities.HideSorters =
  DoNotRenderEntitiesHideSortersEnabled.Value;` alongside the existing
  `DoNotRenderEntities.Enable(...)` call.

## UI

In `UIConfigWindow.cs`, add an indented sub-checkbox immediately under the existing
"Do not render factory entities" checkbox (line ~399), mirroring the `DragBuildPowerPoles` →
`Alternately` layout:

- Parent checkbox at `x`, then `y += 27f`.
- Sub-checkbox at `x + 20f`, font size `13`, bound to `DoNotRenderEntitiesHideSortersEnabled`.
- `DoNotRenderEntitiesEnabled.SettingChanged` handler calls
  `hideSortersCheckBox.SetEnable(DoNotRenderEntitiesEnabled.Value)` so the sub-checkbox is
  greyed out when the parent is off; unsubscribed via `wnd.OnFree`.
- Advance `y += 36f` afterward before the next control, preserving current spacing.

Label translation registered via `I18N.Add` near the existing
`"Do not render factory entities"` entry (UIConfigWindow.cs line ~72):

```csharp
I18N.Add("Hide sorters too", "Hide sorters too", "同时隐藏分拣器");
```

## Documentation

- `README.md`: note the sub-option under the feature (English ~line 60, 中文 ~line 226).
- `CHANGELOG.md`: add an entry (English + 中文) for the new sub-option.

## Out of scope

- **Raycast / click-through.** The existing transpiler keeps inserters click-selectable even
  when not rendered; hidden sorters remain clickable. Left unchanged (harmless). Could be a
  follow-up if click-through of hidden sorters is desired.
- **Vegetation and cargo.** Unchanged. Belts keep showing their moving cargo (per user choice).

## Acceptance criteria

1. Parent OFF: no behavior change regardless of the sub-option value; sorters render normally.
2. Parent ON, sub-option OFF: identical to current behavior (belts + sorters visible, buildings
   hidden).
3. Parent ON, sub-option ON: sorters hidden; belts + cargo still visible; buildings still hidden.
4. Toggling the sub-option at runtime updates the view on the next frame without reloading.
5. The sub-checkbox is greyed out when the parent feature is off.
6. Label shows correctly in English and Chinese.
