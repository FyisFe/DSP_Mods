# "Hide sorters too" Sub-option — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a sub-checkbox under *Do not render factory entities (except belts and sorters)* that, when enabled, also hides sorters so only belts (and their cargo) remain visible.

**Architecture:** A single Harmony prefix on `InserterRenderer.Render`, gated by a static `HideSorters` flag, lives inside the existing `FactoryPatch.DoNotRenderEntities` patch class (only active while the parent feature is on). A new `ConfigEntry<bool>` drives the flag via `SettingChanged`. A new indented UI sub-checkbox is wired to grey out when the parent is off.

**Tech Stack:** C# (.NET Framework 4.7.2), BepInEx plugin, HarmonyX, CommonAPI. Dyson Sphere Program mod (`UXAssist`).

## Global Constraints

- Target/runtime: .NET Framework `net472`, BepInEx plugin GUID `org.soardev.uxassist`.
- **No automated test harness exists.** Each code task's gate is a successful `dotnet build` plus the described manual in-game check. Do not invent a unit-test framework.
- Decompiled game source for reference lives at `C:\Users\Yi\Applications\Games\dsp\code\dsp-01-22-26\` (read-only; not part of the build).
- Config section for all entries below: `"Factory"`.
- New config key: `"DoNotRenderEntitiesHideSorters"`; field: `FactoryPatch.DoNotRenderEntitiesHideSortersEnabled`; default `false`.
- UI label key/string: `"Hide sorters too"` → EN `"Hide sorters too"`, 中文 `"同时隐藏分拣器"`.
- `InserterRenderer`, `ObjectRenderer`, `DynamicRenderer` are global-namespace game types already referenced unqualified in `FactoryPatch.cs`; no new `using` needed.
- Build command (run from repo root `C:\Users\Yi\Applications\Games\dsp\code\DSP_Mods`):
  `dotnet build UXAssist/UXAssist.csproj -c Release`

---

### Task 1: Config entry + sorter-hiding patch logic

**Files:**
- Modify: `UXAssist/Patches/FactoryPatch.cs` (field decl ~line 32; `Init()` ~line 125; `Start()` ~line 157; `DoNotRenderEntities` class ~line 1437)
- Modify: `UXAssist/UXAssist.cs` (config bind ~line 133)

**Interfaces:**
- Produces: `FactoryPatch.DoNotRenderEntitiesHideSortersEnabled` (`ConfigEntry<bool>`); `FactoryPatch.DoNotRenderEntities.HideSorters` (`static bool`).
- Consumes: existing `FactoryPatch.DoNotRenderEntities` `PatchImpl`, existing `DoNotRenderEntitiesEnabled` config entry.

- [ ] **Step 1: Declare the config field**

In `UXAssist/Patches/FactoryPatch.cs`, immediately after the existing line:
```csharp
    public static ConfigEntry<bool> DoNotRenderEntitiesEnabled;
```
add:
```csharp
    public static ConfigEntry<bool> DoNotRenderEntitiesHideSortersEnabled;
```

- [ ] **Step 2: Bind the config entry**

In `UXAssist/UXAssist.cs`, immediately after the existing bind:
```csharp
        FactoryPatch.DoNotRenderEntitiesEnabled = Config.Bind("Factory", "DoNotRenderEntities", false,
            "Do not render factory entities");
```
add:
```csharp
        FactoryPatch.DoNotRenderEntitiesHideSortersEnabled = Config.Bind("Factory", "DoNotRenderEntitiesHideSorters", false,
            "Hide sorters too when not rendering factory entities (leaves only belts visible)");
```

- [ ] **Step 3: Wire the SettingChanged handler**

In `UXAssist/Patches/FactoryPatch.cs` `Init()`, immediately after the existing line:
```csharp
        DoNotRenderEntitiesEnabled.SettingChanged += (_, _) => DoNotRenderEntities.Enable(DoNotRenderEntitiesEnabled.Value);
```
add:
```csharp
        DoNotRenderEntitiesHideSortersEnabled.SettingChanged += (_, _) => DoNotRenderEntities.HideSorters = DoNotRenderEntitiesHideSortersEnabled.Value;
```

- [ ] **Step 4: Set the initial flag value in `Start()`**

In `UXAssist/Patches/FactoryPatch.cs` `Start()`, immediately after the existing line:
```csharp
        DoNotRenderEntities.Enable(DoNotRenderEntitiesEnabled.Value);
```
add:
```csharp
        DoNotRenderEntities.HideSorters = DoNotRenderEntitiesHideSortersEnabled.Value;
