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
