# Folder chevrons alongside folder icons — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an opt-in `folder_chevrons` boolean to the project, outline, and git panels that renders an expand/collapse chevron next to the folder icon, with VS Code-style column alignment, without changing any default behavior.

**Architecture:** Each panel already resolves a `folder_icons: bool` from a `*SettingsContent` struct and renders a single folder/chevron icon slot. We add a parallel `folder_chevrons: bool` (default `false`). A directory's chevron is shown when `folder_chevrons || !folder_icons`. In the new "both" mode (`folder_icons && folder_chevrons`) the chevron renders as a leading element before the folder glyph, and non-directory rows reserve a blank chevron-width spacer so glyph columns stay aligned. All three other setting combinations keep their exact current rendering.

**Tech Stack:** Rust, GPUI, Zed settings system (`settings_content`, `RegisterSetting`/`Settings`), `FileIcons::{get_folder_icon, get_chevron_icon}`.

**Spec:** `docs/superpowers/specs/2026-06-29-folder-chevrons-with-icons-design.md`

## Global Constraints

- New setting key is exactly `folder_chevrons`; default `false` in all three panels. Human-facing copy says "arrows (chevrons)".
- Chevron-show rule is identical in all three panels: `show_chevron = folder_chevrons || !folder_icons`.
- No behavior changes for existing configs: `folder_icons: true` (default) keeps glyph-only; existing `folder_icons: false` keeps chevron-only.
- No new SVG icons; reuse the active icon theme's `chevron_icons` via `FileIcons::get_chevron_icon`.
- Build/lint with `./script/clippy` (not `cargo clippy`), per CLAUDE.md.
- Avoid `unwrap()` in non-test code except the established settings `.unwrap()` pattern that mirrors existing fields. No `let _ =` on fallible calls.
- Do **not** commit or push until the user explicitly asks. Steps below include `git add`/`git commit` for the agentic worker's per-task hygiene; if the user has said "don't commit", stage only and report instead.
- PR title style (when we open it): imperative, no conventional-commit prefix, e.g. `project_panel: Show folder chevrons alongside folder icons`. PR body ends with a `Release Notes:` section.

---

### Task 1: Project panel — settings, render, and unit tests

This is the reference vertical slice. The other panels mirror it.

**Files:**
- Modify: `crates/settings_content/src/workspace.rs:750` (`ProjectPanelSettingsContent`)
- Modify: `crates/project_panel/src/project_panel_settings.rs:22,107` (`ProjectPanelSettings`)
- Modify: `assets/settings/default.json` (~line 792, project_panel block)
- Modify: `crates/project_panel/src/project_panel.rs` (`EntryDetails` ~269; `details_for_entry` ~6345; `render_entry` icon child ~6008)
- Test: `crates/project_panel/src/project_panel_tests.rs`

**Interfaces:**
- Produces: `ProjectPanelSettings.folder_chevrons: bool`; `EntryDetails.folder_chevron: Option<SharedString>`.
- Consumes: existing `FileIcons::get_chevron_icon(expanded: bool, cx) -> Option<SharedString>` and `FileIcons::get_folder_icon(expanded: bool, path: &Path, cx) -> Option<SharedString>`.

- [ ] **Step 1: Add the content-struct field**

In `crates/settings_content/src/workspace.rs`, in `ProjectPanelSettingsContent`, add immediately after the `folder_icons` field (line 750):

```rust
    pub folder_chevrons: Option<bool>,
```

- [ ] **Step 2: Add the resolved-settings field and default mapping**

In `crates/project_panel/src/project_panel_settings.rs`, add to `struct ProjectPanelSettings` after `folder_icons` (line 22):

```rust
    pub folder_chevrons: bool,
```

and in `from_settings` after the `folder_icons` mapping (line 107):

```rust
            folder_chevrons: project_panel.folder_chevrons.unwrap(),
```

- [ ] **Step 3: Add the default to `default.json`**

