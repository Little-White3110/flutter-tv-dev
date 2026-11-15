# dpad v3 — Flutter TV focus reference

Package: `fluttercandies/dpad` ^3.0.0 (MIT). Android TV / Google TV / Amazon Fire TV / Apple TV
(embedded) / Desktop / Web. Requires Flutter >= 3.24.

Why not plain Flutter focus: Flutter's built-in directional focus picks the **geometrically closest**
widget. On real TV layouts focus jumps lanes between sidebar and content, skips rows, escapes
carousels, and forgets positions. dpad replaces the traversal policy with Android `FocusFinder` /
Leanback semantics: **per-region memory** and **beam-based directional traversal**. It is built on
Flutter's own `FocusTraversalPolicy` + `Shortcuts` + `Actions`, so plain `Focus` widgets, buttons,
`TextField`s and dialog `FocusScope`s interoperate natively.

## Install

```yaml
dependencies:
  dpad: ^3.0.0
```

## Root setup

```dart
import 'package:dpad/dpad.dart';

MaterialApp(
  builder: Dpad.wrap(
    debugOverlay: kDebugMode,                  // on-screen focus inspector
    theme: const DpadThemeData(effects: [DpadGlowEffect()], scrollPadding: 64),
    onBack: () { /* returns bool; true = consumed */ },
    onMenu: () { /* menu / context key */ },
    onFocusChange: (node) { /* app-wide tick sound */ },
    shortcuts: { LogicalKeyboardKey.keyS: () => openSearch() },
    keySet: const DpadKeySet().copyWith(select: [LogicalKeyboardKey.f1, ...DpadKeySet.defaultSelect]),
  ),
  home: const HomePage(),
)
```

`Dpad.wrap()` covers every route, dialog and sheet. All key handling and directional navigation
automatically stand down while a `TextField` is focused.

## Widgets

### DpadFocusable — the single focus stop

```dart
DpadFocusable(
  autofocus: true,                    // ONE per screen, on the primary action
  onSelect: () => play(),             // DPAD_CENTER (or tap)
  onLongSelect: () => showSheet(),    // held center
  onFocusChange: (focused) => setState(() => hovered = focused),
  onDirection: (d) {                  // sliders etc: consume left/right
    if (d == TraversalDirection.right) { volumeUp(); return true; }
    if (d == TraversalDirection.left)  { volumeDown(); return true; }
    return false;                     // up/down keep navigating
  },
  effects: const [DpadScaleEffect(scale: 1.1), DpadGlowEffect(color: Colors.amber)],
  child: card,
)
```

Full control, including the **pressed** state of the select key:

```dart
DpadFocusable(
  builder: (context, state, child) => AnimatedScale(
    scale: state.pressed ? 0.97 : (state.focused ? 1.06 : 1.0),
    duration: const Duration(milliseconds: 120),
    child: child,
  ),
  onSelect: play,
  child: card,
)
```

Effects are const-able and composable; first in the list is the outermost wrapper:
`DpadScaleEffect` (with pressed push-down), `DpadBorderEffect` (no layout shift), `DpadGlowEffect`,
`DpadElevationEffect`, `DpadOpacityEffect`, `DpadTintEffect`, `DpadCustomEffect`.

`excludeChildFocus` defaults to `true` — wrapping a button never creates two D-pad stops. Set it to
`false` only when the child must own focus (a `TextField`, which should instead simply not be wrapped).

### DpadRegion — sections with memory

```dart
Row(children: [
  DpadRegion(
    verticalEdge: DpadEdgeBehavior.stop,      // sidebar never drops off screen
    child: SidebarColumn(...),
  ),
  Expanded(child: ListView(children: [
    for (final row in rows)
      DpadRegion(
        memoryKey: 'home/${row.id}',          // survives section-switcher rebuilds
        horizontalEdge: DpadEdgeBehavior.wrap,
        enter: DpadEnterBehavior.restore,
        onEdge: (dir) => playBumpSound(),
        onFocusChange: (inside) => setState(() => highlighted = inside),
        child: SizedBox(height: 220, child: horizontalList(row.cards)),
      ),
  ])),
])
```

**Enter behavior** (where focus lands when entering a region):

| Value | Landing item |
|---|---|
| `restore` (default) | last focused item, falls back to position-nearest -> `entry` -> geometric target |
| `entry` | the item marked `DpadFocusable(entry: true)` |
| `nearest` | geometric nearest (plain Flutter behavior) |

**Edge behavior** (per axis):

| Value | Effect |
|---|---|
| `leave` (default) | focus continues to the best target outside |
| `stop` | key consumed, focus stays, `onEdge` fires (bump animation/sound) |
| `wrap` | wraps to the opposite side — carousels |

### Programmatic control

```dart
final dpad = Dpad.of(context);
dpad.moveDown();
dpad.move(TraversalDirection.left);
dpad.select();
dpad.requestFocus(searchNode);   // jump focus, updates region memory
dpad.ensureVisible();            // padded scroll-into-view
dpad.focused;                    // currently focused node
```

## Layout rule that prevents clipped focus effects

Put the padding **inside** the scroll view, and give the row headroom for the effect:

```dart
SizedBox(
  height: cardHeight + 32,
  child: ListView.builder(
    scrollDirection: Axis.horizontal,
    padding: const EdgeInsets.symmetric(horizontal: 36, vertical: 16),
    ...
  ),
)
```

## Debugging

`debugOverlay: true` outlines the focused node with its label, region and geometry. Focus bugs are
invisible from across the room — this is the primary diagnostic, enable it during development.

Desktop targets (macOS/Windows/Linux) map arrow keys to navigation, so most focus work can be
iterated without an emulator. Verify final behavior on a TV AVD or device.

## Migration notes (2.x -> 3.0)

| 2.x | 3.0 |
|---|---|
| `DpadNavigator(child: app)` | `MaterialApp(builder: Dpad.wrap())` |
| `Dpad.navigateUp(context)` | `Dpad.of(context).moveUp()` |
| `builder: (c, focused, child)` | `builder: (c, state, child)` with `state.focused` / `state.pressed` |
| `FocusEffects.glow()` closures | `effects: const [DpadGlowEffect()]` const classes |
| global history stack + `FocusMemoryOptions` | per-`DpadRegion` memory (`enter: restore` default) |
| `RegionNavigationRule` tables | `DpadRegion(enter:, horizontalEdge:, verticalEdge:)` |
| `DpadFocusable(region:, isEntryPoint:)` | wrap in `DpadRegion`, mark item `entry: true` |
| `onBackPressed` / `onMenuPressed` | `onBack` (returns `bool`) / `onMenu` |
