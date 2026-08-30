# GFX.Application

`GFX.Application` ajoute des contextes applicatifs composables au module
canonique. `Application` conserve l’unique boucle principale du processus ;
`Application.Bundle` possède un ensemble local de Plugins, de systèmes et de
ressources piloté par cette boucle.

Un Bundle peut représenter le contenu temporaire d’une fenêtre, une fenêtre
complète, un outil autonome ou une autre sous-application. Il ne désigne pas
une scène graphique : un futur système de scènes pourra s’appuyer sur les
Bundles sans confondre les deux notions.

## Plugins et ressources

Un Bundle emploie exactement le même protocole `Application.Plugin` qu’une
Application :

```sx
var bundle = Application.Bundle(application)
    ..add_plugin(Plugins.Grid2D())
    ..add_resource(Document())
    ..add_system(Application.Schedule.update, update_document)
```

Il n’existe ni `ScenePlugin`, ni variante `Content` des Plugins 2D, 3D ou
Animation. Un Plugin installé sur un Bundle reçoit un `&Application` standard
qui représente son contexte local.

Les ressources locales masquent les ressources parentes du même type. Une
lecture absente du Bundle remonte vers l’Application sans copier la valeur ; la
suppression ou la finalisation du Bundle ne modifie jamais le parent.

Un Plugin qui fournit une capacité durable peut appeler
`application.extend_bundles(self)`. Le même Plugin est alors préparé dans
chaque Bundle. `Scene2D`, `Scene3D` et `Animation` utilisent ce mécanisme :
installés globalement, ils conservent leurs capacités partagées et raccordent
automatiquement leurs systèmes locaux aux Bundles. Un Plugin comme `Grid2D`
peut aussi ajouter `Scene2D` directement dans son propre Bundle. L’utilisateur
n’a donc aucun second type de Plugin à découvrir ou à déclarer.

## BundleManager

`Plugins.BundleManager` monte, exécute et finalise les Bundles aux frontières
sûres de la boucle principale.

- `add(bundle)` ajoute un Bundle sans retirer ceux qui sont actifs ; plusieurs
  fenêtres ou outils peuvent ainsi progresser ensemble.
- `remove(bundle)` retire un Bundle précis.
- `replace(bundle)` remplace atomiquement tous les Bundles actifs ; ce chemin
  convient à un splashscreen, une page ou une scène de jeu.
- `create()` crée un Bundle associé à l’Application une fois le manager
  installé.

Les opérations demandées pendant une frame sont appliquées au prochain
`pre_update`. Le Bundle sortant termine donc sa frame, exécute `shutdown` puis
`finalize`, avant les phases `initialize` et `startup` du Bundle entrant.

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

Chaque Bundle possède son `Application.Controller`. Arrêter ce contrôleur ne
coupe que le Bundle concerné. `BundleManager` finalise automatiquement les
Bundles arrêtés et arrête l’Application lorsque le dernier a disparu. Une
fenêtre placée dans un Bundle peut donc se fermer indépendamment des autres,
tandis que l’Application reste responsable de l’unique boucle principale.

## Répartition courante

Pour une application à fenêtre unique dont seul le contenu change, placez par
exemple `Window`, `Rendering` et `Scene2D` sur l’Application, puis les ressources
et Plugins propres à chaque contenu dans ses Bundles.

Pour plusieurs fenêtres indépendantes, l’Application peut ne contenir que
`BundleManager`. Chaque Bundle installe alors son propre `Window`, son renderer
et les capacités 2D ou 3D nécessaires. Les deux formes emploient la même API ;
seule la portée choisie pour les Plugins change.

## Exemples exécutables

- [AnalogClock](https://github.com/Matanek/Silex-Examples/blob/main/Sources/AnalogClock.sx)
  héberge le contenu Canvas d’une fenêtre dans un Bundle.
- [BundleTransitions](https://github.com/Matanek/Silex-Examples/blob/main/Sources/BundleTransitions.sx)
  utilise `replace` pour représenter deux scènes avec des Bundles isolés.

Chaque exécutable accepte `--verify` afin d’effectuer une validation bornée et
automatique.