```

- [ ] **Step 5: Add the static flag and the `InserterRenderer.Render` prefix**

In `UXAssist/Patches/FactoryPatch.cs`, find the class header and first prefix:
```csharp
    private class DoNotRenderEntities : PatchImpl<DoNotRenderEntities>
    {
        [HarmonyPrefix]
        [HarmonyPatch(typeof(ObjectRenderer), nameof(ObjectRenderer.Render))]
        [HarmonyPatch(typeof(DynamicRenderer), nameof(DynamicRenderer.Render))]
        private static bool ObjectRenderer_Render_Prefix()
        {
            return false;
        }
```
Replace it with (adds the static field above, and the new prefix below):
```csharp
    private class DoNotRenderEntities : PatchImpl<DoNotRenderEntities>
    {
        public static bool HideSorters;

        [HarmonyPrefix]
        [HarmonyPatch(typeof(ObjectRenderer), nameof(ObjectRenderer.Render))]
        [HarmonyPatch(typeof(DynamicRenderer), nameof(DynamicRenderer.Render))]
        private static bool ObjectRenderer_Render_Prefix()
        {
            return false;
        }

        [HarmonyPrefix]
        [HarmonyPatch(typeof(InserterRenderer), nameof(InserterRenderer.Render))]
        private static bool InserterRenderer_Render_Prefix()
        {
            return !HideSorters;
        }
```
Rationale: returns `true` (run original → render sorters) by default, identical to today; returns `false` (skip → hide sorters) only when `HideSorters` is set. The patch class is only active while the parent feature is enabled.

- [ ] **Step 6: Build to verify it compiles**

Run: `dotnet build UXAssist/UXAssist.csproj -c Release`
Expected: `Build succeeded.` with 0 errors. (Pre-existing warnings, if any, are fine.)

- [ ] **Step 7: Commit**

```bash
git add UXAssist/Patches/FactoryPatch.cs UXAssist/UXAssist.cs
git commit -m "feat: add 'hide sorters too' option for do-not-render feature"
```

---

### Task 2: UI sub-checkbox + localization

**Files:**
- Modify: `UXAssist/UIConfigWindow.cs` (I18N registration ~line 72; checkbox layout ~lines 397-399)

**Interfaces:**
- Consumes: `FactoryPatch.DoNotRenderEntitiesHideSortersEnabled` and `FactoryPatch.DoNotRenderEntitiesEnabled` (from Task 1); `wnd.AddCheckBox(x, y, tab, ConfigEntry, label[, fontSize])`, `checkbox.SetEnable(bool)`, `wnd.OnFree` (existing UI helpers).

- [ ] **Step 1: Register the label translation**

In `UXAssist/UIConfigWindow.cs`, immediately after the existing line:
```csharp
        I18N.Add("Do not render factory entities", "Do not render factory entities (except belts and sorters)", "不渲染工厂建筑实体(除了传送带和分拣器)");
```
add:
```csharp
        I18N.Add("Hide sorters too", "Hide sorters too", "同时隐藏分拣器");
```

- [ ] **Step 2: Add the indented sub-checkbox with enable/disable wiring**

In `UXAssist/UIConfigWindow.cs`, find:
```csharp
        x = 0;
        y += 36f;
        wnd.AddCheckBox(x, y, tab2, FactoryPatch.DoNotRenderEntitiesEnabled, "Do not render factory entities");
        y += 36f;
```
Replace it with:
```csharp
        x = 0;
        y += 36f;
        {
            wnd.AddCheckBox(x, y, tab2, FactoryPatch.DoNotRenderEntitiesEnabled, "Do not render factory entities");
            y += 27f;
            var hideSortersCheckBox = wnd.AddCheckBox(x + 20f, y, tab2, FactoryPatch.DoNotRenderEntitiesHideSortersEnabled, "Hide sorters too", 13);
            FactoryPatch.DoNotRenderEntitiesEnabled.SettingChanged += DoNotRenderEntitiesEnabledChanged;
            wnd.OnFree += () => { FactoryPatch.DoNotRenderEntitiesEnabled.SettingChanged -= DoNotRenderEntitiesEnabledChanged; };
            DoNotRenderEntitiesEnabledChanged(null, null);

            void DoNotRenderEntitiesEnabledChanged(object o, EventArgs e)
            {
                hideSortersCheckBox.SetEnable(FactoryPatch.DoNotRenderEntitiesEnabled.Value);
            }
        }
        y += 36f;
```
Notes: mirrors the existing `DragBuildPowerPoles` → `Alternately` block (indent `x + 20f`, font `13`, `y += 27f` for the sub-row, then `y += 36f` before the next control). `System` (for `EventArgs`) is already imported in this file.

- [ ] **Step 3: Build to verify it compiles**

Run: `dotnet build UXAssist/UXAssist.csproj -c Release`
Expected: `Build succeeded.` with 0 errors.

- [ ] **Step 4: Commit**

```bash
git add UXAssist/UIConfigWindow.cs
git commit -m "feat: add UI sub-checkbox for 'hide sorters too'"
```

---

### Task 3: Documentation + version bump

**Files:**
- Modify: `UXAssist/UXAssist.csproj` (`<Version>` line 6)
- Modify: `UXAssist/CHANGELOG.md` (English list top ~line 6; 中文 list top ~line 404)
- Modify: `UXAssist/README.md` (English feature ~lines 60-61; 中文 feature ~lines 226-227)

**Interfaces:** None (docs/metadata only).

- [ ] **Step 1: Bump the mod version**

In `UXAssist/UXAssist.csproj`, change:
```xml
    <Version>1.5.8</Version>
```
to:
```xml
    <Version>1.5.9</Version>
```

- [ ] **Step 2: Add the English changelog entry**

In `UXAssist/CHANGELOG.md`, find the first English version block:
```markdown
## Changlog

* 1.5.8
```
Insert a new version block so it reads:
```markdown
## Changlog

* 1.5.9
  * `Do not render factory entities (except belts and sorters)`: Add sub-option `Hide sorters too` to also hide sorters, leaving only belts visible.
* 1.5.8
```

- [ ] **Step 3: Add the Chinese changelog entry**

In `UXAssist/CHANGELOG.md`, find the Chinese version list:
```markdown
## 更新日志

* 1.5.8
```
Insert a new version block so it reads:
```markdown
## 更新日志

* 1.5.9
  * `不渲染工厂建筑实体(除了传送带和分拣器)`：新增子选项`同时隐藏分拣器`，可一并隐藏分拣器，只保留传送带可见。
* 1.5.8
```

- [ ] **Step 4: Add the English README sub-bullet**

In `UXAssist/README.md`, find:
```markdown
    * Do not render factory entities (except belts and sorters)
      * This also makes players click though factory entities but belts and sorters
```
Replace with:
```markdown
    * Do not render factory entities (except belts and sorters)
      * This also makes players click though factory entities but belts and sorters
      * `Hide sorters too`: also hide sorters, leaving only belts (and their cargo) visible
```

- [ ] **Step 5: Add the Chinese README sub-bullet**

In `UXAssist/README.md`, find:
```markdown
    * 不渲染工厂建筑实体(除了传送带和分拣器)
      * 这也使玩家可以点穿工厂实体直接点到传送带和分拣器
```
Replace with:
```markdown
    * 不渲染工厂建筑实体(除了传送带和分拣器)
      * 这也使玩家可以点穿工厂实体直接点到传送带和分拣器
      * `同时隐藏分拣器`：连分拣器也一并隐藏，只保留传送带（及其上的货物）可见
```

- [ ] **Step 6: Build to verify the project still loads**

Run: `dotnet build UXAssist/UXAssist.csproj -c Release`
Expected: `Build succeeded.` (confirms the bumped `<Version>` is valid XML.)

- [ ] **Step 7: Commit**

```bash
git add UXAssist/UXAssist.csproj UXAssist/CHANGELOG.md UXAssist/README.md
git commit -m "docs: document 'hide sorters too' sub-option and bump to 1.5.9"
```

---

### Task 4: Manual in-game verification (no commit)

**Files:** None (verification only).

This mod has no automated tests; verify behavior in-game. Deploy the built
`UXAssist.dll` (from `UXAssist/bin/Release/net472/`) to the game's
`BepInEx/plugins/` folder, launch DSP, and load a save with a factory that has
belts, sorters, and other buildings.

- [ ] **Step 1: Parent off — no change**
  - Ensure *Do not render factory entities* is OFF.
  - Expected: everything renders normally regardless of the sub-checkbox value.
  - Expected: the "Hide sorters too" checkbox is greyed out (disabled).

- [ ] **Step 2: Parent on, sub off — current behavior preserved**
  - Turn *Do not render factory entities* ON; leave "Hide sorters too" OFF.
  - Expected: buildings hidden; **belts + cargo visible; sorters visible** (unchanged from before this feature).

- [ ] **Step 3: Parent on, sub on — sorters hidden**
  - With the parent ON, turn "Hide sorters too" ON.
  - Expected: **sorters disappear** on the next frame; belts + their moving cargo stay visible; other buildings stay hidden.

- [ ] **Step 4: Runtime toggle**
  - Toggle "Hide sorters too" off and on again while in-game.
  - Expected: sorters reappear/disappear immediately without reloading the save.

- [ ] **Step 5: Localization**
  - Switch game language between English and 简体中文.
  - Expected: the sub-checkbox reads "Hide sorters too" / "同时隐藏分拣器" respectively.

---

## Self-Review

**Spec coverage:**
- Approach (gated `InserterRenderer.Render` prefix) → Task 1 (Steps 5). ✓
- Config entry + wiring → Task 1 (Steps 1-4). ✓
- UI indented sub-checkbox + greying out → Task 2. ✓
- Label translation EN/中文 → Task 2 (Step 1). ✓
- Docs (README EN+中文, CHANGELOG EN+中文) → Task 3. ✓
- Acceptance criteria 1-6 → Task 4 (Steps 1-5). ✓
- Out of scope (raycast, vegetation, cargo) → no tasks, intentionally untouched. ✓

**Placeholder scan:** No TBD/TODO; every code step shows exact code. ✓

**Type consistency:** `DoNotRenderEntitiesHideSortersEnabled` (`ConfigEntry<bool>`) and `DoNotRenderEntities.HideSorters` (`static bool`) are named identically across Tasks 1 and 2. UI label key `"Hide sorters too"` matches between the `AddCheckBox` call and `I18N.Add`. ✓
