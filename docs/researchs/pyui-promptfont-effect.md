# PyUI: does bundled `promptfont.ttf` have any effect?

## Question

Does `App/PyUI/main-ui/utils/promptfont.ttf` ever actually get loaded/rendered
on a real spruceOS path, or is it dead weight?

Terminology follows `GLOSSARY.md:15-25` (**font fallback**, **tofu**, **run**)
and the sibling research docs
`docs/researchs/pyui-keyboard-font-fallback.md` /
`docs/researchs/pyui-tofu-sweep.md`.

## Verdict

**(A) Dead weight** — high confidence.

`promptfont.ttf` is never opened on any real spruceOS path, for two
*independent* reasons (either one alone is sufficient):

1. **The glyph-coverage font-fallback mechanism itself is dormant.** The
   fallback `FontCache` is lazy and is only ever *accessed* behind the flag
   `allow_fallback_font_for_missing_glyphs` — and an AST scan of every `.py`
   under `App/` found **no call site** that passes that flag `True`
   (positionally or by keyword). The fallback font file is therefore never
   opened, whichever path it points at.
2. **Even the path default is always overridden.** On device, `launch.sh`
   always passes `-pyUiConfig /mnt/SDCARD/App/PyUI/py-ui-config.json`, which
   ships `fallbackFontPath → Themes/SPRUCE/nunwen.ttf`; on desktop,
   `prepare_pyui_config()` derives its config from the same file. So
   `PyUiConfig.get_fallback_font()` returns `nunwen.ttf`, never the
   `promptfont.ttf` default.

**Strongest single piece of evidence:** the AST scan result — the only two
consumers of `cls.fallback_fonts[...]` are gated on
`allow_fallback_font_for_missing_glyphs`
(`App/PyUI/main-ui/display/display.py:735-739` and `:1404-1405`), and no
caller anywhere under `App/` passes that flag `True`
(`display.py:727` defaults it to `False`; exhaustive AST scan of
`render_text`/`get_text_dimensions` calls: 0 hits with the flag or with
enough positional args to reach it).

## Evidence

### 1. Config default vs shipped override

- **Claim:** `promptfont.ttf` is only a *default value* when the config lacks
  a `fallbackFontPath` key.
  → `App/PyUI/main-ui/utils/py_ui_config.py:98-100`:
  `cls._data.get("fallbackFontPath", cls._current_dir / "promptfont.ttf")`.
- **Claim:** the shipped spruce config always carries the key.
  → `App/PyUI/py-ui-config.json:33`: `"fallbackFontPath":
  "/mnt/SDCARD/Themes/SPRUCE/nunwen.ttf"`.
- **Claim:** config load replaces `_data` wholesale, so a successfully loaded
  file always wins over the default.
  → `App/PyUI/main-ui/utils/py_ui_config.py:46-56` (`_read_from_file`;
  on `FileNotFoundError`/bad JSON it falls back to `{}`, see below).
- **Claim:** the upstream stock config `main-ui/config.json` has *no*
  `fallbackFontPath` (so the promptfont default applies only if that file is
  copied) — `App/PyUI/main-ui/config.json:1-6`. It is copied only when the
  target config is missing/empty, via
  `App/PyUI/main-ui/mainui.py:97-102` →
  `App/PyUI/main-ui/utils/config_copier.py:12-31`.
- **Claim:** a settings restore cannot drop the key: `spruceRestore` *merges*
  the backup over the config with `jq '.[0] * .[1]'`
  → `App/spruceRestore/spruceRestore.sh:65-73`.

### 2. `requires_fallback_font` — runtime coverage probe, not a hardcoded set

- `App/PyUI/main-ui/display/display.py:713-722`: returns `True` iff the
  **primary** font of that `FontPurpose` lacks at least one glyph of `text`.
  It probes the actual loaded `TTF_Font` via `_font_has_glyph`
  (`display.py:693-711`), which calls `TTF_GlyphIsProvided32` (or the 16-bit
  `TTF_GlyphIsProvided` on old SDL_ttf — chosen at `display.py:166-170`) and
  caches per `(font pointer, codepoint)`.