In `assets/settings/default.json`, in the `project_panel` block, immediately after `"folder_icons": true,` (line ~792):

```jsonc
    // Whether to also show an expand/collapse arrow (chevron) next to folder icons.
    "folder_chevrons": false,
```

- [ ] **Step 4: Add the `EntryDetails` field**

In `crates/project_panel/src/project_panel.rs`, in `struct EntryDetails` (line 269), add after `icon`:

```rust
    folder_chevron: Option<SharedString>,
```

- [ ] **Step 5: Compute icon + chevron in `details_for_entry`**

In `details_for_entry` (line ~6356), replace the settings read and the `icon` computation block (lines 6356-6384) with:

```rust
        let (show_file_icons, show_folder_icons, show_folder_chevrons) = {
            let settings = ProjectPanelSettings::get_global(cx);
            (
                settings.file_icons,
                settings.folder_icons,
                settings.folder_chevrons,
            )
        };

        let expanded_entry_ids = self
            .state
            .expanded_dir_ids
            .get(&worktree_id)
            .map(Vec::as_slice)
            .unwrap_or(&[]);
        let is_expanded = expanded_entry_ids.binary_search(&entry.id).is_ok();

        let (icon, folder_chevron) = match entry.kind {
            EntryKind::File => {
                let icon = if show_file_icons {
                    FileIcons::get_icon(entry.path.as_std_path(), cx)
                } else {
                    None
                };
                (icon, None)
            }
            _ => {
                let chevron = FileIcons::get_chevron_icon(is_expanded, cx);
                if show_folder_icons {
                    let glyph =
                        FileIcons::get_folder_icon(is_expanded, entry.path.as_std_path(), cx);
                    if show_folder_chevrons {
                        (glyph, chevron)
                    } else {
                        (glyph, None)
                    }
                } else {
                    (chevron, None)
                }
            }
        };
```

- [ ] **Step 6: Populate the new field in the returned `EntryDetails`**

In the `EntryDetails { … }` literal at the end of `details_for_entry` (line ~6430), add after `icon,`:

```rust
            folder_chevron,
```

- [ ] **Step 7: Write the failing unit tests (truth table + file)**

Add to `crates/project_panel/src/project_panel_tests.rs` (near the other `#[gpui::test]` functions). This collects `EntryDetails` for the single directory and the single file via `for_each_visible_entry`:

```rust
#[gpui::test]
async fn test_folder_chevrons_setting(cx: &mut gpui::TestAppContext) {
    init_test(cx);

    let fs = FakeFs::new(cx.executor());
    fs.insert_tree(
        "/root",
        json!({
            "dir": { "nested.txt": "" },
            "file.txt": "",
        }),
    )
    .await;

    let project = Project::test(fs.clone(), ["/root".as_ref()], cx).await;
    let workspace = cx
        .add_window(|window, cx| Workspace::test_new(project.clone(), window, cx));
    let cx = &mut VisualTestContext::from_window(*workspace, cx);
    let panel = workspace
        .update(cx, |workspace, window, cx| {
            ProjectPanel::new(workspace, window, cx)
        })
        .unwrap();

    // (show_folder_icons, show_folder_chevrons) ->
    //   (dir_icon.is_some, dir_chevron.is_some)
    let cases = [
        ((true, false), (true, false)),  // today's default: glyph only
        ((true, true), (true, true)),    // both
        ((false, false), (true, false)), // today's false: chevron in icon slot
        ((false, true), (true, false)),  // chevron only
    ];

    for ((folder_icons, folder_chevrons), (expect_icon, expect_chevron)) in cases {
        cx.update(|_, cx| {
            cx.update_global::<SettingsStore, _>(|store, cx| {
                store.update_user_settings(cx, |settings| {
                    let panel = settings.project_panel.get_or_insert_default();
                    panel.folder_icons = Some(folder_icons);
                    panel.folder_chevrons = Some(folder_chevrons);
                });
            });
        });
        cx.run_until_parked();

        let mut dir = None;
        let mut file = None;
        panel.update_in(cx, |panel, window, cx| {
            panel.for_each_visible_entry(0..10, window, cx, &mut |_, details, _, _| {
                if details.filename == "dir" {
                    dir = Some((details.icon.is_some(), details.folder_chevron.is_some()));
                } else if details.filename == "file.txt" {
                    file = Some((details.icon.is_some(), details.folder_chevron.is_some()));
                }
            });
        });

        assert_eq!(
            dir,
            Some((expect_icon, expect_chevron)),
            "dir with folder_icons={folder_icons}, folder_chevrons={folder_chevrons}"
        );
        // Files never carry a folder chevron, in any mode.
        assert_eq!(
            file.map(|(_, chevron)| chevron),
            Some(false),
            "file with folder_icons={folder_icons}, folder_chevrons={folder_chevrons}"
        );
    }
}
```

