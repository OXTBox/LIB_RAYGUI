# LIB_RAYGUI: the plan

Monday, November 16, 2026. Draft 1, by Fable 5.2, for the owner to correct. TODO A-41.
A plan only: no code yet, and nothing here was compiled or run (section 8 says what was checked).

## 1. What it is for

The owner's brief: "incorporate the raylib GUI library and parrot everything we do in xt3d and
Vibecode3D directly in that gui system, with the raylib white page and colour 0.5 millimetre
Rapidograph technical pens aesthetic."

Three things, in this order of importance:

| Part | What it is |
|---|---|
| **The parrot** | every tool window we make as a `.gui` file (LIB_GUI) also shows inside the xt3d picture, drawn by raygui, with the same names, values and `guiDo` messages. Nobody ports a window by hand: the same file is shown by another face |
| **The binding** | raygui (Ramon Santamaria's one-file immediate-mode GUI for raylib, free zlib licence like raylib) added to the xt3d bridge as `xt3d_Gui*` calls |
| **The look** | "Rapidograph": a white page, outlines only, lines 0.5 mm wide in a few ink colours. It is a LIB_THEME theme, so the xTalk windows can wear it too |

This is the GUI of **iGame3D xRay** (and xRayBox). iGame3D XT, which draws in lines itself, gets
the vector face of LIB_GUI instead; both read the same files.

## 2. What it stands on

- `xt3D_Raylib_Livecode_Bridge`: the C shim (`xt3d_shim.c`), the generator (`gen_xt3d.py`, its
  `EXTRA` list) and the owner's hand-run `build.bat`. **The C part of LIB_RAYGUI lives there**,
  because the shim, the generated file and the `.lcb` must always be rebuilt together. This
  folder holds the xTalk side, the theme mapping, the tests and the notes.
- `raygui.h`, one file, to be put beside `raylib.h` in the bridge. It is not on this machine
  yet (looked for in the bridge folder and `C:\xt3d`); the release that matches raylib 6.2 has to
  be chosen when it is fetched.
- LIB_GUI for the layout array, the widget kinds and the face rules (its section 4).
- LIB_THEME for colours: a new theme file `themes/rapidograph.txt`.

## 3. The two problems this plan had to settle

### 3.1 The mouse

xt3d usually renders with the raylib window **hidden**, into a texture that is copied into an
image in an xTalk window. raygui asks raylib where the mouse is, and a hidden window has no
mouse. Decided: **xTalk feeds the mouse and keys to raygui.**

In the shim, a few lines before raygui is included point raygui's input calls at our own:

```c
static Vector2 xtg_MousePosition(void) { return xtg_fed ? xtg_mouse : GetMousePosition(); }
/* ... the same for button down / pressed / released, wheel, key and char ... */
#define GetMousePosition      xtg_MousePosition
#define IsMouseButtonDown     xtg_MouseButtonDown
/* ... */
#define RAYGUI_IMPLEMENTATION
#include "raygui.h"
```

With the feed off (raylib's own visible window) raygui reads the real mouse as usual. So both
ways of showing xt3d work with one build. This is the first thing to prove (step 1): it has not
been compiled.

### 3.2 Values that raygui changes

raygui hands results back through pointers (`bool *checked`, `int *active`, `char *text`).
The bridge's hard-won rule is that pointers and returned C strings crash. Decided: **each call
takes the value in and gives the new value back**; text comes back through the shim's existing
result buffer. The C side remembers nothing between frames; all state stays in xTalk, in the
LIB_GUI values. That is why these calls are written by hand in the shim (about fifteen), not
generated from `raygui.h` like the raylib ones.

## 4. The calls

### 4.1 Low level (in the bridge; raygui's names, raylib-style places `x, y, width, height`)

| Call | Gives back |
|---|---|
| `xt3d_GuiFeedMouse pX, pY, pDown, pWheel` | - (turns the feed on) |
| `xt3d_GuiFeedKey pKeyCode` / `xt3d_GuiFeedChar pCodepoint` / `xt3d_GuiFeedOff` | - |
| `xt3d_GuiSetStyle pControl, pProperty, pValue` / `xt3d_GuiGetStyle(...)` | - / the value |
| `xt3d_GuiSetFont pFont` | - |
| `xt3d_GuiEnable` / `xt3d_GuiDisable` / `xt3d_GuiLock` / `xt3d_GuiUnlock` | - |
| `xt3d_GuiLabel x, y, w, h, pText` | - |
| `xt3d_GuiButton(x, y, w, h, pText)` | 1 on the frame it is clicked |
| `xt3d_GuiCheckBox(x, y, w, h, pText, pChecked)` | 0 or 1: the new state |
| `xt3d_GuiSlider(x, y, w, h, pLeft, pRight, pValue, pMin, pMax)` | the new number |
| `xt3d_GuiDropdownBox(x, y, w, h, pItems, pActive, pOpen)` | `active,open` |
| `xt3d_GuiListView(x, y, w, h, pItems, pScroll, pActive)` | `scroll,active` |
| `xt3d_GuiTextBox(x, y, w, h, pText, pEditing)` | `changed,editing`; the text is read with the bridge's result reader |
| `xt3d_GuiTabBar(x, y, w, h, pItems, pActive)` | the new active tab |
| `xt3d_GuiGroupBox` / `xt3d_GuiPanel` / `xt3d_GuiLine x, y, w, h, pText` | - |
| `xt3d_GuiWindowBox(x, y, w, h, pTitle)` | 1 when its close box is clicked |

Each starts with the bridge's guard (`if (!IsWindowReady()) return`), so a call with no window
does nothing.

### 4.2 The player (here: `livecode/lib_raygui.livecodescript`)

```
start using stack ".../LIB_RAYGUI/livecode/lib_raygui.livecodescript"

RayGui.Start "rapidograph"                    -- the theme becomes the raygui style
put Gui.Load(tFolder & "/gui/inspector.gui") into tLayoutA
RayGui.Show tLayoutA                          -- becomes LIB_GUI's face for this layout

-- each frame, after the scene is drawn, before the frame is copied to the image:
RayGui.Draw
-- from the image's mouse and key messages (picture-in-a-window way):
RayGui.Mouse pX, pY, pDown
RayGui.Key pKey

on guiDo pName, pValue                        -- the same message the native face sends
end guiDo
if RayGui.Over(pX, pY) is empty then View.Orbit pX, pY   -- the click was not on a widget
```

| Handler | Gives or does |
|---|---|
| `RayGui.Start pTheme` | reads the LIB_THEME file, sets the raygui style, loads the font |
| `RayGui.Show pLayoutA` / `RayGui.Hide` | registers with `Gui.UseFace "raygui"`; from then on `Gui.Get / Set / Show / Enable` work on it |
| `RayGui.Draw` | one bridge call a widget, in the layout's order; sends `guiDo` for whatever was used this frame |
| `RayGui.Mouse` / `RayGui.Key` | pass the xTalk window's mouse and keys to the feed, scaled to the picture |
| `RayGui.Over(pX, pY)` | the name of the widget under that point, or empty (worked out from the places in xTalk, no bridge call) |
| `RayGui.Style pControl, pProperty, pValue` | one style value by raygui's names, for experiments |
| `RayGui.Stop` | feed off, face unregistered |

Menus: raygui has no menu bar. The `menu` kind is drawn as a row of buttons; a pressed one
drops a list (`GuiListView`) under it. Good enough for tool windows; the look is honest raygui.

## 5. The look: Rapidograph

A LIB_THEME file like the others (`key <tab> value`), using raylib's own palette for the inks:

```
# LIB_THEME rapidograph: white page, 0.5 mm technical-pen lines
bg	245,245,245        the page (raylib's RAYWHITE)
panel	245,245,245        no fills: a panel is its outline
text	20,20,24           black ink
border	20,20,24           black ink: every outline
dim	130,130,130        grey ink: disabled, hints
hilite	0,82,172           blue ink: what the mouse is over, what has the keys
accent	230,41,55          red ink: pressed, selected, warnings
ok	0,158,47           green ink: on, filled part of a slider
off	200,200,200        pencil: tracks, unselected tabs
menuBar	245,245,245
menuFace	245,245,245
menuText	20,20,24
pen	0.5                line width in millimetres
```

| Rule of the look | |
|---|---|
| Line width | `pen` mm on the screen: `max(1, round(pen * dpi / 25.4))` pixels. 0.5 mm is 2 px at 100% and 125% scaling, 3 px at 150% |
| No fills | every control's base colour is the page; a control is its outline and its words |
| Ink means state | black = at rest, blue = under the mouse or holding the keys, red = pressed or selected, green = on, grey = cannot be used |
| Lettering | raylib's built-in font to start; later the engine's own stroke letters (see the symbol note below) |

The mapping to raygui's style table (for every control): border colour normal / focused /
pressed / disabled = `border` / `hilite` / `accent` / `dim`; text colour the same four; base
colour all four = `bg`; `BORDER_WIDTH` = the pen in pixels; `DEFAULT BACKGROUND_COLOR` = `bg`;
`LINE_COLOR` = `border`. Set with `xt3d_GuiSetStyle` at `RayGui.Start`; no binary `.rgs` style
files.

**The symbol problem (found while planning).** Our buttons use Unicode symbols (`▶ ⚙ ☑`).
raylib's built-in font has only plain letters, so raygui would draw `?`. Step 2 therefore loads
a font that has them (DejaVu Sans, free to ship, into `Data/Fonts`) with just the symbols the
layouts use. The longer answer is the owner's "binary strings": a symbol kept as a 1-bit glyph
(LIB_BIT) draws the same in every face with no font at all; LIB_GUI's `glyph=` option is where
that will plug in.

## 6. Who owns what (the same words in all four plans)

| Library | Owns | Never does |
|---|---|---|
| **LIB_USERS** | who is at the controls: the roster, the current user, the wording of window titles, the stamp on log lines, the "may I" words | draws nothing, opens no windows |
| **LIB_THEME** (exists) | colours and line weight, as `themes/<name>.txt` | no layout, no widgets |
| **LIB_GUI** | what a window contains: the `.gui` layout file, the widget kinds, the calls `Gui.*`, the message `guiDo`, the native face (real xTalk controls) and the GUI Builder app | no colours of its own (asks LIB_THEME), no content of its own (no launcher list, no user names) |
| **LIB_RAYGUI** | one more face for the same `.gui` file: raygui drawn inside xt3d (iGame3D xRay); the `xt3d_Gui*` calls; how a theme becomes a raygui style | no layout format and no widget kinds of its own |
| **LIB_TOOLBAR** | one app: the floating launcher strip, and its content file `*.toolbar` (what the buttons are, what they open) | no widget code (its window is built by LIB_GUI); it does not decide who may press what (asks LIB_USERS) |

The rules that follow from the table:

1. A window is laid out once, as a `.gui` file, and shown by any face. Nobody ports a window by hand.
2. A new widget kind is added to LIB_GUI first (the format and the native face), then to the other faces.
3. Colour is LIB_THEME's. Who is LIB_USERS's. What is on the launcher is LIB_TOOLBAR's. How it is drawn in raylib is LIB_RAYGUI's.
4. A row of tools inside an app's own window (iGame3D XT's 26 tools) is ordinary `.gui` buttons belonging to that app. LIB_TOOLBAR is only the floating strip that sits outside a window: the projects launcher, or an app's own tools floated the way the 2007 iGame3D main strip was.
5. XTJSON stays the full, lossless record of a stack. A `.gui` file is the small design a person or Spark can write by hand. There is one tool, one way: XTJSON harvest to `.gui`.
6. No script builds a window title by hand: it asks `Users.Title`.

Build order across the four: (1) LIB_USERS core, (2) LIB_GUI stage 1: the format and the native face,
(3) LIB_TOOLBAR, the first real customer, (4) LIB_RAYGUI: one button proven in the bench, then the
player, (5) the GUI Builder app, (6) the vector face for iGame3D XT.

Where LIB_RAYGUI meets the existing libraries: the bridge keeps the C and its build; the xt3d
bench gets one check line for step 1; the Game Player on xRay calls `RayGui.Draw` in its frame
loop only when a layout is showing. The engine contract is not touched: a game still calls only
what the contract lists, so raygui is for tools and the editor until the owner decides otherwise
(LIB_GUI question 4). S-4's "port to raygui" is this library; nothing is ported by hand.

## 7. What is built first

The owner runs every Windows build by hand, so the plan needs **two rebuilds, not fifteen**.

| Step | What | Who | Done when |
|---|---|---|---|
| 0 | fetch `raygui.h` for raylib 6.2 into the bridge; write the feed and three calls (`GuiFeedMouse`, `GuiButton`, `GuiLabel`, `GuiSetStyle`) | Opus | `python3 gen_xt3d.py` lists them; `gcc -std=c99 -fsyntax-only xt3d_shim.c` is clean in WSL |
| 1 | **rebuild 1** (`build.bat`), one bench check: a button drawn into the picture, a fed click answers 1 | owner + Opus | the check passes and the owner sees a button in the bench |
| 2 | the stub library and the player for `label button toggle line panel`; `rapidograph.txt`; the symbol font | Opus | headless tests pass (section 8) |
| 3 | the rest of the calls in section 4.1, all at once; **rebuild 2** | Opus + owner | bench checks for each call |
| 4 | the full player; the toolbar strip and the Status Browser layout shown in the xt3d bench | Opus | the owner compares them with the xTalk windows and judges the look |
| 5 | only if needed: draw a whole layout in one C call (if a big window drops the frame rate); write raygui's own `.rgl` layout files for people who use that tool | later | - |

If step 1 fails (the redirect does not compile against the real raygui, or the fed mouse is
ignored), the fallback is to draw the widgets from xTalk with the 2D calls xt3d already has
(`xt3d_DrawRectangleLinesEx`, `xt3d_DrawText`): no C at all, the same player handlers, the same
look, but not raygui. Steps 2 and 4 are written so that only the bottom layer would change.

## 8. How it is tested headless, and what was checked for this plan

Checked today, by reading only: raygui is not in the bridge yet; the bridge has the 2D drawing,
text, scissor and mouse calls the fallback would need; the generator's `EXTRA` list and the
shim's result buffer work the way section 3.2 relies on. **Nothing was compiled or run.**

| Test | Where | What it shows |
|---|---|---|
| Generator and syntax | WSL: `gen_xt3d.py`, `gcc -fsyntax-only` with `raygui.h` beside the shim | the new calls exist in the `.lcb` and the manifest; the redirect compiles |
| The player against a stub | `OXT-BEYOND/tests/oxt_headless.py`, body `tests/t_raygui_player.txt`, with `tests/xt3d_gui_stubs.livecodescript` standing in for the bridge (each `xt3d_Gui*` records its arguments; the test scripts what each returns) | for a layout: the right calls, in order, with the right places and words; a scripted click gives exactly one `guiDo`; `Gui.Set` changes what the next frame passes; an unknown kind is skipped with a problem line; `RayGui.Over` names the widget under a point |
| The style | same runner | `RayGui.Start "rapidograph"` makes the expected `xt3d_GuiSetStyle` calls: base = page for all four states, pen = 2 px at 96 dpi and 3 px at 144 |
| The real thing | the xt3d bench on Windows (`tools/bench checks`) after each rebuild | one line a call: drawn without a crash, a fed click or drag gives the expected answer; a pixel read back from the picture is the page colour at a corner and ink on a border |
| The look | the owner's eyes | whether it reads as a technical drawing. The AI does not open the screenshots |

## 9. Options rejected

| Option | Why not |
|---|---|
| Draw the widgets from xTalk with xt3d's 2D calls, no raygui | works today with no rebuild, but it is not the raylib GUI library the owner asked for, and costs several bridge calls a widget each frame. Kept as the fallback |
| Generate every raygui function from `raygui.h`, like raylib's | its pointer in-out parameters are exactly what crashes the bridge; fifteen hand-written calls cover the fourteen widget kinds |
| Let raygui read raylib's own mouse | only right in raylib's visible window; wrong in the picture-in-a-window way that the bench and the Game Player use |
| Keep widget state on the C side | two places to hold a value; closing and reopening the view would lose it |
| Parse `.gui` files in C | a second reader to keep in step with LIB_GUI's; only worth it if speed demands (step 5) |
| Dear ImGui | C++, needs a C wrapper and a bigger build; raygui is one C file from raylib's own author |
| Binary `.rgs` style files | not plain text; the look belongs in a LIB_THEME file every face reads |
| A raygui-only window format | then every window exists twice. One `.gui` file, many faces |
| Putting the C in this folder | the bridge's three files must build together; a second C home is how a new `.lcb` meets an old DLL |

## 10. Questions only the owner can answer

| # | Question | Default taken |
|---|---|---|
| 1 | Two rebuilds of the bridge by hand (step 1 and step 3): acceptable? | yes; each with a one-page "what to click" note |
| 2 | The page: raylib's off-white (245,245,245) or pure white? | raylib's off-white |
| 3 | The inks and what they mean (black rest, blue hover, red pressed, green on) | as in section 5; you change `rapidograph.txt` |
| 4 | 0.5 mm measured on the screen (2 px at normal scaling), or simply 1 px? | measured: 2 px |
| 5 | Should the xTalk windows wear Rapidograph too, or keep the dark theme? | keep dark as the default; Rapidograph is offered in Theme Studio |
| 6 | Shipping a font file for the symbols (DejaVu Sans, free licence) until glyphs come from LIB_BIT: fine? | yes |
| 7 | Does "parrot everything" include games' own screens (HUD, menus), or tools only? | tools and the editor only, for now |
