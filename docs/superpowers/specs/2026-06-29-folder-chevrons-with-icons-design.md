# Show both folder icons and expand/collapse arrows

**Issue:** [zed-industries/zed#8661](https://github.com/zed-industries/zed/issues/8661) — "project panel: Allow showing both folder icons and open/closed arrows"

**Date:** 2026-06-29

## Problem

In Zed's tree panels a directory shows *either* a folder icon *or* a disclosure
chevron, never both. The `folder_icons` boolean toggles between them:

- `folder_icons: true` (default) → folder glyph, no chevron.
- `folder_icons: false` → chevron, no folder glyph.

Users migrating from VS Code / Atom expect the folder glyph **and** a leading
expand/collapse arrow at the same time. Without the arrow, folders visually
blend into files, and there is no affordance signalling that a row is
expandable, nor whether an empty-looking folder is collapsed or genuinely empty.

VS Code shows both by default. We do **not** want both by default (it is
visually noisier, per maintainer feedback on the issue), but we want an opt-in
setting that enables it.

## Goal

Add a new opt-in boolean `folder_chevrons` to the `project_panel`,
`outline_panel`, and `git_panel` settings. When enabled alongside
`folder_icons`, a directory row renders a leading chevron followed by the folder
glyph. Default behavior is unchanged.

## Non-goals

- Changing any default appearance. Fresh installs and existing configs look
  identical after this change.
- Replacing the `folder_icons` boolean with an enum (rejected: breaks the
  existing boolean across three panels, the VS Code settings importer, and the
  settings UI, and needs a migration).
- New icons. The chevron reuses the active icon theme's existing
  `chevron_icons`, so no SVGs are added (per CONTRIBUTING.md "no new file
  icons").

## Setting semantics

A new `folder_chevrons: bool` is added next to `folder_icons` in each of the three
panels. Default `false`. The chevron is shown for a directory when:

```text
show_arrow = folder_chevrons == true  OR  folder_icons == false
```

The folder glyph is shown when `folder_icons == true` (unchanged). This yields a
fully backwards-compatible truth table:

| `folder_icons`   | `folder_chevrons`   | Directory renders            | Status            |
| ---------------- | ----------------- | ---------------------------- | ----------------- |
| `true` (default) | `false` (default) | folder glyph only            | today's default   |
| `true`           | `true`            | **chevron + folder glyph**   | the new feature   |
| `false`          | `false`           | chevron only                 | today's `false`   |
| `false`          | `true`            | chevron only                 | same as above     |

Because the chevron always shows whenever `folder_icons` is off, no existing
user's appearance changes on upgrade: `folder_chevrons` defaults to `false`, the
`folder_icons: true` default keeps showing the glyph only, and existing
`folder_icons: false` users keep seeing the chevron via the `|| folder_icons ==
false` clause.

## Indentation / alignment parity with VS Code

In VS Code the chevron occupies a fixed-width leading column. Rows **without** a
chevron (files, and — in VS Code — empty folders) reserve that same blank width
so every icon stays vertically aligned in a single column.

We replicate this **only in the "both" mode** (`folder_icons: true &&
folder_chevrons: true`), because that is the only mode that introduces an extra
leading element. In the other three modes the chevron occupies the single
existing icon slot, so alignment is already correct and nothing changes.

In "both" mode:

- A **directory** renders: `[chevron][folder glyph][name]`.
- A **file** renders: `[blank chevron-width spacer][file glyph][name]`, so the
  file glyph aligns under the directory's folder glyph rather than under its
  chevron.

The spacer is an invisible, flex-none element sized to the chevron's width
(`IconSize::default`, matching the existing invisible placeholder already used
at project_panel.rs:6052-6056 for entries with no icon). This keeps the file and
folder glyph columns aligned exactly as in the VS Code reference screenshot.

All Zed directory entries are expandable (even empty ones render a chevron
today in chevron-only mode), so — unlike VS Code — folders always get a real
chevron, never a spacer. This is consistent with Zed's current chevron-only
behavior and needs no empty-vs-non-empty detection.

## Architecture and touch points

The change is small and mechanical, repeated across three parallel panels. Each
panel already has: a content struct (`Option<bool>`), a resolved settings struct
(`bool`), a default in `default.json`, an entry-detail/icon computation, a
render path, and a settings-UI toggle. We extend each in the same way.

### Shared data model (project_panel example; mirror in the others)

`EntryDetails` (project_panel.rs:269) currently carries a single
`icon: Option<SharedString>`. Add:

```rust
folder_chevron: Option<SharedString>,
```

This holds the chevron path when a chevron should render *in addition to* the
folder glyph. It is `None` when the chevron occupies the main `icon` slot (the
three non-"both" modes) or for files.

`details_for_entry` (project_panel.rs:6345) computes both fields:

```rust
let (show_file_icons, show_folder_icons, show_folder_chevrons) = {
    let settings = ProjectPanelSettings::get_global(cx);
    (settings.file_icons, settings.folder_icons, settings.folder_chevrons)
};
// ... is_expanded as today ...
let (icon, folder_chevron) = match entry.kind {
    EntryKind::File => (
        if show_file_icons {
            FileIcons::get_icon(entry.path.as_std_path(), cx)
        } else {
            None
        },
        None,
    ),
    _ => {
        let chevron = FileIcons::get_chevron_icon(is_expanded, cx);
        if show_folder_icons {
            let glyph = FileIcons::get_folder_icon(is_expanded, entry.path.as_std_path(), cx);
            if show_folder_chevrons {
                (glyph, chevron) // both
            } else {
                (glyph, None) // folder glyph only (today's default)
            }
        } else {
            (chevron, None) // chevron in the main slot (today's `false`)
        }
    }
};
```

The "blank spacer for files in both mode" is a render-time concern, derived from
settings at render rather than stored on every file's `EntryDetails` — see
render below.

### Render

In `render_entry` (project_panel.rs:5935+, the `ListItem` builder), before the
existing `.child(icon …)` block, prepend the chevron when `details.folder_chevron`
is `Some`, and prepend an invisible spacer for files when both-mode is active:

- Directory with `folder_chevron: Some(path)` → leading
  `Icon::from_path(path).color(Color::Muted)` sized like the existing icons.
- File, when `folder_icons && folder_chevrons` (read from settings in
  `render_entry`, which already reads `settings`) → leading invisible spacer
  matching the existing `h_flex().size(IconSize::default().rems()).invisible().flex_none()`
  placeholder.
- Otherwise → no leading element (unchanged for all current configurations).

The existing diagnostic-decoration logic stays on the main `icon` slot
untouched; the chevron is a plain, undecorated leading glyph.

`DraggedProjectEntryView` (project_panel.rs:5802) keeps using `details.icon`
(the folder/file glyph) only — the drag preview does not need the chevron.

### outline_panel and git_panel

Apply the identical pattern:

- `outline_panel`: chevron logic lives at outline_panel.rs:2390-2393 and
  2487-2490; settings struct `outline_panel_settings.rs`. The outline panel uses
  `get_chevron_icon`/`get_folder_icon` the same way.
- `git_panel`: chevron logic at git_panel.rs:6853-6858; settings struct
  `git_panel_settings.rs`. Note git_panel already watches `folder_icons` changes
  at git_panel.rs:911-953 — extend that watcher to also react to
  `folder_chevrons` so the panel refreshes when the setting changes.

### Settings plumbing (all three panels)

1. **Content structs** — add `pub folder_chevrons: Option<bool>` next to each
   `folder_icons: Option<bool>`:
   - `ProjectPanelSettingsContent` — `settings_content/src/workspace.rs:750`.
   - `OutlinePanelSettingsContent` — `settings_content/src/settings_content.rs:1087`.
   - `GitPanelSettingsContent` — `settings_content/src/settings_content.rs:668`.
2. **Resolved settings structs** — add `pub folder_chevrons: bool` and the
   `.unwrap()` default mapping:
   - `project_panel/src/project_panel_settings.rs:22,107`
   - `outline_panel/src/outline_panel_settings.rs:13,56`
   - `git_ui/src/git_panel_settings.rs:26,69`
3. **`assets/settings/default.json`** — add `"folder_chevrons": false` with a doc
   comment immediately after each `folder_icons` (lines ~792, ~907, ~978):
   ```jsonc
   // Whether to show folder icons or chevrons for directories in the project panel.
   "folder_icons": true,
   // Whether to also show an expand/collapse arrow (chevron) next to folder icons.
   "folder_chevrons": false,
   ```
4. **Settings UI** — add a toggle next to each existing `folder_icons` toggle in
   `settings_ui/src/page_data.rs` (project_panel ~5032, outline_panel ~5705,
   git_panel ~6024), wired to read/write the new `folder_chevrons` field.
5. **VS Code importer** — `settings/src/vscode_import.rs`. VS Code's tree always
   shows arrows, so map nothing automatically (leave `folder_chevrons: None`,
   matching the existing conservative handling at vscode_import.rs:808). No
   behavior import; document the decision in a code comment.

### Docs

- `docs/src/visual-customization.md` (the project-panel block ~472 and the
  panel block ~592) — document `folder_chevrons`.
- `docs/src/reference/all-settings.md` (~4956 and ~5493) — add the key. If this
  file is generated, regenerate; otherwise edit by hand to match `default.json`.

## Testing

Per CONTRIBUTING.md, non-trivial changes need tests, and UI changes should
consider visual regression tests.

### Unit tests (deterministic, CI-friendly)

Add tests in `crates/project_panel/src/project_panel_tests.rs` driven through
the existing `for_each_visible_entry` / `EntryDetails` path (the same mechanism
`visible_entries_as_strings` at project_panel_tests.rs:10089 already uses). For a
fixture tree with a directory and a file, assert the `(icon, folder_chevron)`
pairing for each of the four setting combinations:

- `folder_icons: true,  folder_chevrons: false` → dir: `icon=folder, folder_chevron=None`.
- `folder_icons: true,  folder_chevrons: true`  → dir: `icon=folder, folder_chevron=Some(chevron)`.
- `folder_icons: false, folder_chevrons: false` → dir: `icon=chevron, folder_chevron=None`.
- `folder_icons: false, folder_chevrons: true`  → dir: `icon=chevron, folder_chevron=None`.
- In every combination, a **file** has `folder_chevron=None`.

Also assert the collapsed/expanded chevron differs (`get_chevron_icon(is_expanded)`)
so expand state is reflected. Settings are set via the test settings store the
same way existing project-panel tests configure `ProjectPanelSettings`.

Optionally extend `visible_entries_as_strings` to surface a marker (e.g. a
leading `⌄`/`›`) when `folder_chevron.is_some()`, enabling concise string-snapshot
assertions; keep this behind the existing helper so current snapshots are
unaffected unless arrows are enabled.

A minimal equivalent settings-resolution test (default is `false`, override
parses to `true`) covers the settings plumbing for `outline_panel` and
`git_panel` without duplicating the full render assertions.

### Visual regression test (macOS, local baselines)

Per `docs/src/development/macos.md`, add a project-panel scenario with
`folder_chevrons` enabled to the visual test runner
(`crates/zed/src/visual_test_runner.rs`, alongside the existing
`project_panel` test at ~line 396). The scenario opens the project panel on the
fixture tree with `folder_icons: true, folder_chevrons: true` and captures
`project_panel_folder_chevrons`. Baselines are gitignored and generated locally:

```sh
# from known-good main, before changes
git checkout upstream/main
UPDATE_BASELINE=1 cargo run -p zed --bin zed_visual_test_runner --features visual-tests
git checkout -
# after changes, regenerate baselines for the intentional new scenario
UPDATE_BASELINE=1 cargo run -p zed --bin zed_visual_test_runner --features visual-tests
```

### Manual verification for the PR

Capture light + dark mode screenshots of the project panel in all four
`folder_icons` × `folder_chevrons` combinations (focus on the "both" mode and
alignment of files vs folders) to attach to the PR, satisfying the CONTRIBUTING
UI checklist and the "attach screenshots" guidance.

## PR hygiene (from CLAUDE.md / CONTRIBUTING.md)

- Branch off `main`; **do not** commit or push until explicitly asked.
- The work targets an existing maintainer-acknowledged issue (#8661, maintainer
  said a setting for both "seems reasonable"), so it fits the "small enhancement
  to make a feature work for more people" category that PRs are welcomed for.
- Title: imperative, no conventional-commit prefix, optional crate scope, e.g.
  `project_panel: Show folder arrows alongside folder icons`.
- One thing only: the `folder_chevrons` setting + its three-panel wiring + docs +
  tests. No unrelated refactors.
- End the PR body with a `Release Notes:` section:
  ```
  Release Notes:

  - Added a `folder_chevrons` setting to show expand/collapse arrows (chevrons) alongside folder icons in the project, outline, and git panels (#8661).
  ```
- Sign-off / CLA: the human contributor must have signed the Zed CLA; the AI
  policy requires a human who understands the change in the loop.

## Risks and mitigations

- **Alignment regressions in non-"both" modes** — mitigated by only adding the
  leading chevron/spacer when `folder_chevron.is_some()` or both-mode is active;
  all other code paths are byte-for-byte unchanged.
- **Three-panel drift** — the same helper logic is duplicated; keep the chevron
  rule (`folder_chevrons || !folder_icons`) identical in all three to avoid
  divergence. Consider a tiny shared helper if it reads cleanly, but do not
  introduce a cross-crate dependency just for this.
- **`all-settings.md` generation** — confirm whether it is generated before
  hand-editing.