> If `ProjectPanel::new`, `Workspace::test_new`, `VisualTestContext`, or `Project::test` are imported under different names in this test module, match the imports used by the nearest existing `#[gpui::test]` (e.g. `init_test` neighbors). Do not invent helpers.

- [ ] **Step 8: Run the test to verify it fails**

Run: `cargo test -p project_panel test_folder_chevrons_setting`
Expected: FAIL to compile (`folder_chevron`/`folder_chevrons` unknown) until Steps 1-6 are in, then assertion logic verified.

- [ ] **Step 9: Render the chevron and the file spacer in `render_entry`**

In `render_entry`, the `ListItem` `.child(if let Some(icon) = &icon { … })` block starts at line ~6008. Prepend a leading slot **before** that `.child(...)` call so the row becomes `[chevron-or-spacer][icon][name]`. Insert this `.child(...)` immediately before the existing icon `.child(`:

```rust
                    .when_some(details.folder_chevron.clone(), |this, chevron| {
                        this.child(
                            h_flex()
                                .flex_none()
                                .child(Icon::from_path(chevron.to_string()).color(Color::Muted)),
                        )
                    })
                    .when(
                        details.folder_chevron.is_none()
                            && settings.folder_icons
                            && settings.folder_chevrons,
                        |this| {
                            this.child(
                                h_flex()
                                    .size(IconSize::default().rems())
                                    .invisible()
                                    .flex_none(),
                            )
                        },
                    )
```

`details` is in scope in `render_entry` (it is read throughout this function); `settings` is the `ProjectPanelSettings` already bound near the top of `render_entry`. The spacer mirrors the existing no-icon placeholder at lines 6052-6056, so files in both-mode align under folders.

- [ ] **Step 10: Run the test to verify it passes**

Run: `cargo test -p project_panel test_folder_chevrons_setting`
Expected: PASS.

- [ ] **Step 11: Build and lint**

Run: `./script/clippy -p project_panel -p settings_content` then `cargo build -p project_panel`
Expected: no errors/warnings introduced.

- [ ] **Step 12: Commit**

```bash
git add crates/settings_content/src/workspace.rs \
        crates/project_panel/src/project_panel_settings.rs \
        crates/project_panel/src/project_panel.rs \
        crates/project_panel/src/project_panel_tests.rs \
        assets/settings/default.json
git commit -m "project_panel: Add folder_chevrons setting to show chevrons with folder icons"
```

---

### Task 2: Outline panel — settings and render