- `font_purpose` (Cheevos side) just maps Cheevos text roles onto PyUI's
  `FontPurpose` members: `App/Cheevos/cheevos/ui/pyui/text.py:41-62`.
- Cheevos uses `requires_fallback_font` **only to strip/substitute characters**
  (`displayable()` at `text.py:82-112`, `_has_glyph` at `text.py:65-79`) —
  i.e. it deliberately avoids drawing missing glyphs instead of routing them
  to the fallback font.

### 3. Every use of the fallback flag — exhaustive

`allow_fallback_font_for_missing_glyphs` occurs **only** in
`App/PyUI/main-ui/display/display.py`:

| Site | Value |
|---|---|
| `display.py:727` (`render_text` signature) | default `False` |
| `display.py:729-730` (frame-draw log, replay compares only) | — |
| `display.py:735-739` (render branch) | gated on the flag |
| `display.py:1403-1405` (`get_text_dimensions`) | default `False`, gated |

An AST scan of every `.py` under `App/` (including `App/Cheevos` and
`App/PyUI`) found **zero** callers passing the flag — neither as keyword nor
with enough positional arguments to reach it (`render_text` 10th positional,
`get_text_dimensions` 3rd). Other render entry points (on-screen keyboard:
`App/PyUI/main-ui/display/on_screen_keyboard.py:107-112`; Wi-Fi/Bluetooth ✓:
`App/PyUI/main-ui/menus/settings/wifi_menu.py:113`,
`bluetooth_menu.py:125`) call `Display.render_text` without the flag.

### 4. `Theme.get_fallback_font` is a *different* mechanism

- `App/PyUI/main-ui/themes/theme.py:572-578`: `Theme.get_fallback_font()`
  returns the language font (`Language.get_font_for_purpose(FontPurpose.LIST)`)
  or `main-ui/themes/font.ttf`. It is the **file-level** fallback used by
  `Theme.get_font` when a theme's font file doesn't exist
  (`theme.py:564-570`) — per `docs/researchs/pyui-keyboard-font-fallback.md:27-33`,
  "fallback hiện tại chỉ ở mức file… không ở mức glyph".
- The display's **glyph-level** fallback uses the *other* one:
  `Display.init_fonts` → `cls.fallback_fonts = FontCache(PyUiConfig.get_fallback_font())`
  (`display.py:272-277`). So the two `get_fallback_font`s coexist without
  interacting; `theme.py` never consults `PyUiConfig.get_fallback_font`, and
  `display.py` never consults `Theme.get_fallback_font` for the fallback cache.
- Note (consistent with `docs/researchs/pyui-tofu-sweep.md:77-84`): no
  `lang/*.json` declares a `fonts` key, so `Theme.get_font`'s language
  override (`theme.py:529-531`) is itself dormant; the file-level fallback
  lands on `themes/font.ttf`, which itself lacks `↵` U+21B5
  (`docs/researchs/pyui-tofu-sweep.md:109`, confirmed by measurement below).

### 5. Laziness: the fallback file is never even opened

- `FontCache` stores the path and loads a font only on first dict access
  (`display.py:43-53`, `__missing__` → `Display._load_font(purpose, self.font_path)`).
- The only accesses are `cls.fallback_fonts[purpose]` at `display.py:738` and
  `:1405`, both behind the always-`False` flag (see §3). Therefore
  **no fallback font file — neither nunwen nor promptfont — is ever opened**,
  and `Display.deinit_fonts` (`display.py:395-397`) iterates an always-empty
  cache.

## Runtime path trace

**On device (spruceOS):**

1. `App/PyUI/launch.sh:41` (and 8 more invocations: `:85, :119, :192, :212,
   :257, :308, :336, :371`) starts PyUI with
   `-pyUiConfig /mnt/SDCARD/App/PyUI/py-ui-config.json`.
