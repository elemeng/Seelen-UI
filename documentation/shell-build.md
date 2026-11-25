# Shell build (TopBar + Dock)

This fork ships a trimmed Seelen UI: only the **TopBar** (`@seelen/fancy-toolbar`) and
the **Dock** (`@seelen/weg`) are kept, plus the overlays they depend on.

## What stays

| Part | Id | Frontend | Bundled widget | Backend |
| --- | --- | --- | --- | --- |
| TopBar | `@seelen/fancy-toolbar` | `src/ui/svelte/fancy-toolbar/` | `src/static/widgets/toolbar/` | `src/background/widgets/toolbar/` |
| Dock | `@seelen/weg` | `src/ui/svelte/weg/` | `src/static/widgets/weg/` | `src/background/widgets/weg/` |
| Dock hover preview | `@seelen/weg-preview` | `src/ui/svelte/weg-preview/` | `src/static/widgets/weg-preview/` | shared |
| Right-click menus | `@seelen/context-menu` | `src/ui/svelte/context-menu/` | `src/static/widgets/context-menu/` | shared |
| Tooltips | `@seelen/tooltip` | `src/ui/svelte/tooltip/` | `src/static/widgets/tooltip/` | shared |
| Dialogs | `@seelen/dialog` | `src/ui/react/dialog/` | `src/static/widgets/dialog/` | shared |
| Runtime shell | — | `src/ui/vanilla/` | — | — |

`context-menu`, `tooltip`, `dialog` and `weg-preview` are internal overlays: they are
force-enabled in `libs/core/src/state/settings/by_widget.rs` (`SYSTEM_WIDGET_IDS`) and
are not user-toggleable.

Everything else was removed: window manager, wallpaper layer, app menu / launcher,
all flyouts and popups (`system-tray`, `quick-settings`, `calendar`, `media`, `network`,
`bluetooth`, `notifications`, `user-menu`, `power-menu`), task switcher, workspaces
viewer, icon editor, keyboard selector, settings panel, and the `wm_*` layout plugins.

## How the build is trimmed

1. `libs/core/src/state/settings/by_widget.rs` — `is_enabled` no longer defaults
   `@seelen/*` widgets to enabled; only the ids above are reported as enabled.
   `@seelen/window-manager` and `@seelen/wallpaper-manager` are hard-disabled.
2. `libs/core/src/state/settings/mod.rs` — `SeelenWallSettings::default()` has
   `enabled: false`, so a legacy config can not resurrect the wallpaper layer.
3. `scripts/build/steps/discover.ts` — `ALLOWED_WIDGET_FOLDERS` whitelists the apps
   that get bundled; `vanilla` (shared webview runtime) is never filtered.
4. `src/static/themes/default/metadata.yml` — only styles of the shipped widgets are
   included. **A `!include` of a deleted style file breaks theme loading**, so keep
   this file in sync when adding or removing a widget.

The backend modules of the removed widgets (`src/background/widgets/{window_manager,
wallpaper_manager,popups,task_switcher}`) are still compiled but never instantiate a
webview, because `is_enabled` returns `false` for them. Removing them touches
`libs/core` command declarations, `libs/slu-ipc` CLI variants and handler call sites,
so it is intentionally left as a separate step.

## Re-adding a component

1. Restore its folder(s): `src/ui/<framework>/<id>/`, `src/static/widgets/<id>/`,
   and its theme style (if any).
2. Add the folder name to `ALLOWED_WIDGET_FOLDERS` in `scripts/build/steps/discover.ts`.
3. Allow it in `is_enabled` (`libs/core/src/state/settings/by_widget.rs`) — either
   map it to its settings field or add it to `SYSTEM_WIDGET_IDS` for overlays.
4. Re-add its `!include` line in `src/static/themes/default/metadata.yml`.

To get the settings panel back for debugging, restore `src/ui/react/settings/`,
add `settings` to `ALLOWED_WIDGET_FOLDERS`, and drop the early `@seelen/settings`
return in `is_enabled`.

## Verification

```bash
npx deno@2 task build          # only needed when libs/core changes (run in libs/core)
npm run build:ui               # expect: React 1, Svelte 5, Vanilla 3
npx tsc --noEmit
npx svelte-check --tsconfig ./tsconfig.json
cargo check
npm run dev                    # visual check: one TopBar, one Dock, nothing else
```

Visual checklist: only TopBar + Dock appear; Dock lists open windows and switches
them; hovering a Dock item opens the preview; right-clicking a Dock item opens the
menu; the Windows taskbar is restored on exit (`src/service/shutdown.rs` is untouched).

## Expected dead interactions

Clicking a Dock/TopBar element whose widget was removed does nothing (the trigger
returns "Can't trigger a disabled widget"):

- Dock: start-menu tile (falls back to the native Windows start menu), user menu,
  icon editor, settings.
- TopBar: tray popup, quick settings, calendar, media, network/bluetooth/notifications,
  keyboard selector.

Wallpapers are handled by Windows itself now; the "settings" entry is gone because the
settings panel is not bundled.
