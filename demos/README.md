# Demos : singletons, static, WeakReference (Godot 4.7 / C#)

## 1. Les trois façons de faire un singleton

### a) Autoload + propriété statique — `singletons/GameState.cs`

C'est le pattern par défaut en Godot. Le nœud est un autoload (Project Settings → Autoload),
donc l'engine le crée, l'ajoute sous `/root/` et le libère. La propriété `static Instance`
n'est qu'un raccourci d'accès, elle ne possède rien.

Points importants :
- `Instance` est assignée dans `_EnterTree`, pas `_Ready` : `_EnterTree` passe avant tous les
  `_Ready` des scènes, donc n'importe quel nœud peut appeler `GameState.Instance` dès son `_Ready`.
- `Instance` est remise à `null` dans `_ExitTree`. Sans ça, en fermant le jeu ou en rechargeant
  l'assembly depuis l'éditeur, tu gardes un pointeur vers un objet natif détruit → crash ou
  `ObjectDisposedException`.
- Le garde `if (Instance != null && Instance != this)` protège contre le cas où quelqu'un
  instancie la scène une deuxième fois à la main.

À utiliser quand : le singleton a besoin du cycle de vie Godot (`_Process`, signaux, timers,
resources, arbre de scène).

### b) Singleton C# pur `Lazy<T>` — `singletons/SaveService.cs`

Pas de `Node`, pas d'autoload, pas de `.tscn`. Construit à la première utilisation, thread-safe.
Constructeur privé pour que personne ne puisse en faire un deuxième.

À utiliser quand : le service ne touche pas à l'arbre de scène (I/O, config, sérialisation,
réseau, calcul). C'est plus simple, plus testable, et ça ne dépend pas de l'ordre des autoloads.

### c) Classe `static` pure — `singletons/StaticVsInstance.cs` (`DamageMath`, `SceneRoutes`)

Ce n'est pas vraiment un singleton, c'est un espace de noms de fonctions. Aucun état.

À utiliser quand : fonctions pures et constantes uniquement.

### Comment y accéder

```csharp
GameState.Instance.Score += 10;                       // rapide, mais couplage dur
GetNode<GameState>("/root/GameState");                 // testable, resolvable par chemin
[Export] public GameState State;                       // injection : le plus testable
```

Pour un vrai jeu, garder `Instance` pour le confort mais éviter de l'appeler dans 200 fichiers.
Récupérer la référence une fois dans `_Ready` et la stocker.

---

## 2. static ou pas static ?

La question à se poser : **est-ce qu'il peut logiquement y en avoir deux ?**

| Cas | Choix |
|---|---|
| Fonction pure : `ApplyArmor(damage, armor)` | `static` |
| Constante : chemins de scènes, layers, tags | `static readonly` / `const` |
| État partagé par tout le jeu et un seul par process | singleton (instance derrière `static Instance`) |
| État qui appartient à une entité (PV, inventaire, RNG) | **jamais** `static` |
| Quoi que ce soit qui doit être réinitialisé entre deux parties | **jamais** `static` |

### Pourquoi le `static` d'état fait mal en Godot

`BadGlobalCounter` dans `StaticVsInstance.cs` montre le piège :

1. **Rien ne le remet à zéro.** Recharger la scène ne réinitialise pas les statiques. Il faut y
   penser à la main, et un jour on oublie.
2. **Fuite mémoire.** Un `static` qui pointe vers un `Node` empêche le wrapper C# d'être collecté
   même après `QueueFree()`. Une liste statique d'ennemis qui n'est jamais vidée = fuite garantie.
3. **Rechargement d'assembly.** Quand l'éditeur recompile, les statiques sont réinitialisées
   silencieusement pendant que le jeu tourne. Les bugs qui en découlent sont insupportables à
   debug.
4. **Non testable, non parallélisable.** Deux tests qui touchent la même statique interfèrent.

Règle pratique : `static` pour le comportement et les constantes, instance pour l'état.
`RunCounter` (instance) vs `BadGlobalCounter` (static) dans le fichier montrent la même
fonctionnalité des deux côtés de la ligne.

### `readonly` et `sealed` tant qu'on y est

- `private readonly` sur les champs qui ne changent pas après le constructeur : le compilateur
  vérifie, et ça documente l'intention sans commentaire.
