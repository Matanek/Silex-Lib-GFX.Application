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