2. `App/PyUI/main-ui/mainui.py:179-180`: `verify_config_exists(args.pyUiConfig)`
   (copies stock `main-ui/config.json` only if the file is missing —
   `config_copier.py:12-31`), then `PyUiConfig.init(args.pyUiConfig)`.
   (The argparse default `/mnt/SDCARD/Saves/pyui-config.json` at
   `mainui.py:36` is never used by spruce's launcher.)
3. `py_ui_config.py:46-49` loads the JSON; `py-ui-config.json:33` supplies
   `fallbackFontPath → /mnt/SDCARD/Themes/SPRUCE/nunwen.ttf`.
4. `Display.init_fonts` (`display.py:277`): `fallback_fonts = FontCache(nunwen)`.
5. Every `render_text` call defaults the flag to `False` (`display.py:727`) →
   `display.py:735` always takes the primary-font branch. Fallback cache never
   accessed; promptfont never referenced.

**On desktop:** `App/PyUI/main-ui/desktop.py:122-136,167` builds
`<data>/Saves/pyui-config.json` from `App/PyUI/py-ui-config.json` (dropping
only `*Cmd` keys, `desktop.py:94-100`; `fallbackFontPath` survives and is
rewritten to the checkout's `Themes/SPRUCE/nunwen.ttf` by `on_desktop`,
`desktop.py:103-109`). Same result.

**Degenerate cases** (still no effect): if the device config were deleted,
`config_copier.py` would copy the stock `main-ui/config.json` (no
`fallbackFontPath`), `get_fallback_font()` would return the promptfont
default — but the flag is still always `False`, so even then promptfont is
never opened.

## Glyph coverage findings

Measured with a pure-stdlib `struct` cmap parser (same method as
`docs/researchs/pyui-tofu-sweep.md`, §1 step 4; no fontTools/PIL available in
`.venv/` or system Python — verified). Script: `/tmp/opencode/cmap_check.py`
(local-only, not committed).

| Font | codepoints | `⇪`21EA | `↑`2191 | `←`2190 | `↵`21B5 | `→`2192 | `↓`2193 | `✓`2713 | `·`00B7 | `█`2588 | CJK/Кириллица/emoji |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `App/PyUI/main-ui/utils/promptfont.ttf` (name record: **"PromptFont"**) | 1,339 | Y | Y | Y | Y | Y | Y | **.** | Y | Y | none (あ/中/Б/α/😀 all missing) |
| `Themes/SPRUCE/nunwen.ttf` (name record: "gruvbox") | 36,012 | Y | Y | Y | Y | Y | Y | Y | Y | Y | CJK/Kana/Hangul/Cyrillic/Greek yes; Thai/emoji no |
| `App/PyUI/main-ui/themes/font.ttf` (Inconsolata Medium, file-level fallback) | 882 | Y | Y | Y | **.** | Y | Y | Y | Y | Y | none |

Reading:

- promptfont covers exactly the keyboard/Konami symbol set
  (`⇪ ↑ ← ↵ → ↓`) the tofu sweep catalogued as at-risk
  (`docs/researchs/pyui-tofu-sweep.md:22-50`) — that is clearly *why* an
  upstream-style symbol font was bundled (also matching ticket
  `.scratch/pyui-symbol-fallback/issues/01-bundle-promptfont.md:3`).
- But it **lacks `✓` U+2713**, the checkmark rendered by Wi-Fi/Bluetooth menus
  (`wifi_menu.py:113`, `bluetooth_menu.py:125`), and lacks all CJK/Cyrillic/
  Greek/emoji — so it could never have been a general fallback anyway;
  `nunwen.ttf` strictly dominates it on every UI-relevant codepoint probed.
- Consistency check with `docs/researchs/pyui-tofu-sweep.md:110`: nunwen row
  matches (all-Y except Thai/emoji); `themes/font.ttf` matches its row at
  `:109` (missing `↵` only).

## History / intent

- `git log --all -- App/PyUI/main-ui/utils/promptfont.ttf` shows exactly **one**
  commit: `d08baf8ff` "Log times in mS and only to one decimal"
  (2026-10-12 12:50:09) — it added `promptfont.ttf` (Bin 0 → 255,828 bytes)
  together with `time_logger.py` changes. The commit message doesn't mention
  the font; it came in as upstream-PyUI-style plumbing.
- 13 seconds later, `37bc2dfc1` "Add config option for fallback font"
  (12:50:22) added both `get_fallback_font()` (default = promptfont) **and**
  `"fallbackFontPath": ".../nunwen.ttf"` to the shipped config — i.e. spruce
  introduced the mechanism already pointed at nunwen, with promptfont only as
  the code-level default. `faae06048` (12:51:40) fixed the missing `cls.` on
  `_current_dir` in that commit.
- The glyph-coverage fallback machinery in `display.py`
  (`fallback_fonts`, the flag) predates all of this: it arrived with upstream
  sync `cb3659dc8` (2026-10-12 11:14) and was touched by `67f5943ce`. In this
  branch it was never activated by any caller.
- The **symbol-fallback work superseded promptfont deliberately**:
  - `feat/pyui-symbol-fallback` (branch tip, **not merged** — ancestor check
    against HEAD fails): ticket 01's bundle commit `2581bbaf5` "PyUI: bundle
    promptfont.ttf (OFL) as symbol font" lives only on
    `backup/pyui-symbol-fallback`; at the branch tip the file,
    `Licenses/fonts/promptfont.txt` and `menus/common/symbols.py` are **gone**,
    replaced by a per-glyph fallback chain whose stock candidates are all
    `nunwen.ttf` paths (`font_fallback.py` on that branch:
    `DEFAULT_STOCK_CANDIDATES`, gated by `use_fallback_fonts_for_missing_glyphs`,
    commit `974f28d31`).
  - `feat/pyui-keyboard-font-fallback` (also not merged) implements its own
    run-based per-glyph fallback (`display/font_fallback.py`:
    `split_fallback_runs`) — again no promptfont.
  - Neither branch tip contains any `promptfont.ttf` (verified with
    `git ls-tree -r`).

## Recommendation

1. **Remove `App/PyUI/main-ui/utils/promptfont.ttf`** (verdict A). It cannot
   load on any real path, and both open feature branches already replaced the
   idea with nunwen-based per-glyph fallback. While at it, either delete the
   dead default in `py_ui_config.py:100` or point it at
   `Themes/SPRUCE/nunwen.ttf` so the default and the shipped config agree.
2. **Licence note:** promptfont is SIL OFL 1.1 (Shinmera / Yukari Hafner,
   based on Xolonium — per `2581bbaf5`'s message and
   `.scratch/pyui-symbol-fallback/issues/01-bundle-promptfont.md:14`). On
   *this* branch no OFL text or attribution ships for it
   (`Licenses/fonts/` contains only `bevietnampro.txt`; repo-wide grep for
   "promptfont|xolonium|shinmera" in `Licenses/` finds nothing), so the
   current tree arguably *violates* OFL's distribution requirement. Removing
   the file **fixes** that hygiene issue; OFL imposes no obligation to keep or
   copyleft-adjacent files. If the symbol-fallback branches later re-bundle
   it, carry `Licenses/fonts/promptfont.txt` as ticket 01 specified.
3. If the dormant `allow_fallback_font_for_missing_glyphs` mechanism is ever
   activated, prefer nunwen (full UI coverage, §Glyph coverage) over promptfont;
   the repo's own research (`docs/researchs/pyui-tofu-sweep.md:152-163`, S1)
   already recommends nunwen for per-purpose fallback.

*Investigation only; no code or config was modified. Working tree left clean.*
