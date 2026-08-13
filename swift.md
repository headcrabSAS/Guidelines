# Guidelines Swift

Ce document couvre les bonnes pratiques spécifiques au langage Swift.

Consultez également [general.md](general.md) pour les pratiques communes à tous les langages.

---

## Sommaire

- [Convention de nommage](#convention-de-nommage)
- [let vs var](#let-vs-var)
- [Optionals](#optionals)
- [Guard et Early Return](#guard-et-early-return)
- [Structs vs Classes](#structs-vs-classes)
- [Protocoles](#protocoles)
- [Extensions](#extensions)
- [Propriétés](#propriétés)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Mémoire et ARC](#mémoire-et-arc)
- [Async / Await](#async--await)
- [Flags de compilation](#flags-de-compilation)

---

## Convention de nommage

| Élément | Convention | Exemple |
|---|---|---|
| Classe | PascalCase | `class PlayerController` |
| Classe de base | PascalCase + suffixe `Base` | `class UnitBase` |
| Struct | PascalCase | `struct PlayerStats` |
| Protocole de capacité | `ICan` + PascalCase | `ICanMove`, `ICanFight` |
| Protocole de contrat | `I` + PascalCase | `IAuthStore`, `IAPIClient` |
| Enum | `E` + PascalCase | `EGameState` |
| Valeur d'enum | camelCase | `.mainMenu`, `.gameOver` |
| Fonction / Méthode | camelCase | `func computeScore()` |
| Propriété | camelCase | `var currentScore` |
| Constante | camelCase | `let maxPlayers = 4` |
| Collection | Pluriel | `var enemies: [Enemy]` |

> **Note sur les conventions Apple** : Apple ne recommande ni le préfixe `I` sur les protocoles, ni le préfixe `E` sur les enums. Nous les conservons par cohérence avec nos projets C# — un développeur qui passe d'un projet à l'autre retrouve ses repères immédiatement. La cohérence interne prime sur la conformité aux conventions de l'écosystème.

### Constantes

Swift utilise `let` pour les constantes — camelCase, sans préfixe `K_`. Le compilateur distingue `let` (immuable) de `var` (mutable), rendant tout préfixe redondant.

```swift
// ✅
let maxPlayers = 4
let sessionTimeout: TimeInterval = 86_400
static let defaultName = "Player"

// ❌ — Préfixe inutile en Swift, le compilateur l'impose déjà
let K_MAX_PLAYERS = 4
```

### Protocoles — deux types, deux nommages

Les protocoles jouent deux rôles distincts qui appellent des conventions de nommage différentes.

**Protocoles de capacité** — ce qu'un objet *peut faire* :
```swift
ICanMove, ICanFight, ICanCollect
```

**Protocoles de contrat/service** — ce qu'un objet *est* dans l'architecture (repository, store, client...) :
```swift
IAuthStore      // deux implémentations : KeychainStore, InMemoryStore
IAPIClient      // implémentation réelle + mock pour les tests
IAuthRepository
IPermissionsManager  // contrat de base pour les managers de permissions
```

Les deux coexistent. `ICanAuthStore` serait ridicule — le préfixe `ICan` ne s'applique qu'aux capacités comportementales. Pour les rôles architecturaux, `IXxx` avec un nom de type est le bon pattern.

---

## let vs var

Déclarez tout en `let` par défaut. Passez à `var` uniquement quand la mutation est nécessaire.

```swift
// ✅
let player = Player()
let scores = [10, 20, 30]

// ✅ — var justifié : la valeur change
var currentScore = 0
currentScore += points

// ❌ — var inutile si la valeur ne change jamais
var maxLives = 3  // → let maxLives = 3
```

**Pourquoi ?** Le compilateur avertit quand un `var` n'est jamais muté (`"Variable 'x' was never mutated; consider changing to 'let' constant"`). Cet avertissement est une aide, pas une nuisance — traitez-le comme une erreur. Un `let` communique l'intention d'immuabilité à qui lit le code.

---

## Optionals

Un optional (`T?`) représente une valeur qui peut être absente. Swift force à gérer explicitement ce cas — c'est l'une de ses protections les plus importantes contre les crashes.

### Jamais de force-unwrap sans raison absolue

L'opérateur `!` (force-unwrap) crash si la valeur est `nil`. Il est acceptable uniquement quand l'absence de valeur est impossible par construction et que vous pouvez l'argumenter.

```swift
// ❌ — Crash garanti si user est nil
let name = user!.name

// ✅ — if let : valeur utilisable uniquement si présente
if let user = currentUser {
    display(user.name)
}

// ✅ — guard let : early exit si absente (voir section suivante)
guard let user = currentUser else { return }
display(user.name)

// ✅ — optional chaining : nil si user est nil, pas de crash
let name = currentUser?.name

// ✅ — nil coalescing : valeur par défaut
let name = currentUser?.name ?? "Guest"
```

### Nommage des optionals

Un optional dont le nom implique qu'il est toujours présent est trompeur.

```swift
// ❌ — Le nom suggère une présence garantie
var currentUser: User?

// ✅ — Si l'absence est possible, le nom peut le refléter
var loggedInUser: User?
```

---

## Guard et Early Return

`guard` est le mécanisme naturel d'early return en Swift. Il est préférable à `if let` imbriqués quand la valeur est requise pour continuer.

```swift
// ❌ — Imbrication difficile à lire
func processOrder(_ order: Order?) {
    if let order = order {
        if order.isValid {
            if !order.items.isEmpty {
                submit(order)
            }
        }
    }
}

// ✅ — guard aplatit l'imbrication, la logique nominale est au premier plan
func processOrder(_ order: Order?) {
    guard let order = order else { return }
    guard order.isValid else { return }
    guard !order.items.isEmpty else { return }

    submit(order)
}
```

`guard let` lie la variable jusqu'à la fin du scope courant — contrairement à `if let` qui la confine au bloc. C'est son avantage principal : la valeur unwrappée est disponible pour tout ce qui suit.

---

## Structs vs Classes

Swift préfère les **structs** (types valeur) aux **classes** (types référence). Utilisez une struct par défaut, et passez à une classe uniquement quand c'est nécessaire.

```swift
// ✅ — Struct pour la data, les modèles, les configurations
struct PlayerStats {
    let level: Int
    var health: Int
    var score: Int
}

// ✅ — Classe pour les objets avec identité, cycle de vie, ou héritage
class GameSession {
    var players: [Player] = []
    func start() { ... }
}
```

**Utilisez une struct quand :**
- L'objet représente de la donnée (modèle, configuration, état)
- La copie est le comportement attendu (chaque appelant a sa propre version)
- Pas besoin d'héritage

**Utilisez une classe quand :**
- L'objet a une identité propre et un cycle de vie géré
- L'héritage est nécessaire
- L'interop Objective-C est requise
- Des références partagées sont intentionnelles

**Pourquoi ?** Les structs sont copiées — elles n'ont pas de retain cycles, pas de problème de partage involontaire d'état. Elles sont plus prévisibles et plus faciles à raisonner. Le compilateur optimise aussi mieux les structs dans la plupart des cas.

### `final` par défaut pour les classes

Toute classe qui n'est pas conçue pour être héritée doit être marquée `final`. Même principe que `sealed` en C# — voir [general.md](general.md).

```swift
// ✅
final class NetworkManager { ... }

// ✅ — Base explicitement conçue pour l'héritage
class UnitBase { ... }
```

---

## Protocoles

Les protocoles sont l'équivalent des interfaces en C#. Ils définissent un contrat sans implémentation (sauf via les extensions de protocole).

```swift
protocol CanMove {
    var speed: Float { get }
    func move(to position: CGPoint)
}

protocol CanFight {
    var attackPower: Int { get }
    func attack(target: CanFight)
}

// Une classe peut adopter plusieurs protocoles
final class Player: CanMove, CanFight {
    var speed: Float = 5.0
    var attackPower: Int = 10

    func move(to position: CGPoint) { ... }
    func attack(target: any CanFight) { ... }
}
```

### Séparation des conformances

Adoptez les protocoles dans des **extensions séparées** plutôt que dans la déclaration principale. Cela isole le code lié à chaque protocole et facilite la lecture.

```swift
// ✅
final class Player {
    var name: String
    var health: Int
}

extension Player: CanMove {
    var speed: Float { 5.0 }
    func move(to position: CGPoint) { ... }
}

extension Player: Equatable {
    static func == (lhs: Player, rhs: Player) -> Bool {
        lhs.name == rhs.name
    }
}
```

---

## Extensions

Les extensions permettent d'ajouter des fonctionnalités à un type existant — y compris des types de la bibliothèque standard.

```swift
// ✅ — Extension utilitaire sur un type standard
extension String {
    var isValidEmail: Bool {
        contains("@") && contains(".")
    }
}

extension Array where Element: Numeric {
    var sum: Element { reduce(0, +) }
}
```

**Règles :**
- Une extension doit avoir une responsabilité claire et cohérente.
- Évitez les extensions fourre-tout. Si une extension grandit trop, c'est le signal de créer un type dédié.
- N'ajoutez pas de fonctionnalités à un type tiers si elles ne sont pas génériques — créez un wrapper ou un type dédié à la place.

---

## Propriétés

### Propriétés calculées vs méthodes

Une **propriété calculée** est appropriée quand la valeur est dérivée directement d'un état existant et que le calcul est léger.

Une **méthode** est préférable quand le calcul est coûteux, a des effets de bord, ou prend des paramètres.

```swift
// ✅ — Propriété calculée : dérivée simple, sans coût
var isAlive: Bool { health > 0 }
var fullName: String { "\(firstName) \(lastName)" }

// ✅ — Méthode : calcul coûteux ou avec effets de bord
func generateReport() -> Report { ... }
func distance(to other: Player) -> Float { ... }
```

### Property observers

`willSet` et `didSet` permettent de réagir aux changements d'une propriété. Ils sont acceptables pour des effets de bord légers et directs — pas pour de la logique métier complexe (même règle que les setters en C#).

```swift
var health: Int = 100 {
    didSet {
        healthBar.update(to: health)  // ✅ — mise à jour UI directe
    }
}
```

### `lazy`

Une propriété `lazy` n'est initialisée qu'au premier accès. Utile pour des objets coûteux à créer dont on n'est pas sûr d'avoir besoin.

```swift
// ✅ — N'est créé que si effectivement utilisé
lazy var expensiveRenderer = HeavyRenderer()
```

---

## Gestion des erreurs

Swift propose deux mécanismes principaux : `throws`/`try`/`catch` et `Result<Success, Failure>`.

### throws / try / catch

Préférez `throws` pour les fonctions synchrones qui peuvent échouer de manière récupérable.

```swift
enum NetworkError: Error {
    case invalidURL
    case timeout
    case serverError(statusCode: Int)
}

func fetchUser(id: String) throws -> User {
    guard let url = URL(string: "https://api.example.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }
    // ...
}

// À l'appel
do {
    let user = try fetchUser(id: "42")
    display(user)
} catch NetworkError.invalidURL {
    showError("Invalid URL")
} catch NetworkError.serverError(let code) {
    showError("Server error: \(code)")
} catch {
    showError("Unexpected error: \(error)")
}
```

### Result\<Success, Failure\>

`Result` est utile quand vous voulez propager un résultat sans lever d'exception immédiatement — par exemple dans des callbacks ou des valeurs de retour asynchrones (avant async/await).

```swift
func fetchUser(id: String, completion: (Result<User, NetworkError>) -> Void) {
    // ...
    completion(.success(user))
    // ou
    completion(.failure(.timeout))
}

// À l'appel
fetchUser(id: "42") { result in
    switch result {
    case .success(let user): display(user)
    case .failure(let error): showError(error)
    }
}
```

**Règle générale** : préférez `throws`/`async throws` pour le code moderne. `Result` reste utile pour les APIs de callback ou quand vous stockez le résultat d'une opération pour le traiter plus tard.

---

## Mémoire et ARC

Swift gère la mémoire via l'ARC (Automatic Reference Counting). Il n'y a pas de garbage collector — les objets sont libérés quand leur compteur de références tombe à zéro.

### Retain cycles

Un **retain cycle** se produit quand deux objets se référencent mutuellement en `strong` — aucun des deux ne peut être libéré.

```swift
// ❌ — Retain cycle : Session retient Delegate, Delegate retient Session
class Session {
    var delegate: SessionDelegate?
}

class SessionDelegate {
    var session: Session?  // strong → retain cycle
}

// ✅ — weak brise le cycle
class SessionDelegate {
    weak var session: Session?
}
```

### weak vs unowned

| | `weak` | `unowned` |
|---|---|---|
| Type | Optional (`T?`) | Non-optional (`T`) |
| Si la référence est nil | Retourne nil | Crash |
| Quand l'utiliser | Quand la durée de vie de l'objet référencé est incertaine | Quand vous êtes sûr que l'objet existera aussi longtemps que vous |

```swift
// weak — le delegate peut ne plus exister
weak var delegate: SessionDelegate?

// unowned — le owner existera forcément aussi longtemps que self
unowned let owner: UserManager
```

**Règle** : en cas de doute, utilisez `weak`. `unowned` évite l'optional mais crash si l'hypothèse est fausse.

### Capture lists dans les closures

Les closures capturent les références en `strong` par défaut. Utilisez une **capture list** pour éviter les retain cycles ou les closures qui retiennent un objet mort.

```swift
// ❌ — self est retenu fortement, retain cycle possible si la closure est stockée
networkManager.fetch { result in
    self.display(result)
}

// ✅ — [weak self] : self peut être nil si l'objet est détruit pendant le fetch
networkManager.fetch { [weak self] result in
    guard let self else { return }
    self.display(result)
}

// ✅ — [unowned self] : acceptable si la durée de vie est garantie (ex: self-contained callback)
button.onTap { [unowned self] in
    self.handleTap()
}
```

---

## Async / Await

Swift 5.5+ introduit la concurrence structurée. C'est l'approche recommandée pour tout nouveau code asynchrone.

### Fonctions async

```swift
// ✅
func fetchUser(id: String) async throws -> User {
    let data = try await networkService.get("/users/\(id)")
    return try JSONDecoder().decode(User.self, from: data)
}

// À l'appel
Task {
    do {
        let user = try await fetchUser(id: "42")
        display(user)
    } catch {
        showError(error)
    }
}
```

### Nommage

Contrairement à C#, Swift n'impose pas de suffixe `Async` sur les fonctions async — le mot-clé `async` dans la signature est suffisamment explicite. Ne l'ajoutez pas.

```swift
// ✅
func fetchUser(id: String) async throws -> User

// ❌ — Redondant en Swift
func fetchUserAsync(id: String) async throws -> User
```

### @MainActor

Utilisez `@MainActor` pour garantir qu'un code s'exécute sur le thread principal. Obligatoire pour toute mise à jour d'UI depuis une tâche async.

```swift
// ✅ — La mise à jour UI est garantie sur le main thread
@MainActor
func updateUI(with user: User) {
    nameLabel.text = user.name
}

// ✅ — Annotation au niveau de la classe entière
@MainActor
final class ProfileViewController: UIViewController {
    // Tout le code ici s'exécute sur le main thread
}
```

### Annulation

Swift supporte l'annulation coopérative via `Task`. Vérifiez l'annulation dans les opérations longues.

```swift
func processItems(_ items: [Item]) async throws {
    for item in items {
        try Task.checkCancellation()  // lève CancellationError si annulé
        await process(item)
    }
}

// Annulation depuis l'extérieur
let task = Task {
    try await processItems(items)
}

// Plus tard :
task.cancel()
```

### Actors

Un `actor` protège son état interne contre les accès concurrents — l'équivalent d'un objet thread-safe sans locks manuels.

```swift
// ✅ — Accès concurrent sécurisé sans mutex
actor ScoreManager {
    private var scores: [String: Int] = [:]

    func addScore(_ score: Int, for player: String) {
        scores[player, default: 0] += score
    }

    func score(for player: String) -> Int {
        scores[player] ?? 0
    }
}

// L'appel depuis l'extérieur doit être async
let manager = ScoreManager()
await manager.addScore(10, for: "Alice")
```

---

## Flags de compilation

Activez les warnings stricts dans le projet Xcode :

- **Treat Warnings as Errors** : Build Settings → `SWIFT_TREAT_WARNINGS_AS_ERRORS = YES`
- **Strict Concurrency Checking** : Build Settings → `SWIFT_STRICT_CONCURRENCY = complete` (Swift 5.10+, prépare à Swift 6)

Pour supprimer un warning localement quand c'est inévitable :

```swift
// Suppression ciblée
#if compiler(>=5.9)
nonisolated(unsafe) var legacyCallback: (() -> Void)?
#endif
```

Préférez toujours corriger le warning plutôt que le supprimer.