- `sealed` sur les classes non destinées à l'héritage : le JIT peut dévirtualiser les appels.

---

## 3. WeakReference

### Le principe

Une référence normale empêche le GC de collecter l'objet. Une `WeakReference<T>` pointe vers
l'objet **sans le maintenir en vie**. Si plus personne d'autre ne le tient, le GC le ramasse et
la weak reference devient vide.

```csharp
var weak = new WeakReference<Payload>(payload);
if (weak.TryGetTarget(out Payload p))
    Use(p);      // vivant
else
    Rebuild();   // collecte
```

Toujours `TryGetTarget`, jamais `IsAlive` puis déréférencement : entre les deux lignes, le GC
peut passer.

Voir `weakrefs/WeakReferenceBasics.cs` — il force un `GC.Collect()` pour montrer la transition.
(Un `GC.Collect()` manuel est un outil de démo, pas de production.)

### Les trois vrais cas d'usage

**a) Cache qui ne doit pas maintenir en vie** — `weakrefs/WeakCache.cs`

Un cache d'assets lourds : si le reste du jeu utilise encore la texture, le cache la ressert ;
sinon le GC la libère. Attention : le `Dictionary` lui-même garde les *clés* et les objets
`WeakReference` vides à vie → d'où le `Sweep()` périodique. C'est le piège classique de ce
pattern.

Si les clés sont des objets, `ConditionalWeakTable<TKey, TValue>` fait ça correctement et
automatiquement (clés faibles, nettoyage par le GC).

**b) Éviter les fuites par abonnement d'événement** — `weakrefs/WeakEventBus.cs`

En C#, `bus.SomeEvent += OnThing;` fait que **le bus tient l'abonné en vie**, pas l'inverse.
C'est la première cause de fuite mémoire en C#. Deux solutions :

1. Se désabonner proprement dans `_ExitTree` (voir `SingletonUsage.cs`) — préférable quand c'est
   possible.
2. Un bus à références faibles quand on ne contrôle pas la durée de vie des abonnés.

Note : les **signaux Godot** (`[Signal]` + `EmitSignal`) ne souffrent pas de ça de la même
manière, la connexion est côté natif et est coupée quand un des deux nœuds est libéré. Mais un
`+=` sur un événement C# classique, si.

**c) Attacher des données à un objet qu'on ne possède pas** — `ConditionalWeakTable`
dans `weakrefs/GodotObjectLifetime.cs`

Permet d'associer des métadonnées à un `Node` sans modifier sa classe et sans le maintenir en
vie. L'entrée disparaît toute seule quand le nœud est collecté.

### Le piège Godot spécifique

Un `Node` C# c'est **deux objets** : le wrapper managé et l'objet natif. Ils ne meurent pas
ensemble.

- `QueueFree()` détruit le natif. Le wrapper C# reste vivant tant qu'une référence managée le
  tient.
- Donc une `WeakReference<Node>` peut très bien te rendre un nœud **dont le natif est déjà mort**.
  Toucher `node.Name` à ce moment-là lève `ObjectDisposedException`.

D'où la double vérification, obligatoire :

```csharp
if (slot.TryGetTarget(out Node node) && GodotObject.IsInstanceValid(node))
    Use(node);
```

`ResolveOrForget` dans `GodotObjectLifetime.cs` encapsule ça.

Pour référencer un nœud « faiblement » en Godot, le plus idiomatique reste en réalité :
- stocker un `NodePath` et faire `GetNodeOrNull<T>(path)`,
- ou stocker le nœud et vérifier `IsInstanceValid` avant chaque usage.

`WeakReference<Node>` sert surtout quand le souci est la **mémoire managée** (empêcher un cache
ou un event bus de retenir des milliers de wrappers), pas la validité du nœud.

### Quand ne PAS utiliser WeakReference

- Comme rustine sur une fuite mal comprise. Trouver qui retient l'objet, c'est ça la correction.
- Pour du cache court terme : les objets survivront de toute façon jusqu'au prochain GC, et
  disparaîtront de manière imprévisible. Un cache à taille bornée (LRU) est plus prévisible.
- Sur des objets légers : le coût d'une `WeakReference` (allocation + handle GC) dépasse
  largement l'objet.
- Pour des `RefCounted` Godot : leur durée de vie est gérée par comptage de références côté
  natif, mélanger les deux modèles porte à confusion.
