# LIB_RAYGUI

Monday, November 16, 2026. Planned, not built yet: read `PLAN.md`.

raygui, raylib's own small GUI library, inside xt3d (iGame3D xRay): every tool window we lay
out once as a `.gui` file (LIB_GUI) also shows inside the 3D picture, drawn by raygui, with the
same names, values and messages. The look is "Rapidograph": a white page, outlines only, lines
0.5 mm wide in a few ink colours. xTalk, from the HyperCard family, stays the scripting language.

| It owns | It leaves to |
|---|---|
| the `xt3d_Gui*` calls, the player (`RayGui.*`), how a theme becomes a raygui style | the layout file and widget kinds: LIB_GUI. Colours: LIB_THEME (`rapidograph.txt`). The C build: the xt3d bridge folder |

## Licence

MIT (see `LICENSE`), Copyright (c) 2026 xAction.
