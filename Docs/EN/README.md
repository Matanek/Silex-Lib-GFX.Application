# GFX.Application

`GFX.Application` adds composable application contexts to the canonical
module. `Application` keeps the process's single main loop;
`Application.Bundle` owns a local set of Plugins, systems, and resources driven
by that loop.

A Bundle may represent temporary window content, a complete window, an
independent tool, or another sub-application. It does not mean a graphics
scene: a future scene system may build on Bundles without conflating the two
concepts.

## Plugins and resources

A Bundle uses exactly the same `Application.Plugin` protocol as an
Application:

```sx
var bundle = Application.Bundle(application)
    ..add_plugin(Plugins.Grid2D())
    ..add_resource(Document())
    ..add_system(Application.Schedule.update, update_document)
```

There is no `ScenePlugin`, nor a `Content` variant of the 2D, 3D, or Animation
Plugins. A Plugin installed on a Bundle receives an ordinary `&Application`
representing its local context.

Local resources shadow parent resources of the same type. A missing read falls
back to the Application without copying the value; removing or finalizing the
Bundle never changes its parent.

A Plugin that provides a durable capability may call
`application.extend_bundles(self)`. The same Plugin is then prepared in every
Bundle. `Scene2D`, `Scene3D`, and `Animation` use this mechanism: when installed
globally, they retain their shared capabilities and automatically connect their
local systems to Bundles. A Plugin such as `Grid2D` may also add `Scene2D`
directly inside its own Bundle. Users therefore have no second Plugin type to
discover or declare.

## BundleManager

`Plugins.BundleManager` mounts, drives, and finalizes Bundles at safe main-loop
boundaries.

- `add(bundle)` adds a Bundle without removing active ones, so multiple windows
  or tools can progress together.
- `remove(bundle)` removes one specific Bundle.
- `replace(bundle)` atomically replaces every active Bundle; this path suits a
  splash screen, page, or game scene.
- `create()` creates a Bundle associated with the Application after the manager
  has been installed.

Operations requested during a frame are applied at the next `pre_update`. The
outgoing Bundle therefore completes its frame and runs `shutdown` then
`finalize` before the incoming Bundle runs `initialize` and `startup`.

```sx
use GFX.Application
use GFX.Plugins

class Page { let number:int }

func update_page(
    page:@Page,
    manager:&Plugins.BundleManager,
    controller:&Application.Controller
) {
    if page.number == 2 { controller.stop(); return }
    manager.replace(manager.create()
        ..add_resource(Page(number:2))
        ..add_system(Application.Schedule.update, update_page))
}

func main() {
    var application = Application()
    var first = Application.Bundle(application)
        ..add_resource(Page(number:1))
        ..add_system(Application.Schedule.update, update_page)

    application.add_plugin(Plugins.BundleManager(first))
    assert(application.run() == 0)
}
```

Each Bundle owns its `Application.Controller`. Stopping it only stops that
Bundle. `BundleManager` automatically finalizes stopped Bundles and stops the
Application when the last one is gone. A window placed inside a Bundle may
therefore close independently while the Application remains responsible for
the single main loop.

## Common layouts

For a single-window application whose content changes, place `Window`,
`Rendering`, and `Scene2D` on the Application, then place content-specific
resources and Plugins in its Bundles.

For several independent windows, the Application may contain only
`BundleManager`. Each Bundle then installs its own `Window`, renderer, and
required 2D or 3D capabilities. Both layouts use the same API; only the chosen
Plugin scope changes.

## Executable examples

- [AnalogClock](https://github.com/Matanek/Silex-Examples/blob/main/Sources/AnalogClock.sx)
  hosts one window's Canvas content in a Bundle.
- [SceneTransitions](https://github.com/Matanek/Silex-Examples/blob/main/Sources/SceneTransitions.sx)
  uses `replace` to represent two scenes with isolated Bundles.

Each executable accepts `--verify` for bounded automated validation.
