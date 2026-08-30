# GFX.Application

`GFX.Application` adds a shorter-lived scene to the canonical application
module. GFX still provides the `Application` type; this package adds
`Application.Scene` without a platform facade or game-engine object model.

```sx
use GFX.Application

struct Score { var value:int }

func update(score:&Score) {
    score.value += 1
}

func main() {
    var application = Application()
    var scene = Application.Scene(application)
        ..add_resource(Score(value:0))
        ..add_system(Application.Schedule.update, update)

    scene.mount()
    scene.run_schedule(Application.Schedule.update)
    scene.unmount()
}
```

## Lifetime

Constructing and configuring a scene runs no system. `mount()` runs
`initialize`, then `startup`, exactly once. `run_schedule` accepts ordinary
frame phases. `unmount()` runs `shutdown`, then `finalize`, releases local
resources and entities, and permanently finalizes the scene.
Any handle previously returned by `scene.resources()` is invalidated at the
same boundary: it can neither read parent resources nor insert another value.

The scene owns its `ECS.World` and `ECS.Commands`, so two scenes may use the
same component types without sharing entities. Structural commands are applied
after `post_update` and during shutdown.
A final flush after `finalize` guarantees that no deferred command survives the
removed context.

## Global and local resources

Resources added to a scene are local. When a system requests a type absent from
the scene, injection consults the associated Application resources without
copying them. A local resource of the same type always shadows the global one.
Removing or finalizing the scene never changes the global store.

The package depends only on `GFX` and `GFX.ECS`. Scene2D, Scene3D, Rendering,
Canvas, Animation, and Audio integrate separately; they are not part of this
first generic context.

## Choose the application scene or graphics domain

These names answer different intentions:

| Capability | What it owns |
| --- | --- |
| `Application` | the loop, windows, GPU, and resources that survive transitions |
| `Application.Scene` | a replaceable system plan, its local resources, and its `ECS.World` |
| `Scene2D` / `Scene3D` | the specialized components and systems that describe and render content |

An application scene is therefore neither a 2D nor a 3D scene. It may contain
no graphics, use one domain, or compose both. For 2D content, install
`Plugins.Scene2D()` once on the Application so the renderer and its caches stay
alive, then add `Plugins.Scene2DContent()` to every scene that owns the
components. The same separation applies to Scene3D and Animation. All these
Plugins are exposed directly under `GFX.Plugins`, so catalog completion is
enough to discover them.

Systems remain ordinary functions registered through
`add_system(Schedule, callback)`. Their parameters inject existing resources
and queries; a scene does not automatically turn a class into an entity or a
method into a system.

## Scene plugins

`Application.ScenePlugin` lets a specialized package configure a scene while
it is ready. `scene.add_plugin(...)` retains the Plugin, calls `build(scene)`
once per identifier, and rejects further configuration after mounting.

A scene Plugin that depends on a global capability records it with
`scene.require_application_plugin(id)`. Validation runs immediately before
mounting, after the Application's deferred Plugins have resolved, so incomplete
configuration fails before `initialize` and names the missing capability.

Graphics packages therefore separate two intents: `Plugin()` installs global
capabilities whose lifetime spans the Application, while `Content()` connects
systems that read components from the active scene. One scene may combine
`Plugins.Scene2DContent()`, `Plugins.Scene3DContent()`, and
`Plugins.AnimationContent()`, for example.

## Application hosting

`Plugins.SceneManager` exposes both the static Plugin and the injectable
host that owns the single active scene. It adds six fixed relays to the global
schedules, so replacing a scene never changes the Application plan.

```sx
use GFX.Application
use GFX.Plugins

class Level { let number:int }

func next_level(
    manager:&Plugins.SceneManager,
    level:@Level,
    controller:&Application.Controller
) {
    if level.number == 2 { controller.stop(); return }
    manager.request(manager.create()
        ..add_resource(Level(number:level.number + 1))
        ..add_system(Application.Schedule.update, next_level))
}

func main() {
    var application = Application()
    var initial = Application.Scene(application)
        ..add_resource(Level(number:1))
        ..add_system(Application.Schedule.update, next_level)
    var manager = Plugins.SceneManager(initial)

    application.add_plugin(manager)
    application.run()
}
```

The Application mounts the initial scene during `startup`. A request made in a
frame remains pending until the beginning of the next `pre_update`: the
outgoing scene therefore completes its frame, then runs `shutdown` and
`finalize` before the incoming scene runs `initialize` and `startup`. A second
request before that boundary reports an explicit error.

On Application shutdown, any pending request is cancelled and the active scene
is finalized. Install Plugins that provide global capabilities first and
`SceneManager` last; the reverse Plugin shutdown order then guarantees that the scene
releases its uses before those capabilities.

## Executable examples

- [AnalogClock](https://github.com/Matanek/Silex-Examples/blob/main/Sources/AnalogClock.sx)
  retains its Canvas drawings and updates only the hand transforms in one
  scene.
- [SceneTransitions](https://github.com/Matanek/Silex-Examples/blob/main/Sources/SceneTransitions.sx)
  replaces two contexts that use the same resource and component types without
  mixing their entities.

Each demo compiles as an ordinary public application. Its executable accepts
`--verify` to open a hidden window, run a bounded number of frames, assert its
invariants, and stop automatically.
