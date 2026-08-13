# Guidelines C#

Ce document couvre les bonnes pratiques spécifiques au langage C#.

Consultez également [general.md](general.md) pour les pratiques communes à tous les langages.
Pour les spécificités Unity, voir [unity.md](unity.md).

---

## Sommaire

- [Convention de nommage](#convention-de-nommage)
- [var](#var)
- [Propriétés et Getters / Setters](#propriétés-et-getters--setters)
- [Null et pattern matching](#null-et-pattern-matching)
- [Opérateurs null-conditionnels](#opérateurs-null-conditionnels)
- [IDisposable et using](#idisposable-et-using)
- [Async / Await](#async--await)
- [LINQ](#linq)
- [Flags de compilation](#flags-de-compilation)

---

## Convention de nommage

| Élément                | Convention                  | Exemple                                  |
| ---------------------- | --------------------------- | ---------------------------------------- |
| Classe                 | PascalCase                  | `class PlayerController`                 |
| Classe de base         | PascalCase + suffixe `Base` | `class UnitBase`                         |
| Classe scellée         | PascalCase (pas de suffixe) | `sealed class Coin`                      |
| Interface              | `I` + PascalCase            | `interface ICanMove`                     |
| Enum                   | `E` + PascalCase            | `enum EGameState`                        |
| Valeur d'enum          | PascalCase                  | `EGameState.MainMenu`                    |
| Méthode                | PascalCase                  | `void ComputeScore()`                    |
| Propriété publique     | PascalCase                  | `public int Score { get; private set; }` |
| Variable membre privée | camelCase                   | `private int currentScore`               |
| Variable locale        | camelCase                   | `var elapsedTime = 0f`                   |
| Constante              | `K_` + UPPER_SNAKE_CASE     | `const int K_MAX_PLAYERS = 4`            |
| Collection             | Pluriel                     | `List<Enemy> enemies`                    |
| Méthode async          | Suffixe `Async`             | `Task<User> GetUserAsync()`              |

### Pourquoi le préfixe `K_` pour les constantes ?

`K_` (du grec _konstante_) distingue visuellement une constante d'une variable dans le code, sans avoir à aller vérifier sa déclaration. En C#, `const` et `static readonly` ont des comportements différents — le préfixe rappelle qu'il s'agit d'une valeur figée, quelle que soit la forme de déclaration.

### Pourquoi le préfixe `E` pour les enums ?

Une valeur d'enum sans son préfixe de type peut être ambiguë dans certains contextes. `EGameState.MainMenu` indique explicitement qu'on manipule un état de jeu, pas un entier ou un string.

---

## var

`var` est autorisé **uniquement quand le type est littéralement écrit sur la même ligne**, c'est-à-dire lors d'une construction `new` ou d'un cast explicite.

```csharp
// ✅ — Le type est écrit à droite, var évite la répétition
var player = new Player();
var enemies = new List<Enemy>();
var config = (GameConfig)rawConfig;

// ❌ — Le type n'est pas visible sans ouvrir la définition de la méthode
var user = GetCurrentUser();
var components = service.GetComponents();

// ✅ — Type explicite : lisible sans IDE, lisible en code review
User user = GetCurrentUser();
List<ComponentDto> components = service.GetComponents();
```

**Pourquoi cette règle ?** La règle "var quand c'est évident" est subjective : ce qui est évident pour l'auteur ne l'est pas forcément pour le reviewer, ni pour quelqu'un qui lit la PR sur GitHub sans hover d'IDE. En limitant `var` aux cas où le type est écrit sur la même ligne, la règle devient objective et vérifiable en deux secondes.

`var` ne s'utilise que pour les **variables locales**. Les membres de classe, les paramètres et les types de retour sont toujours déclarés avec leur type explicite.

---

## Propriétés et Getters / Setters

### Auto-properties

Pour un accès simple à un membre (lire ou écrire sans logique additionnelle), utilisez les **auto-properties** de C# :

```csharp
// ✅ — Lecture publique, écriture privée
public int Score { get; private set; }

// ✅ — Lecture seule depuis l'extérieur
public string PlayerName { get; private set; }

// ✅ — Read-only (assignable uniquement dans le constructeur)
public Guid Id { get; }
```

### `private set` vs `init`

C# propose deux façons de restreindre l'écriture d'une propriété :

- **`private set`** : la propriété peut être modifiée par la classe elle-même à tout moment de son cycle de vie.
- **`init`** (C# 9+) : la propriété ne peut être assignée **qu'à la construction** de l'objet (constructeur ou object initializer). Elle devient immuable après.

```csharp
public class Player
{
    // init — assigné une fois à la création, immuable ensuite
    public Guid Id { get; init; }
    public string Name { get; init; }

    // private set — peut être modifié pendant la durée de vie de l'objet
    public int Score { get; private set; }
    public bool IsAlive { get; private set; }
}

// À la construction : init et private set sont tous les deux assignables
var player = new Player
{
    Id = Guid.NewGuid(),   // ✅ init
    Name = "Alice",        // ✅ init
};

// Après construction :
player.Score = 100;        // ✅ private set, modifiable depuis l'intérieur de la classe
player.Id = Guid.NewGuid(); // ❌ erreur de compilation — init-only
```

**Règle** : si une propriété n'a pas besoin de changer après construction, utilisez `init`. C'est un contrat explicite d'immuabilité qui le compilateur fait respecter.

### `const` vs `static readonly`

Ces deux formes déclarent des valeurs fixes, mais elles ont des comportements différents :

| | `const` | `static readonly` |
|---|---|---|
| Évaluation | Compile-time | Runtime (premier accès à la classe) |
| Types autorisés | Primitifs, `string`, `enum` | N'importe quel type |
| Peut être calculé | Non | Oui |
| Risque en lib partagée | ⚠️ Valeur inlinée | ✅ Valeur lue à l'exécution |

```csharp
// const — pour les valeurs primitives simples qui ne changeront jamais
private const int K_MAX_PLAYERS = 4;
private const string K_DEFAULT_SAVE_SLOT = "slot_0";

// static readonly — pour les types complexes ou les valeurs calculées
private static readonly TimeSpan K_SESSION_TIMEOUT = TimeSpan.FromHours(24);
private static readonly Vector3 K_DEFAULT_GRAVITY = new Vector3(0f, -9.81f, 0f);
private static readonly IReadOnlyList<string> K_VALID_ROLES = new[] { "Admin", "User", "Moderator" };
```

**Le risque de `const` en bibliothèque partagée** : si une constante est définie dans une lib et consommée par un autre assembly, sa valeur est **copiée à la compilation** dans l'assembly consommateur. Si tu changes la constante dans la lib et ne recompiles pas le consommateur, celui-ci utilisera encore l'ancienne valeur. `static readonly` n'a pas ce problème — la valeur est lue à l'exécution depuis l'assembly qui la définit.

**Règle** : `const` pour les primitifs simples qui ne changeront jamais et qui ne sont pas exportés dans une lib publique. `static readonly` pour tout le reste.

### Traitement supplémentaire

Faire du traitement dans un getter ou un setter est une **mauvaise pratique** : le traitement est caché à l'appelant, qui s'attend à un simple accès.

Si un traitement est nécessaire lors de l'assignation, exposez une méthode dédiée dont le nom décrit ce qu'elle fait, et gardez le getter en auto-property.

```csharp
// ❌ — Le setter cache un comportement non-évident
public int Health
{
    get => health;
    set => health = Math.Clamp(value, 0, maxHealth); // L'appelant ne s'y attend pas
}

// ✅ — Le traitement est dans une méthode explicitement nommée
public int Health { get; private set; }

public void TakeDamage(int amount)
{
    Health = Math.Clamp(Health - amount, 0, maxHealth);
}
```

### Visibilité

Par défaut, tout membre est `private`. On indique **toujours le mot-clé explicitement** — ne pas le laisser implicite même si C# l'infère par défaut.

```csharp
// ❌ — Implicitement private, mais pas explicite
int score = 0;

// ✅
private int score = 0;
```

**Pourquoi ?** Deux raisons liées à la lisibilité :
- En lisant `private int score`, on sait immédiatement et sans contexte que ce membre est privé. Sans le mot-clé, il faut connaître les règles de visibilité par défaut de C# — une connaissance qu'on ne peut pas supposer chez tous les lecteurs.
- Dans un bloc de déclarations mixtes (certaines `public`, d'autres `private`, d'autres `protected`), l'absence du mot-clé crée une asymétrie visuelle qui ralentit la lecture.

---

## Null et pattern matching

En C# moderne, préférez `is null` et `is not null` à `== null` et `!= null` pour les null checks.

```csharp
// ✅
if (user is null) { return; }
if (address is not null) { ... }

// Acceptable, mais moins expressif
if (user == null) { return; }
```

**Pourquoi `is null` ?** L'opérateur `==` peut être surchargé — une classe peut redéfinir ce que signifie "être égal à null". `is null` est un pattern match qui vérifie l'identité référentielle pure, sans passer par aucune surcharge. Le résultat est toujours prévisible.

> ⚠️ **Exception Unity** : avec les `MonoBehaviour` et les objets Unity en général, utilisez **toujours `== null`**. Unity surcharge `==` pour gérer les GameObjects détruits — `is null` ne prend pas en compte cette surcharge et peut produire des résultats incorrects. Voir [unity.md](unity.md).

### Pattern matching étendu

Le pattern matching va au-delà du null check — il permet de tester le type et d'extraire la valeur en une seule opération :

```csharp
// ✅ — Test de type + cast en une ligne
if (entity is ICanFight fighter)
{
    fighter.InflictDamage(this);
}

// Équivalent verbeux sans pattern matching
if (entity is ICanFight)
{
    var fighter = (ICanFight)entity;
    fighter.InflictDamage(this);
}

// ✅ — Switch expression avec patterns
string Describe(Shape shape) => shape switch
{
    Circle c   => $"Circle with radius {c.Radius}",
    Rectangle r => $"Rectangle {r.Width}x{r.Height}",
    _           => "Unknown shape"
};
```

---

## Opérateurs null-conditionnels

C# propose trois opérateurs pour travailler avec des valeurs potentiellement nulles :

| Opérateur | Nom                        | Usage                                            |
| --------- | -------------------------- | ------------------------------------------------ |
| `?.`      | Null-conditional           | Accès à un membre sans lever d'exception si null |
| `??`      | Null-coalescing            | Valeur par défaut si null                        |
| `??=`     | Null-coalescing assignment | Assigne uniquement si null                       |

```csharp
// ?. — court-circuite si user est null, retourne null sans exception
var name = user?.Profile?.DisplayName;

// ?? — valeur de repli
var displayName = user?.Name ?? "Guest";

// ??= — on ne set cache que si la variable est null au moment de l'appel. Sinon cette ligne est "ignorée"
cache ??= new Dictionary<string, object>();
```

> ⚠️ **Ces trois opérateurs ne doivent jamais être utilisés avec des `MonoBehaviour` ou tout objet Unity.** Ils utilisent l'opérateur `==` standard, pas la surcharge Unity. Un GameObject détruit peut passer les checks sans être réellement valide. Voir [unity.md](unity.md).

---

## IDisposable et using

Tout objet qui implémente `IDisposable` (connexions réseau, flux de fichiers, contextes de BDD…) doit être libéré explicitement. Utilisez systématiquement un bloc `using`.

```csharp
// ✅ — using block : Dispose() est appelé automatiquement, même en cas d'exception
using (var connection = new SqlConnection(connectionString))
{
    connection.Open();
    // ...
}

// ✅ — using declaration (C# 8+) : plus concis, Dispose() à la fin du scope
using var stream = File.OpenRead(filePath);
var content = ReadAll(stream);
// stream.Dispose() est appelé ici automatiquement
```

Ne jamais laisser un `IDisposable` sans `using` ou appel manuel à `Dispose()` — c'est une fuite de ressource garantie.

Si votre classe possède des membres `IDisposable`, implémentez vous-même `IDisposable` et libérez-les dans `Dispose()`.

---

## Async / Await

### Nommage

Toute méthode async doit porter le suffixe **`Async`**.

```csharp
public async Task<User> GetUserAsync(int userId) { ... }
public async Task SaveOrderAsync(Order order) { ... }
public async Task<bool> ValidateTokenAsync(string token) { ... }
```

**Pourquoi ?** Le suffixe signale à l'appelant qu'il doit `await` la méthode. Sans lui, on peut oublier l'`await` et récupérer une `Task` à la place du résultat — erreur silencieuse si le compilateur ne l'attrape pas.

### CancellationToken

Les opérations longues ou réseau doivent accepter un `CancellationToken` pour permettre l'annulation propre de la tâche.

```csharp
// ✅ — Paramètre optionnel avec valeur par défaut
public async Task<Report> GenerateReportAsync(
    ReportParameters parameters,
    CancellationToken cancellationToken = default)
{
    var data = await FetchDataAsync(parameters, cancellationToken);
    cancellationToken.ThrowIfCancellationRequested();
    return BuildReport(data);
}
```

Propagez toujours le token aux appels async internes — un token non propagé est un token inutile.

**Pourquoi ?** Sans mécanisme d'annulation, une tâche lancée continue de consommer des ressources même si son résultat n'est plus utile (l'utilisateur a annulé, la requête HTTP a expiré, l'écran a été fermé...). `CancellationToken` implémente l'annulation **coopérative** : la tâche vérifie régulièrement si elle doit s'arrêter, et se termine proprement.

### ConfigureAwait

Dans le code **applicatif** (Unity, WPF, WinForms), ne pas utiliser `ConfigureAwait(false)` — le contexte de synchronisation est nécessaire pour accéder à l'UI thread après un `await`.

Dans le code de **bibliothèque** (une lib partagée sans contexte UI), utilisez `ConfigureAwait(false)` pour éviter de capturer le contexte inutilement et réduire le risque de deadlock.

```csharp
// Dans une lib partagée
public async Task<string> FetchDataAsync(string url)
{
    var response = await httpClient.GetAsync(url).ConfigureAwait(false);
    return await response.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

---

## LINQ

LINQ est puissant mais peut nuire à la lisibilité et aux performances s'il est mal utilisé.

### Préférez la syntaxe fluent aux query expressions

```csharp
// ✅ — Syntaxe fluent : cohérente avec le reste du code C#
var activeAdmins = users
    .Where(user => user.IsActive)
    .Where(user => user.Role == UserRole.Admin)
    .OrderBy(user => user.Name)
    .ToList();

// ❌ — Query syntax : moins cohérente, plus verbeuse pour les cas simples
var activeAdmins = (from u in users
                    where u.IsActive && u.Role == UserRole.Admin
                    orderby u.Name
                    select u).ToList();
```

### Matérialisez au bon moment

Un `IEnumerable<T>` LINQ n'est évalué qu'au moment où il est parcouru (lazy evaluation). Si vous allez parcourir la collection plusieurs fois, matérialisez-la une seule fois avec `.ToList()` ou `.ToArray()`.

```csharp
// ❌ — La requête est exécutée deux fois
var results = items.Where(x => x.IsValid);
var count = results.Count();    // 1ère évaluation
var first = results.First();    // 2ème évaluation

// ✅ — La requête est exécutée une seule fois
var results = items.Where(x => x.IsValid).ToList();
var count = results.Count;
var first = results[0];
```

### Préférez un Dictionary à un `.Where()` dans une boucle

Chercher un élément avec `.Where()` ou `.FirstOrDefault()` à l'intérieur d'une boucle est une opération **O(n²)** : pour chaque élément de la boucle externe, on parcourt toute la collection interne.

Si vous avez besoin de faire des lookups répétés dans une collection, construisez un `Dictionary` une seule fois avant la boucle pour passer à **O(n)**.

```csharp
// ❌ — O(n²) : pour chaque order, on parcourt toute la liste users
foreach (var order in orders)
{
    var user = users.FirstOrDefault(u => u.Id == order.UserId);
    ProcessOrder(order, user);
}

// ✅ — O(n) : index construit une fois, lookup en O(1)
var userById = users.ToDictionary(u => u.Id);

foreach (var order in orders)
{
    if (userById.TryGetValue(order.UserId, out var user))
    {
        ProcessOrder(order, user);
    }
}
```

`TryGetValue` est préféré à l'indexeur `dict[key]` car il ne lève pas d'exception si la clé est absente.

### Évitez les LINQ dans les boucles chaudes

Dans les chemins critiques (boucle de jeu, traitement de masse), préférez des boucles `for`/`foreach` classiques — elles évitent les allocations liées aux délégués et aux enumerators LINQ.

---

## Flags de compilation

Configurez tout nouveau projet C# pour traiter les **warnings comme des erreurs**.

Dans le fichier `.csproj` :

```xml
<PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

**`Nullable enable`** active les annotations de nullabilité — le compilateur avertit quand une référence nullable est utilisée sans vérification. C'est l'un des outils les plus efficaces pour éliminer les `NullReferenceException` à la compilation plutôt qu'au runtime.

Si un warning ne peut pas être corrigé (dépendance tierce, API obsolète imposée), supprimez-le **localement** avec `#pragma warning disable` et **restaurez-le immédiatement après**. Ne jamais désactiver globalement.

```csharp
#pragma warning disable CS0612 // Obsolete
    LegacyApi.DoSomething();
#pragma warning restore CS0612
```

Pour Unity spécifiquement, voir [unity.md](unity.md) pour la configuration via `csc.rsp`.
