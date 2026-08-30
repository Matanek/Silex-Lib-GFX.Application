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
`Scene2D.Content()`, `Scene3D.Content()`, and `Animation.Content()`, for
example.

## Application hosting

`Application.Scenes` is both the static Plugin and the injectable host that
owns the single active scene. It adds six fixed relays to the global schedules,
so replacing a scene never changes the Application plan.

```sx
use GFX.Application

class Level { let number:int }

func next_level(
    scenes:&Application.Scenes,
    level:@Level,
    controller:&Application.Controller
) {
    if level.number == 2 { controller.stop(); return }
    scenes.request(scenes.create()
        ..add_resource(Level(number:level.number + 1))
        ..add_system(Application.Schedule.update, next_level))
}

func main() {
    var application = Application()
    var initial = Application.Scene(application)
        ..add_resource(Level(number:1))
        ..add_system(Application.Schedule.update, next_level)
    var scenes = Application.Scenes(initial)

    application.add_plugin(scenes)
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
`Scenes` last; the reverse Plugin shutdown order then guarantees that the scene
releases its uses before those capabilities.