**Files:**
- Modify: `crates/settings_content/src/settings_content.rs:1087` (`OutlinePanelSettingsContent`)
- Modify: `crates/outline_panel/src/outline_panel_settings.rs:13,56` (`OutlinePanelSettings`)
- Modify: `assets/settings/default.json` (~line 907, outline_panel block)
- Modify: `crates/outline_panel/src/outline_panel.rs:2390-2393` and `2487-2490`
- Test: `crates/outline_panel/src/outline_panel_settings.rs` (or the crate's existing test module)

**Interfaces:**
- Produces: `OutlinePanelSettings.folder_chevrons: bool`.

- [ ] **Step 1: Add the content-struct field**

In `crates/settings_content/src/settings_content.rs`, in `OutlinePanelSettingsContent`, after `folder_icons` (line 1087):

```rust
    pub folder_chevrons: Option<bool>,
```

- [ ] **Step 2: Add the resolved-settings field and default mapping**

In `crates/outline_panel/src/outline_panel_settings.rs`, after `folder_icons` in the struct (line 13):

```rust
    pub folder_chevrons: bool,
```

and in `from_settings` after the `folder_icons` mapping (line 56):

```rust
            folder_chevrons: panel.folder_chevrons.unwrap(),
```

- [ ] **Step 3: Add the default to `default.json`**

In `assets/settings/default.json`, in the `outline_panel` block, after its `"folder_icons": true,` (line ~907):

```jsonc
    // Whether to also show an expand/collapse arrow (chevron) next to folder icons.
    "folder_chevrons": false,
```

- [ ] **Step 4: Render chevron + glyph at the first directory site**

In `crates/outline_panel/src/outline_panel.rs`, replace the `icon` computation at lines 2390-2396 (the `let icon = if settings.folder_icons { … }.map(Icon::from_path).map(|icon| icon.color(color).into_any_element());`) with a small helper-inline that produces a leading chevron when needed. Replace with:

```rust
                let show_chevron = settings.folder_chevrons || !settings.folder_icons;
                let glyph = if settings.folder_icons {
                    FileIcons::get_folder_icon(
                        is_expanded,
                        directory.entry.path.as_std_path(),
                        cx,
                    )
                } else {
                    FileIcons::get_chevron_icon(is_expanded, cx)
                }
                .map(Icon::from_path)
                .map(|icon| icon.color(color).into_any_element());
                let leading_chevron = (settings.folder_icons && show_chevron)
                    .then(|| FileIcons::get_chevron_icon(is_expanded, cx))
                    .flatten()
                    .map(Icon::from_path)
                    .map(|icon| icon.color(color).into_any_element());
                let icon = match (leading_chevron, glyph) {
                    (Some(chevron), Some(glyph)) => Some(
                        h_flex()
                            .gap_1()
                            .child(chevron)
                            .child(glyph)
                            .into_any_element(),
                    ),
                    (None, glyph) => glyph,
                    (Some(chevron), None) => Some(chevron),
                };
```

> `h_flex` is already imported in this file (it renders rows throughout). Verify the surrounding code binds `icon` as `Option<AnyElement>`; the `into_any_element()` calls keep the type identical to the original.

- [ ] **Step 5: Render chevron + glyph at the folded-directory site**

Apply the same transformation at lines 2487-2493 (the folded-directory `let icon = …`), using `&Path::new(&name)` as the folder-icon path (matching the original) and `is_expanded` from that block:

```rust
            let show_chevron = settings.folder_chevrons || !settings.folder_icons;
            let glyph = if settings.folder_icons {
                FileIcons::get_folder_icon(is_expanded, &Path::new(&name), cx)
            } else {
                FileIcons::get_chevron_icon(is_expanded, cx)
            }
            .map(Icon::from_path)
            .map(|icon| icon.color(color).into_any_element());
            let leading_chevron = (settings.folder_icons && show_chevron)
                .then(|| FileIcons::get_chevron_icon(is_expanded, cx))
                .flatten()
                .map(Icon::from_path)
                .map(|icon| icon.color(color).into_any_element());
            let icon = match (leading_chevron, glyph) {
                (Some(chevron), Some(glyph)) => Some(
                    h_flex()
                        .gap_1()
                        .child(chevron)
                        .child(glyph)
                        .into_any_element(),
                ),
                (None, glyph) => glyph,
                (Some(chevron), None) => Some(chevron),
            };
```

> Outline-panel entries are headings/symbols, not a uniform file grid; per the spec we do not add a file-row spacer here — the chevron simply precedes the folder glyph on directory rows. This matches the panel's existing variable-icon layout.

- [ ] **Step 6: Write the settings-resolution test**

Add to the outline_panel test module (search the crate for an existing `#[gpui::test]` or `#[test]` that builds `OutlinePanelSettings`; if none exists, add a `#[gpui::test]` mirroring `project_panel`'s `init_test` settings-store pattern):

```rust
#[gpui::test]
async fn test_outline_panel_folder_chevrons_default(cx: &mut gpui::TestAppContext) {
    init_test(cx); // crate-local init that installs a test SettingsStore + JustBase theme
    let settings = cx.read(|cx| OutlinePanelSettings::get_global(cx).folder_chevrons);
    assert!(!settings, "folder_chevrons must default to false");
}
```

> If the outline_panel crate has no `init_test`, reuse the pattern from `crates/project_panel/src/project_panel_tests.rs:10805` (create `SettingsStore::test`, `theme_settings::init(JustBase)`, `crate::init`). Keep it minimal.

- [ ] **Step 7: Build, lint, test**

Run: `./script/clippy -p outline_panel -p settings_content` then `cargo test -p outline_panel folder_chevrons`
Expected: builds clean; test passes.

- [ ] **Step 8: Commit**

```bash
git add crates/settings_content/src/settings_content.rs \
        crates/outline_panel/src/outline_panel_settings.rs \
        crates/outline_panel/src/outline_panel.rs \
        assets/settings/default.json
git commit -m "outline_panel: Add folder_chevrons setting to show chevrons with folder icons"
```

---

### Task 3: Git panel — settings, render, and settings watcher

**Files:**
- Modify: `crates/settings_content/src/settings_content.rs:668` (`GitPanelSettingsContent`)
- Modify: `crates/git_ui/src/git_panel_settings.rs:26,69` (`GitPanelSettings`)
- Modify: `assets/settings/default.json` (~line 978, git_panel block)
- Modify: `crates/git_ui/src/git_panel.rs:6853-6904` (directory render) and `:911,919,946,953` (settings watcher)
- Test: `crates/git_ui/src/git_panel_settings.rs` or git_ui test module

**Interfaces:**
- Produces: `GitPanelSettings.folder_chevrons: bool`.

- [ ] **Step 1: Add the content-struct field**

In `crates/settings_content/src/settings_content.rs`, in `GitPanelSettingsContent`, after `folder_icons` (line 668):

```rust
    pub folder_chevrons: Option<bool>,
```

- [ ] **Step 2: Add the resolved-settings field and default mapping**

In `crates/git_ui/src/git_panel_settings.rs`, after `folder_icons` in the struct (line 26):

```rust
    pub folder_chevrons: bool,
```

and in `from_settings` after the `folder_icons` mapping (line 69):

```rust
            folder_chevrons: git_panel.folder_chevrons.unwrap(),
```

- [ ] **Step 3: Add the default to `default.json`**

In `assets/settings/default.json`, in the `git_panel` block, after its `"folder_icons": true,` (line ~978):

```jsonc
    // Whether to also show an expand/collapse arrow (chevron) next to folder icons.
    "folder_chevrons": false,
```

- [ ] **Step 4: Render the leading chevron on directory rows**

In `crates/git_ui/src/git_panel.rs`, after the existing `folder_icon` / `fallback_folder_icon` bindings (lines 6853-6871), add a chevron binding:

```rust
        let show_chevron = settings.folder_chevrons || !settings.folder_icons;
        let leading_chevron = (settings.folder_icons && show_chevron).then(|| {
            FileIcons::get_chevron_icon(entry.expanded, cx)
                .map(|path| Icon::from_path(path).size(IconSize::Small).color(Color::Muted))
                .unwrap_or_else(|| {
                    Icon::new(if entry.expanded {
                        IconName::ChevronDown
                    } else {
                        IconName::ChevronRight
                    })
                    .size(IconSize::Small)
                    .color(Color::Muted)
                })
        });
```

Then in the `name_row` builder (line ~6887), prepend the chevron as the first child of the `h_flex`, before the existing folder-icon `.child(...)`:

```rust
        let name_row = h_flex()
            .min_w_0()
            .gap_1()
            .pl(px(entry.depth as f32 * TREE_INDENT))
            .when_some(leading_chevron, |this, chevron| this.child(chevron))
            .child(
                folder_icon
                    .map(|folder_icon| {
                        Icon::from_path(folder_icon)
                            .size(IconSize::Small)
                            .color(Color::Muted)
                    })
                    .unwrap_or_else(|| {
                        Icon::new(fallback_folder_icon)
                            .size(IconSize::Small)
                            .color(Color::Muted)
                    }),
            )
            .child(self.entry_label(entry.name.clone(), label_color).truncate());
```

> Git-panel file rows are rendered separately (this function renders a directory entry). Per the spec we keep the change scoped to directory rows: the chevron precedes the folder glyph; we do not introduce a file-row spacer in the git panel because its file and directory rows already use independent layouts. If, on visual review, file/dir glyph columns look misaligned in both-mode, add a matching `IconSize::Small` invisible spacer to the file-row `name_row`; note that as a follow-up rather than guessing now.

- [ ] **Step 5: React to the new setting in the settings watcher**

In the `observe_global_in::<SettingsStore>` block (lines 905-955), add a `was_folder_chevrons` tracker mirroring `was_folder_icons`:

- After line 911 (`let mut was_folder_icons = …`):

```rust
            let mut was_folder_chevrons = GitPanelSettings::get_global(cx).folder_chevrons;
```

- After line 919 (`let folder_icons = settings.folder_icons;`):

```rust
                let folder_chevrons = settings.folder_chevrons;
```

- Change the redraw condition at line 946 from:

```rust
                if file_icons != was_file_icons || folder_icons != was_folder_icons {
                    cx.notify();
                }
```

to:

```rust
                if file_icons != was_file_icons
                    || folder_icons != was_folder_icons
                    || folder_chevrons != was_folder_chevrons
                {
                    cx.notify();
                }
```

- After line 953 (`was_folder_icons = folder_icons;`):

```rust
                was_folder_chevrons = folder_chevrons;
```

- [ ] **Step 6: Write the settings-resolution test**

Add a default-value test in the git_ui test module (mirror the outline-panel test; use the crate's existing test init if present):

```rust
#[gpui::test]
async fn test_git_panel_folder_chevrons_default(cx: &mut gpui::TestAppContext) {
    init_test(cx);
    let value = cx.read(|cx| GitPanelSettings::get_global(cx).folder_chevrons);
    assert!(!value, "folder_chevrons must default to false");
}
```

> Use whatever `init_test`/settings-store bootstrap the git_ui tests already use. If none, replicate the minimal `SettingsStore::test` + `JustBase` + `crate::init` pattern.

- [ ] **Step 7: Build, lint, test**

Run: `./script/clippy -p git_ui -p settings_content` then `cargo test -p git_ui folder_chevrons`
Expected: builds clean; test passes.

- [ ] **Step 8: Commit**

```bash
git add crates/settings_content/src/settings_content.rs \
        crates/git_ui/src/git_panel_settings.rs \
        crates/git_ui/src/git_panel.rs \
        assets/settings/default.json
git commit -m "git_panel: Add folder_chevrons setting to show chevrons with folder icons"
```

---

### Task 4: Settings UI toggles

**Files:**
- Modify: `crates/settings_ui/src/page_data.rs` (project_panel ~5032, outline_panel ~5705, git_panel ~6024)

**Interfaces:**
- Consumes: the three `folder_chevrons` content fields from Tasks 1-3.

- [ ] **Step 1: Read the existing `folder_icons` toggle entries**

Open `crates/settings_ui/src/page_data.rs` and read the three `folder_icons` settings-item definitions at lines ~5032, ~5705, ~6024. Each is a struct literal with `json_path: Some("<panel>.folder_icons")`, a getter closure reading `…folder_icons`, and a setter closure writing `…folder_icons = value`.

- [ ] **Step 2: Add a parallel `folder_chevrons` item for the project panel**

Immediately after the project-panel `folder_icons` item (the block ending near line 5050), add a sibling item that is identical except: `json_path: Some("project_panel.folder_chevrons")`, getter reads `…folder_chevrons`, setter writes `…folder_chevrons = value`, and a user-facing title/description like `"Folder Chevrons"` / `"Show expand/collapse arrows (chevrons) next to folder icons."`. Copy the exact surrounding struct shape from the adjacent `folder_icons` item — do not invent field names.

- [ ] **Step 3: Add the parallel item for the outline panel**

Same as Step 2 but after the outline-panel `folder_icons` item (~line 5717), with `json_path: Some("outline_panel.folder_chevrons")` and outline getters/setters.

- [ ] **Step 4: Add the parallel item for the git panel**

Same as Step 2 but after the git-panel `folder_icons` item (~line 6032), with `json_path: Some("git_panel.folder_chevrons")` and git getters/setters.

- [ ] **Step 5: Build and lint**

Run: `./script/clippy -p settings_ui`
Expected: builds clean.

- [ ] **Step 6: Commit**

```bash
git add crates/settings_ui/src/page_data.rs
git commit -m "settings_ui: Add folder_chevrons toggles for project, outline, and git panels"
```

---

### Task 5: Docs and VS Code importer note

**Files:**
- Modify: `docs/src/visual-customization.md` (~line 472 and ~line 592)
- Modify: `docs/src/reference/all-settings.md` (~line 4956 and ~line 5493)
- Modify: `crates/settings/src/vscode_import.rs` (~line 808, near `folder_icons: None`)

**Interfaces:** none (docs/comments only).

- [ ] **Step 1: Document the setting in `visual-customization.md`**

In `docs/src/visual-customization.md`, next to each `"folder_icons"` example line (~472 and ~592), add:

```jsonc
    "folder_chevrons": false,   // Show expand/collapse arrows (chevrons) next to folder icons
```

- [ ] **Step 2: Document the setting in `all-settings.md`**

`docs/src/reference/all-settings.md` is a hand-maintained reference (it is not generated by a script in this repo — confirm with `grep -rn "all-settings" script/`; if a generator exists, run it instead of editing). Add `"folder_chevrons": false,` immediately after each `"folder_icons": true,` occurrence (~4956 and ~5493), matching the surrounding indentation and any inline comment style.

- [ ] **Step 3: Add a VS Code importer comment**

In `crates/settings/src/vscode_import.rs`, near the project-panel mapping that sets `folder_icons` (line ~808, where it is `None`), add a brief comment documenting that `folder_chevrons` is intentionally not imported because VS Code always shows arrows and we keep Zed's default opt-in:

```rust
            // VS Code always shows folder arrows; we leave `folder_chevrons`
            // unset so Zed keeps its opt-in default rather than forcing them on.
            folder_chevrons: None,
```

Place the field in the same struct literal as `folder_icons: None,` so the content struct is fully constructed.

- [ ] **Step 4: Verify docs build references and grep for misses**

Run: `grep -rn "folder_chevrons" docs/ crates/ assets/`
Expected: entries in all three panels' defaults, both docs files, settings UI, importer, and code — no stray `folder_arrows`.

- [ ] **Step 5: Commit**

```bash
git add docs/src/visual-customization.md docs/src/reference/all-settings.md crates/settings/src/vscode_import.rs
git commit -m "docs: Document folder_chevrons setting"
```

---

### Task 6: Visual regression scenario

**Files:**
- Modify: `crates/zed/src/visual_test_runner.rs` (near the `project_panel` scenario at ~line 396)

**Interfaces:** Consumes the rendering from Task 1.

- [ ] **Step 1: Read the existing project_panel visual scenario**

Open `crates/zed/src/visual_test_runner.rs` and read the project-panel scenario block (~lines 320-415): how the panel is loaded/opened, how settings are applied (look for a `SettingsStore` update like the one at ~line 969-972), and how `capture_and_compare`/`run_*` save a baseline named `"project_panel"`.

- [ ] **Step 2: Add a both-mode scenario**

After the existing `project_panel` capture (~line 415), set `folder_icons: true, folder_chevrons: true` via the same `SettingsStore::update_user_settings` mechanism the runner already uses, re-stabilize the UI, and capture a baseline named `project_panel_folder_chevrons`. Mirror the exact helper calls used by the existing `project_panel` scenario (e.g. `capture_and_compare(... "project_panel_folder_chevrons" ..., update_baseline)`); do not introduce a new capture helper.

```rust
    // Enable both folder icons and chevrons, then re-capture.
    cx.update(|cx| {
        cx.update_global::<settings::SettingsStore, _>(|store, cx| {
            store.update_user_settings(cx, |settings| {
                let panel = settings.project_panel.get_or_insert_default();
                panel.folder_icons = Some(true);
                panel.folder_chevrons = Some(true);
            });
        });
    });
    // (re-stabilize + capture "project_panel_folder_chevrons" using the same
    //  pattern as the existing project_panel scenario above)
```

- [ ] **Step 3: Generate baselines locally and run the suite (macOS only)**

Per `docs/src/development/macos.md`:

```sh
git stash            # known-good tree, or: git checkout upstream/main
UPDATE_BASELINE=1 cargo run -p zed --bin zed_visual_test_runner --features visual-tests
git stash pop        # restore changes
UPDATE_BASELINE=1 cargo run -p zed --bin zed_visual_test_runner --features visual-tests
cargo run -p zed --bin zed_visual_test_runner --features visual-tests
```

Expected: the new `project_panel_folder_chevrons` scenario passes against its freshly generated (gitignored) baseline; existing scenarios still pass. If not on macOS, skip running and note that baselines must be generated on a macOS machine before merge.

- [ ] **Step 4: Build and lint**

Run: `./script/clippy -p zed --features visual-tests` (or `cargo build -p zed --bin zed_visual_test_runner --features visual-tests`)
Expected: builds clean.

- [ ] **Step 5: Commit**

```bash
git add crates/zed/src/visual_test_runner.rs
git commit -m "zed: Add visual regression scenario for folder chevrons"
```

---

## Final verification (before opening the PR)

- [ ] `cargo test -p project_panel -p outline_panel -p git_ui folder_chevrons` all pass.
- [ ] `./script/clippy` clean for every touched crate.
- [ ] `grep -rn "folder_arrows" .` returns nothing (the chosen key is `folder_chevrons`).
- [ ] Manually run `cargo run` and capture light + dark screenshots of the project panel in all four `folder_icons` × `folder_chevrons` combinations (focus on both-mode alignment of file vs folder glyphs) for the PR description.
- [ ] PR body ends with:

  ```
  Release Notes:

  - Added a `folder_chevrons` setting to show expand/collapse arrows (chevrons) alongside folder icons in the project, outline, and git panels (#8661).
  ```

## Self-review notes (resolved)

- **Spec coverage:** setting semantics (Tasks 1-3), VS Code alignment spacer (Task 1 Step 9), three-panel scope (Tasks 1-3), settings UI (Task 4), docs + importer (Task 5), unit + visual tests (Tasks 1/2/3/6), PR hygiene (final section). All spec sections map to a task.
- **Outline/git file-row spacer:** the spec's spacer parity is concretely required only for the project panel's uniform file grid; for outline (heading rows) and git (separate file-row layout) the plan prepends the chevron without a spacer and flags alignment as a visual-review follow-up rather than guessing layout code. This is a deliberate, documented narrowing, not a gap.
- **Type consistency:** key is `folder_chevrons` everywhere; `EntryDetails.folder_chevron` (singular) is the only place the singular form appears, and only within Task 1.
