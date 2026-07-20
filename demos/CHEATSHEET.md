# Godot 4 + C# — la feuille de triche

## Les 7 règles qui évitent 90 % des bugs

| # | Règle |
|---|---|
| 1 | `GetNode` dans `_Ready`, **jamais** dans `_Process`. On récupère une fois, on stocke. |
| 2 | Tout ce qui bouge se multiplie par `delta`. Sans exception. |
| 3 | Un `+=` sur un événement C# exige un `-=` dans `_ExitTree`. Sinon : fuite. |
| 4 | `QueueFree()`, pas `Free()`. |
| 5 | Avant de toucher un nœud gardé en mémoire : `IsInstanceValid(node)`. |
| 6 | `static` pour le comportement, **jamais** pour l'état d'une partie. |
| 7 | Physique et mouvement dans `_PhysicsProcess`, affichage et input dans `_Process`. |

---

## Je veux… → j'écris…

| Je veux | J'écris |
|---|---|
| Un enfant de mon nœud | `GetNode<Label>("UI/Score")` dans `_Ready` |
| Sans planter s'il n'existe pas | `GetNodeOrNull<Label>("UI/Score")` |
| Le désigner dans l'inspecteur | `[Export] private Label _score;` |
| Créer une scène à la volée | `[Export] PackedScene _prefab;` puis `_prefab.Instantiate()` |
| Ajouter au monde | `AddChild(node)` |
| Détruire | `node.QueueFree()` |
| Attendre 2 secondes | `await ToSignal(GetTree().CreateTimer(2.0), SceneTreeTimer.SignalName.Timeout);` |
| Animer une propriété | `CreateTween().TweenProperty(this, "position", target, 0.5);` |
| Prévenir les autres | `[Signal] delegate void DiedEventHandler();` + `EmitSignal(SignalName.Died)` |
| Écouter | `truc.Died += OnDied;` (et `-=` dans `_ExitTree`) |
| Un réglage éditable | `[Export] public float Speed { get; set; } = 300f;` |
| Afficher un truc | `GD.Print($"valeur = {x}")` |
| Un avertissement visible | `GD.PushWarning("...")` |

---

## `_Process` ou `_PhysicsProcess` ?

| | `_Process` | `_PhysicsProcess` |
|---|---|---|
| Fréquence | variable (le FPS) | fixe (60/s) |
| Pour | UI, caméra, input, timers, effets | déplacement, collisions, IA, `MoveAndSlide` |
| Piège | dépend de la machine du joueur | ne pas y mettre de dessin |

---

## static ou pas static ?

**La question :** est-ce qu'il peut logiquement en exister deux ?

| Cas | Réponse |
|---|---|
| `ApplyArmor(damage, armor)` — fonction pure | ✓ `static` |
| Chemins de scènes, constantes | ✓ `static readonly` / `const` |
| PV, inventaire, position | ✗ jamais |
| Ce qui doit repartir à zéro entre 2 parties | ✗ jamais |
| Le gestionnaire audio, le save | singleton (instance derrière `static Instance`) |

**Comportement → static. État → instance.**

---

## struct ou class ?

| | `struct` | `class` |
|---|---|---|
| Passé à une méthode | **copié** | **partagé** |
| Bon pour | petites valeurs : position, dégâts, couleur | tout le reste |
| Défaut recommandé | `readonly record struct` | `sealed class` |

Si tu hésites : **`class`**. Le `struct` mal utilisé crée des bugs silencieux (voir `bases/StructVsClass.cs`).

---

## Quelle collection ?

| Besoin | Type |
|---|---|
| Taille connue et figée | `float[]` |
| Ça grossit, l'ordre compte | `List<T>` |
| Retrouver par identifiant | `Dictionary<K,V>` |
| « Est-ce que je l'ai déjà vu ? » | `HashSet<T>` |
| File d'attente | `Queue<T>` |
| Annuler / historique | `Stack<T>` |

---

## Les pièges qui coûtent une soirée

| ✗ Faux | ✓ Juste |
|---|---|
| `if (a == b)` sur des `float` | `Mathf.IsEqualApprox(a, b)` |
| `int ratio = current / max;` | `float ratio = (float)current / max;` |
| `GetNode` dans `_Process` | cacher dans `_Ready` |
| `node.Free()` | `node.QueueFree()` |
| `if (node != null)` sur un nœud libéré | `if (IsInstanceValid(node))` |
| `Position += Vector2.Right * 5f;` | `... * speed * (float)delta;` |
| `bus.Event += Handler;` sans `-=` | `-=` dans `_ExitTree` |
| Concaténer des strings chaque frame | `StringBuilder`, ou ne mettre à jour que si ça change |
| `static List<Enemy>` qu'on ne vide jamais | retirer dans `_ExitTree` |

---

## Trois façons de gérer un « manager » global

| Approche | Fichier | Quand |
|---|---|---|
| Autoload + `static Instance` | `singletons/GameState.cs` | il a besoin de l'arbre de scène (signaux, `_Process`) |
| `Lazy<T>` C# pur | `singletons/SaveService.cs` | il ne touche pas à l'arbre (save, config, réseau) |
| `static class` | `singletons/StaticVsInstance.cs` | que des fonctions et des constantes |

Détail qui compte : assigner `Instance` dans **`_EnterTree`** (pas `_Ready`), et la remettre à `null` dans `_ExitTree`.

---

## Mémoire : les 3 choses à savoir

1. **Un `Node` C#, c'est deux objets** : le wrapper C# et l'objet natif. `QueueFree()` tue le natif, pas le wrapper. D'où `IsInstanceValid`.
2. **Ce qui retient un objet en vie** : une variable `static`, une `List` jamais vidée, un événement jamais désabonné. C'est presque toujours l'un des trois.
3. **`WeakReference<T>`** pointe sans retenir. Utile pour les caches et les bus d'événements — pas comme rustine sur une fuite qu'on n'a pas comprise.

---

## Par où commencer dans les fichiers

Dans l'ordre, du plus utile au plus pointu :

```
bases/NodeLifecycle.cs      quand chaque fonction est appelee — LANCE CELUI-LA EN PREMIER
bases/GetNodeWays.cs        les 6 facons de recuperer un noeud
gameplay/DeltaTime.cs       pourquoi tout se multiplie par delta
pieges/Pieges.cs            faux / juste, cote a cote
bases/NullSafety.cs         ?. ?? ??= : ne plus jamais crasher sur null
bases/CollectionsChoice.cs  quelle collection pour quel besoin
bases/StructVsClass.cs      copie vs partage
gameplay/Components.cs      composer plutot qu'heriter (HealthComponent)
gameplay/SignalsVsEvents.cs signaux Godot vs events C#
gameplay/StateMachine.cs    machine a etats (version simple + version propre)
gameplay/ObjectPool.cs      recycler au lieu d'allouer
gameplay/AsyncInGodot.cs    await, Tween, threads
singletons/                 les managers globaux           -> voir README.md
weakrefs/                   references faibles et fuites   -> voir README.md
```

`README.md` = les explications longues sur les singletons, le static et les `WeakReference`.
Cette page = tout le reste, en condensé.
