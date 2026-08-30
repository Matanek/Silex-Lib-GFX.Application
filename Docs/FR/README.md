# GFX.Application

`GFX.Application` complète le module applicatif canonique avec une scène de
durée de vie plus courte. Le type `Application` reste fourni par GFX ; le
package ajoute `Application.Scene` sans façade de plateforme ni moteur de jeu.

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

## Durée de vie

Construire et configurer une scène n’exécute aucun système. `mount()` exécute
`initialize`, puis `startup`, une seule fois. `run_schedule` accepte les phases
de frame ordinaires. `unmount()` exécute `shutdown`, puis `finalize`, libère les
ressources et entités locales et rend la scène définitivement inutilisable.
Tout handle obtenu auparavant avec `scene.resources()` est invalidé au même
moment : il ne peut ni lire les ressources parentes, ni réinsérer une valeur.

La scène possède son `ECS.World` et ses `ECS.Commands`. Deux scènes peuvent donc
employer les mêmes types de composants sans partager leurs entités. Les
commandes structurelles sont appliquées après `post_update` et pendant l’arrêt.
Une dernière application après `finalize` garantit qu’aucune commande différée
ne survit au contexte retiré.

## Ressources globales et locales

Les ressources ajoutées à la scène sont locales. Lorsqu’un système demande un
type absent de la scène, l’injection consulte les ressources de l’Application
associée sans les copier. Une ressource locale de même type masque toujours la
ressource globale. Le retrait ou la finalisation de la scène ne modifie jamais
le magasin global.

Le package dépend seulement de `GFX` et `GFX.ECS`. Les domaines Scene2D,
Scene3D, Rendering, Canvas, Animation et Audio se raccordent séparément ; ils ne
font pas partie de ce premier contexte générique.

## Plugins de scène

`Application.ScenePlugin` permet à un package spécialisé de configurer une
scène tant qu’elle est prête. `scene.add_plugin(...)` conserve le Plugin,
appelle `build(scene)` une seule fois par identifiant et refuse toute nouvelle
configuration après le montage.

Un Plugin de scène qui dépend d’une capacité globale l’enregistre avec
`scene.require_application_plugin(id)`. La vérification a lieu juste avant le
montage, après la résolution différée des Plugins de l’Application ; une
configuration incomplète échoue donc avant `initialize` avec l’identifiant de
la capacité manquante.

Les packages graphiques séparent ainsi deux intentions : `Plugin()` installe
les capacités globales dont la durée de vie couvre l’Application, tandis que
`Content()` raccorde les systèmes qui lisent les composants de la scène active.
Une même scène peut par exemple combiner `Scene2D.Content()`,
`Scene3D.Content()` et `Animation.Content()`.

## Hébergement dans Application

`Application.Scenes` est à la fois le Plugin statique et l’hôte injectable qui
possède l’unique scène active. Il ajoute six relais fixes aux schedules globaux ;
un changement de scène ne modifie donc jamais le plan de l’Application.

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

L’Application monte la scène initiale pendant `startup`. Une demande reçue
pendant une frame reste en attente jusqu’au début du prochain `pre_update` : la
scène sortante termine donc sa frame, puis exécute `shutdown` et `finalize`
avant les `initialize` et `startup` de la scène entrante. Une seconde demande
tant que cette frontière n’est pas atteinte produit une erreur explicite.

À l’arrêt de l’Application, une demande encore en attente est annulée et la
scène active est finalisée. Installer d’abord les Plugins fournissant les
capacités globales, puis `Scenes`, respecte l’ordre inverse d’arrêt des Plugins
et garantit que la scène libère ses usages avant ces capacités.
