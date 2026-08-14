# Guidelines Générales

Ce document couvre les bonnes pratiques applicables à **tous les projets et langages** chez Headcrab.

Pour les spécificités par langage, consultez les fichiers dédiés :

- [C#](csharp.md)
- [Unity](unity.md)
- [Swift](swift.md)
- [API REST](api.md)

---

## Sommaire

- [Langue](#langue)
- [Nommage](#nommage)
- [Gestion des accolades](#gestion-des-accolades)
- [Indentation et Early Return](#indentation-et-early-return)
- [Taille des fonctions](#taille-des-fonctions)
- [Taille des classes](#taille-des-classes)
- [Gestion des espaces](#gestion-des-espaces)
- [Scope](#scope)
- [Booléens](#booléens)
- [Constantes et Magic Numbers](#constantes-et-magic-numbers)
- [Documentation](#documentation)
- [Commentaires](#commentaires)
- [Imports](#imports)
- [Programmation défensive](#programmation-défensive)
- [Strings](#strings)
- [Types abstraits](#types-abstraits)
- [Principes SOLID](#principes-solid)
- [Flags de compilation](#flags-de-compilation)
- [Async / Await](#async--await)
- [Git](#git)

---

## Langue

Sauf projet spécifique, **la langue de travail est l'anglais** : fichiers, classes, variables, fonctions, commentaires, messages de commit...

Pourquoi l'anglais ?

- Les bibliothèques, la documentation et les messages d'erreur sont quasi-systématiquement en anglais. Chercher une solution sur Google ou Stack Overflow est infiniment plus efficace en anglais.
- Un projet entièrement en anglais reste lisible pour des collaborateurs futurs, quelle que soit leur origine.
- Mélanger deux langues dans un même projet crée une incohérence qui nuit à la lisibilité et complique les recherches dans le code.

---

## Nommage

### Les noms doivent refléter précisément leur rôle

Un bon nom élimine le besoin de commentaire. Si tu dois commenter une variable pour expliquer ce qu'elle contient, c'est que son nom est mauvais.

```csharp
// ❌ Que contient `d` ? Que vaut `s` ? Pourquoi 86400 ?
var d = DateTime.Now;
if ((d - start).TotalSeconds > 86400) { ... }

// ✅ La logique est lisible sans commentaire
var currentTime = DateTime.Now;
const int K_SECONDS_PER_DAY = 86400;
if ((currentTime - sessionStartTime).TotalSeconds > K_SECONDS_PER_DAY) { ... }
```

On accepte les `i`, `j`, `k` pour les boucles for courtes — "Nous ne sommes pas des sauvages."

### Cohérence

Utilise toujours le même terme pour la même chose. Si tu nommes `userId` dans un endroit, ne nomme pas `accountId` ailleurs pour désigner la même donnée.

### Interfaces

Les interfaces représentent une relation **"can do"** : ce qu'une classe _peut faire_.

```csharp
ICanMove, ICanDie, ICanFight, ICanCollect
```

Le préfixe `I` majuscule est conservé pour distinguer rapidement une interface d'une classe de base dans une signature.

> Il existe un débat sur ce préfixe hérité de la notation hongroise.

> **Débat sur le nommage** : deux écoles s'affrontent.
>
> La convention classique préconise un suffixe `-able`, `-er` ou `-or` : `Serializable`, `Printable`, `IEnumerator`... Le problème, c'est que ces noms décrivent souvent une capacité générique sans préciser ce qu'elle fait concrètement dans le contexte du projet.
>
> Notre choix est de privilégier **la précision** : l'interface doit décrire exactement ce que la classe peut faire, pas une capacité abstraite. `ICanSerializeToJson` est plus clair que `ISerializable`. `ICanMove` dit exactement ce qu'il faut savoir.
>
> Nous acceptons d'autres conventions, du moment qu'elles sont **cohérentes** dans le projet et que le nom reste précis.

### Classes de base

Une classe dédiée à l'héritage porte le suffixe **Base**.

```csharp
UnitBase → FlightUnit, GroundUnit
CollectibleBase → Coin, Ammo, LifePack
```

**Faut-il reprendre le nom de la classe de base dans les héritiers ?** La règle est que le nom de la classe enfant doit être **auto-descriptif**. Inclure le nom de la base ou non est laissé à l'appréciation selon ce qui apporte le plus de clarté.

- `FlightUnit`, `GroundUnit` → garder "Unit" aide, il fait partie de l'identité de l'objet
- `Coin`, `Ammo`, `LifePack` → ajouter "Collectible" (`CoinCollectible`) serait redondant, le nom se suffit à lui-même

Le critère : **est-ce que le nom est clair sans contexte ?** Si oui, pas besoin de répéter la base.

### Collections

Les collections sont au **pluriel**.

```csharp
List<int> scores;
int[,] positions;
Dictionary<string, User> usersByEmail;
```

---

## Gestion des accolades

Sauf spécification du langage (JS, Swift…), les accolades s'ouvrent **sur la ligne suivante**.

Les accolades d'un `if` sont **toujours** requises, même pour une seule instruction.

```csharp
// ✅
private void MyFunction()
{
    if (myCondition)
    {
        DoSomething();
    }
    else
    {
        DoSomethingElse();
    }
}

// ❌ — On ajoute une instruction un an plus tard et le bug est discret
if (myCondition)
    DoSomething();
```

**Pourquoi les accolades en dessous ?** Les accolades ouvrante et fermante sont alignées verticalement, ce qui permet de voir d'un coup d'œil le début et la fin d'un bloc, sans avoir à chercher où il se termine.

**Pourquoi toujours les accolades ?** Deux raisons :

- **Lisibilité** : le bloc est clairement délimité, quelle que soit son contenu.
- **Sécurité** : ajouter une instruction dans un `if` sans accolades crée un bug silencieux qui ne génère aucune erreur de compilation.

---

## Indentation et Early Return

Évitez les `if` imbriqués. Préférez **sortir tôt** de la fonction quand une condition n'est pas remplie (early return / guard clause).

```csharp
// ❌ — Le lecteur doit garder en mémoire toutes les conditions imbriquées
private void ProcessOrder(Order order)
{
    if (order is not null)
    {
        if (order.IsValid)
        {
            if (order.Items.Count > 0)
            {
                // La vraie logique est ici, au fond de 3 niveaux
                Submit(order);
            }
        }
    }
}

// ✅ — On élimine les cas invalides au plus tôt, le cas nominal est au premier plan
private void ProcessOrder(Order order)
{
    if (order is null)
    {
        return;
    }

    if (!order.IsValid)
    {
        return;
    }

    if (order.Items.Count == 0)
    {
        return;
    }

    // La vraie logique est ici, lisible sans contexte imbriqué
    Submit(order);
}
```

**Pourquoi ?** Chaque niveau d'imbrication supplémentaire augmente la charge cognitive. Le lecteur doit garder en mémoire l'état de toutes les conditions ouvertes pour comprendre un bloc. Un seul niveau d'indentation est la cible ; deux niveaux sont acceptables ; au-delà, c'est un signal que la fonction doit être découpée.

---

## Taille des fonctions

Une fonction doit faire **une seule chose**. Si tu dois écrire "et" pour décrire ce que fait une fonction, elle en fait probablement trop.

Vise des fonctions qui tiennent sur un écran standard (~40 lignes). Au-delà, envisage de la décomposer en sous-fonctions nommées.

```csharp
// ❌ — Une fonction qui fait tout
private void HandleGameOver()
{
    // 20 lignes pour calculer les scores
    // 15 lignes pour sauvegarder en base
    // 10 lignes pour afficher l'écran de fin
}

// ✅ — Chaque responsabilité est isolée et nommée
private void HandleGameOver()
{
    var finalScore = ComputeFinalScore();
    SaveScoreToDatabase(finalScore);
    ShowGameOverScreen(finalScore);
}
```

Des fonctions courtes sont plus faciles à lire, à tester et à réutiliser.

---

## Taille des classes

Il n'y a pas de règle absolue, mais une classe qui dépasse ~500 lignes est souvent le signe qu'elle a trop de responsabilités. C'est un signal, pas une obligation de refactoring immédiat.

Interroge-toi : est-ce que ma classe pourrait être décrite en une phrase sans utiliser "et" ? Si non, elle fait probablement trop de choses.

> Ce principe est le **S** de SOLID : _Single Responsibility Principle_. Une classe, une responsabilité. Voir [Principes SOLID](#principes-solid).

---

## Gestion des espaces

Sauf spécification du langage :

- Un espace _entre_ les **opérateurs** : `a + b`, `x > y`
- Un espace _après_ une **virgule** : `MyFunction(a, b, c)`
- Un espace _après_ `if`, `while`, `for`, `switch`, `catch`
- Un espace _après_ l'ouverture et _avant_ la fermeture d'une **accolade** dans une lambda : `x => { return x + 1; }`
- **Pas** d'espace avant un **point-virgule**
- **Pas** d'espace à l'intérieur des **parenthèses** ou **crochets** : `array[0]`, `MyFunction(a)`
- **Pas** d'espace entre le nom d'une fonction et ses paramètres : `MyFunction(...)` pas `MyFunction (...)`

```csharp
// ✅
private void MyFunction(int param1, int param2)
{
    if (param1 > param2)
    {
        // ...
    }

    for (int i = 0; i < 10; i++)
    {
        // ...
    }

    var filtered = myList.Where(x => x > 2);
}
```

---

## Scope

Toute variable ou fonction qui n'a pas besoin d'être accessible depuis l'extérieur **doit être `private`**.

Toute classe qui n'est pas conçue pour être héritée **doit être scellée** (`sealed` en C#, `final` en Swift/Kotlin).

**Pourquoi ?** C'est le principe du moindre privilège : on n'expose que ce qui doit l'être. Une surface publique réduite signifie moins d'effets de bord possibles, moins de couplage involontaire, et un refactoring plus sûr — on peut modifier l'implémentation interne sans risquer de casser quelque chose ailleurs.

**Sur `sealed` en particulier** : sceller une classe par défaut, c'est un choix de conception explicite. Enlever `sealed` le jour où l'héritage devient nécessaire coûte une seconde. Découvrir qu'une classe non prévue pour l'héritage a été héritée dans un endroit inattendu peut coûter beaucoup plus.

> Ce principe est lié au **O** de SOLID : _Open/Closed Principle_. Une classe est **ouverte** à l'extension (via composition, interfaces, injection) mais **fermée** à la modification directe de son implémentation interne. `sealed` matérialise la partie "fermée". Voir [Principes SOLID](#principes-solid).

---

## Booléens

Préférez **toujours des booléens positifs**, même si ce booléen est systématiquement utilisé sous sa forme négative dans le code. Il vaut mieux écrire `!canShoot` à chaque fois que d'avoir un booléen négatif qui impose une double négation mentale.

```csharp
// ✅ — Le nom est positif, la négation est explicite et lisible
private bool canShoot = false;
if (!canShoot) { ... }  // "si je ne peux pas tirer" — clair

// ❌ — Même si `cannotFly` est toujours utilisé sous forme négative,
//      le résultat est une double négation qui force le cerveau à recalculer
private bool cannotFly = true;
if (!cannotFly) { ... } // "si je ne peux pas ne pas voler" 😵
                        // → Renommer en `canFly`
```

---

## Constantes et Magic Numbers

Ne laissez jamais de **nombres ou chaînes "magiques"** dans le code. Tout littéral qui a une signification métier ou technique doit être nommé.

```csharp
// ❌ — Que représente 86400 ? Pourquoi 3 ? Qu'est-ce que "ADM" ?
if (elapsedSeconds > 86400) { ... }
for (int i = 0; i < 3; i++) { ... }
if (role == "ADM") { ... }

// ✅ — Avec une constante nommée
private const int K_SECONDS_PER_DAY = 86_400;
private const int K_MAX_RETRY_COUNT = 3;
private const string K_ADMIN_ROLE = "ADM";

if (elapsedSeconds > K_SECONDS_PER_DAY) { ... }
for (int i = 0; i < K_MAX_RETRY_COUNT; i++) { ... }
if (role == K_ADMIN_ROLE) { ... }
```

Quand les valeurs représentent un **ensemble fini d'états**, préférez une **enum** plutôt qu'une constante — elle groupe les valeurs liées et empêche d'en passer une invalide.

```csharp
// ❌ — Des strings magiques, rien n'empêche une faute de frappe
if (role == "ADM") { ... }
if (role == "USR") { ... }

// ✅ — L'enum ferme l'ensemble des valeurs possibles
enum UserRole { Admin, User, Moderator }
if (role == UserRole.Admin) { ... }
```

**Un magic number qui nécessite un commentaire pour être compris est la preuve que le code n'est pas bon** — c'est le signal qu'il faut extraire une constante ou une enum. Voir [Commentaires](#commentaires).

**Pourquoi ?** Un magic number ne documente ni son origine ni son intention. Quelqu'un qui voit `86400` doit le calculer mentalement ; quelqu'un qui voit `K_SECONDS_PER_DAY` comprend immédiatement. Et quand la valeur change, il n'y a qu'un seul endroit à modifier.

---

## Documentation

Toute fonction **doit être documentée** via le système de documentation du langage. La documentation n'est pas un commentaire — c'est le **contrat** de la fonction.

L'IDE affiche cette documentation au survol, sans que le lecteur ait besoin d'ouvrir le fichier source. C'est le premier outil de compréhension d'une codebase.

**C# — Visual Studio génère le squelette avec `///`**

```csharp
/// <summary>
/// Determines whether the player can perform an action based on current state.
/// Returns false if the player is stunned, dead, or in a cutscene.
/// </summary>
/// <param name="action">The action to evaluate.</param>
/// <returns>True if the action can be performed, false otherwise.</returns>
private bool CanPerformAction(PlayerAction action)
{
    // ...
}
```

**Swift — Xcode génère le squelette via clic droit > Add Documentation**

```swift
/// Determines whether the player can perform an action based on current state.
/// - Parameter action: The action to evaluate.
/// - Returns: `true` if the action can be performed, `false` otherwise.
func canPerform(_ action: PlayerAction) -> Bool { ... }
```

**Python — docstring entre `"""`**

```python
def can_perform_action(action: PlayerAction) -> bool:
    """
    Determines whether the player can perform an action based on current state.

    Args:
        action: The action to evaluate.

    Returns:
        True if the action can be performed, False otherwise.
    """
    ...
```

La documentation décrit **ce que fait** la fonction, **ce qu'elle attend** et **ce qu'elle retourne** — pas comment elle le fait. L'implémentation peut changer ; le contrat doit rester stable.

---

## Commentaires

Un code clair se passe de commentaires inline. Si tu es tenté d'expliquer **ce que fait** une ligne, réécris-la pour qu'elle soit explicite.

Un commentaire inline explique **pourquoi**, jamais **quoi**.

```csharp
// ❌ — Décrit ce que fait le code, que le code dit déjà
counter++;  // Increment counter

// ✅ — Explique une décision non-évidente
counter++;  // Starts at 1: SAP does not support zero-based indexing

// ✅ — Documente une limitation technique
// Unity overrides == for destroyed GameObjects; don't use `is null` here
if (target == null) { ... }

// ✅ — Documente une règle métier complexe
// Session expires after 24h of inactivity, not 24h after creation (product decision — see #142)
if (elapsedSinceLastAction > K_SESSION_TIMEOUT) { ... }
```

**Les cas légitimes pour un commentaire inline :**

- Expliquer **pourquoi** un choix a été fait (et non un choix plus évident)
- Documenter une **règle métier** non triviale
- Documenter une **limitation technique** ou un contournement, idéalement avec un lien vers le ticket

**Si tu ressens le besoin d'ajouter un commentaire pour expliquer un magic number, c'est le signe qu'il faut le remplacer par une constante nommée.** Voir [Constantes et Magic Numbers](#constantes-et-magic-numbers).

**Les commentaires ne servent pas à versionner le code.** Supprimez le code commenté "au cas où". Git est là pour ça.

```csharp
// ❌ — Git conserve l'historique, ce bloc commenté ne fait qu'encombrer
// private void OldMethod()
// {
//     ...
// }
```

---

## Imports

- Supprimez les imports **non utilisés** — ils augmentent le bruit et, en C/C++, le temps de compilation.
- Triez les imports par **ordre alphabétique**.

**Pourquoi l'ordre alphabétique ?**

- Trouver rapidement si un import est déjà présent.
- Éviter les doublons.
- En **C/C++**, l'ordre des `#include` a un impact réel sur les temps de compilation (precompiled headers, self-sufficiency des headers). Une convention stable évite les réorganisations accidentelles qui cassent ces optimisations. En C# ou Swift, c'est avant tout une question de lisibilité.

La plupart des IDEs trient automatiquement :

- Visual Studio : `Ctrl + R, Ctrl + G`
- Xcode : via un plugin ou manuellement

---

## Programmation défensive

### Yoda conditions

Quand vous comparez avec une constante, placez la **constante en premier**.

```csharp
// ✅
if (100 == score) { ... }
if (null == user) { ... }

// ❌ — En C/C++, une faute de frappe assigne 100 à score et retourne toujours true
if (score = 100) { ... }
```

`100 = score` ne compile dans aucun langage et évite ce type de bug. C'est particulièrement critique en **C/C++** où `=` au lieu de `==` dans un `if` est un bug silencieux. En C# ou Swift les compilateurs et analyseurs avertissent de ce cas, mais la convention reste utile pour l'uniformité.

### Null checks

Vérifiez toujours vos références avant de les utiliser.

```csharp
// C# — préférez `is` et `is not` en dehors d'Unity
if (address is null || address.User is null)
{
    return null;
}

return address.User;
```

**Pourquoi `is null` plutôt que `== null` ?** L'opérateur `==` peut être surchargé — une classe peut redéfinir ce que signifie "être égal à null". `is null` est un pattern match qui vérifie l'identité référentielle pure, sans passer par une surcharge possible. C'est plus prévisible, et c'est la raison pour laquelle Unity impose `==` (il surcharge justement `==` pour gérer les GameObjects détruits — voir [unity.md](unity.md)).

### Validation des entrées

Toute donnée qui vient de l'extérieur est **non fiable par défaut** : saisie utilisateur, paramètres d'URL, lecture de fichier, réponse d'API tierce. Elle doit être validée avant d'être utilisée.

Validez **à l'entrée du système** — longueur, format, plage de valeurs, type. Plus la validation est proche de la source, moins il y a de risque de propager une donnée invalide dans le reste du code.

```csharp
// ❌ — On fait confiance à l'entrée sans vérification
public void SetPlayerName(string name)
{
    this.playerName = name;
}

// ✅ — On valide avant d'accepter
public void SetPlayerName(string name)
{
    if (string.IsNullOrWhiteSpace(name))
    {
        throw new ArgumentException("Player name cannot be empty.");
    }

    if (name.Length > K_MAX_NAME_LENGTH)
    {
        throw new ArgumentException($"Player name cannot exceed {K_MAX_NAME_LENGTH} characters.");
    }

    this.playerName = name;
}
```

> **Note sur le taint tracking** : un concept plus avancé consiste à "teinter" (_taint_) les données d'origine externe et à vérifier qu'elles sont assainies avant d'être utilisées dans des contextes sensibles (requêtes SQL, commandes système, affichage HTML...). C'est une pratique que nous souhaitons approfondir — cette section sera mise à jour.

---

## Strings

- Préférez les **string interpolations** aux concaténations : elles sont plus lisibles et plus performantes.

```csharp
// ✅
string message = $"Welcome back, {user.Name}! You have {inbox.Count} messages.";

// ❌
string message = "Welcome back, " + user.Name + "! You have " + inbox.Count + " messages.";
```

- Pour les **comparaisons**, utilisez `ToUpper()`, `ToLower()`, ou une comparaison `StringComparison` pour éviter les bugs liés à la casse.
- Pour tester une chaîne vide, préférez `.IsNullOrEmpty()` ou `.IsNullOrWhiteSpace()` à `str == ""` — le premier gère aussi `null`.
- Les chaînes de connexion, identifiants, clés d'API et autres valeurs de configuration ne doivent **jamais** être en dur dans le code. Utilisez des fichiers de configuration, des variables d'environnement ou un système de secrets.

```csharp
// ❌ — Ne jamais committer ça
private const string K_DB_PASSWORD = "superSecretPassword123";

// ✅
var dbPassword = Environment.GetEnvironmentVariable("DB_PASSWORD");
```

---

## Types abstraits

Pour les paramètres de fonctions et les valeurs de retour, **préférez le type le plus abstrait qui répond au besoin**.

```csharp
// ❌ — Expose une implémentation concrète, contraint l'appelant inutilement
public List<User> GetActiveUsers() { ... }
private void ProcessItems(List<Order> orders) { ... }

// ✅ — L'appelant n'a pas besoin de savoir que c'est une List
public IReadOnlyCollection<User> GetActiveUsers() { ... }
private void ProcessItems(IEnumerable<Order> orders) { ... }
```

**Pourquoi ?** Exposer `List<T>` comme type de retour donne implicitement accès à `Add`, `Remove`, `Clear`... même si l'appelant n'a qu'à itérer. Utiliser `IReadOnlyCollection<T>` ou `IEnumerable<T>` communique l'intention et empêche les modifications involontaires.

Ce principe s'applique aussi aux paramètres : une fonction qui n'a besoin que d'itérer n'a pas à imposer un `List<T>` — elle accepte n'importe quelle collection si on lui passe un `IEnumerable<T>`.

> Ce principe est lié au **L** de SOLID : _Liskov Substitution Principle_. Un sous-type doit pouvoir se substituer à son type parent sans casser le comportement attendu. En pratique : si tu dépends de `IEnumerable<T>` plutôt que de `List<T>`, n'importe quelle collection peut être passée — tableau, set, résultat de requête LINQ — sans changer le code appelant. Voir [Principes SOLID](#principes-solid).

---

## Principes SOLID

SOLID est un acronyme qui regroupe cinq principes de conception orientée objet. Ils apparaissent à plusieurs endroits de ce document et sont rappelés ici pour référence.

| Lettre | Principe                | En résumé                                                                                                                                                         |
| ------ | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **S**  | _Single Responsibility_ | Une classe = une responsabilité. Si elle fait plusieurs choses, elle doit être découpée.                                                                          |
| **O**  | _Open/Closed_           | Une classe est ouverte à l'**extension** (nouveaux comportements via composition ou interfaces), mais fermée à la **modification** de son implémentation interne. |
| **L**  | _Liskov Substitution_   | Un sous-type doit pouvoir remplacer son type parent sans casser le comportement attendu.                                                                          |
| **I**  | _Interface Segregation_ | Plusieurs interfaces spécifiques valent mieux qu'une interface générale. Une classe ne doit pas être forcée d'implémenter ce qu'elle n'utilise pas.               |
| **D**  | _Dependency Inversion_  | Dépendre des abstractions, pas des implémentations concrètes. Une classe ne devrait pas instancier ses dépendances — elle devrait les recevoir.                   |

Ces principes ne sont pas des règles à appliquer mécaniquement. Ce sont des guides pour identifier quand une conception peut être améliorée. Un code qui les respecte est généralement plus facile à tester, à faire évoluer et à comprendre.

---

## Flags de compilation

Tout nouveau projet doit être configuré pour que les **warnings soient traités comme des erreurs**.

Un warning ignoré aujourd'hui est un bug potentiel demain. En forçant l'erreur, on évite l'accumulation de dette technique : un warning non traité est bloquant, pas invisible.

Voir les instructions spécifiques dans [csharp.md](csharp.md) et [unity.md](unity.md).

---

## Async / Await

### Async jusqu'au bout

L'async se propage. Ne bloquez **jamais** du code asynchrone de manière synchrone avec `.Result`, `.Wait()` ou `GetAwaiter().GetResult()` — c'est la cause la plus courante de deadlock.

```csharp
// ❌ — Deadlock garanti dans un contexte synchronisé (UI thread, ASP.NET classic...)
var result = GetDataAsync().Result;
var result = GetDataAsync().GetAwaiter().GetResult();

// ✅
var result = await GetDataAsync();
```

Si une fonction appelle une fonction async, elle doit elle-même être async. Ne pas suivre cette règle revient à construire un barrage au milieu d'une rivière.

### async void est interdit (sauf event handlers)

`async void` avale les exceptions silencieusement — elles ne peuvent pas être catchées par l'appelant.

```csharp
// ❌ — Si une exception est levée, elle est perdue
private async void LoadData() { ... }

// ✅
private async Task LoadData() { ... }

// ✅ — Seule exception légitime : les event handlers
private async void OnButtonClick(object sender, EventArgs e) { ... }
```

### Fire-and-forget : à éviter

Lancer une tâche sans l'awaiter fait disparaître silencieusement toute exception qu'elle lève.

```csharp
// ❌ — Exception silencieusement perdue
_ = SendNotificationAsync(user);

// ✅ — Si on ne peut pas awaiter, au minimum logguer l'échec
_ = SendNotificationAsync(user).ContinueWith(
    t => logger.LogError(t.Exception, "Notification send failed"),
    TaskContinuationOptions.OnlyOnFaulted
);
```

### Annulation coopérative

Les opérations longues doivent supporter l'annulation. L'appelant doit pouvoir demander l'arrêt propre d'une opération en cours, sans l'interrompre brutalement.

Le mécanisme est spécifique au langage — voir [csharp.md](csharp.md) pour `CancellationToken` et [swift.md](swift.md) pour la cancellation des `Task`.

### Nommage des méthodes async

Le suffixe et les conventions de nommage pour les méthodes async sont spécifiques au langage — voir [csharp.md](csharp.md) et [swift.md](swift.md).

---

## Git

### Branches

Les noms de branches sont en **kebab-case** (minuscules, mots séparés par des tirets).

Préfixez vos branches selon leur nature :

| Préfixe             | Usage                                       | Exemple                             |
| ------------------- | ------------------------------------------- | ----------------------------------- |
| `feature/`          | Nouvelle fonctionnalité                     | `feature/player-dash`               |
| `fix/` ou `bugfix/` | Correction de bug planifiée                 | `bugfix/123-score-overflow`         |
| `hotfix/`           | Correction urgente en production            | `hotfix/crash-on-startup`           |
| `refactor/`         | Refactoring sans changement de comportement | `refactor/cleanup-inventory-system` |
| `docs/`             | Documentation uniquement                    | `docs/update-api-guidelines`        |
| `chore/`            | Tâches techniques (CI, dépendances…)        | `chore/update-dependencies`         |

**Pourquoi ?** On identifie en un coup d'œil la nature et la priorité d'une branche. Un `hotfix/` signale immédiatement une urgence ; un `fix/` référencé avec le numéro d'issue permet de relier le commit au problème rapporté.

**Règles :**

- On ne pousse jamais directement sur `main` ou `develop`. Toujours passer par une PR.
- Une branche = une tâche. Ne mélangez pas feature et fix dans la même branche.
- Une PR doit rester **lisible** : personne n'a envie de reviewer 200 fichiers en une fois. Découpez les grosses features en PR successives si nécessaire.

### Commits

Les messages de commit doivent être **courts et précis**, écrits en anglais, à l'**impératif**.

```
// ✅
Add player dash ability
Fix score overflow on level completion
Remove unused AudioManager references

// ❌
fix
wip
changes
Correction du bug du score (trop vague + mauvaise langue)
```

Un bon message de commit répond à la question : _"Si je merge ce commit, il va… [message]."_

Pour les commits complexes, utilisez une description longue :

```
Add player dash ability

Dash is triggered by double-tapping the move direction.
Cooldown is configurable via the inspector (default: 1.5s).
Dash ignores collision with enemies but not with walls.

Closes #42
```

### Pull Requests

- Si une PR n'est pas prête à être reviewée, préfixez son titre avec **`[WIP]`** ou passez-la en **Draft**.
- Le titre de la PR suit le même format que les commits : court, précis, impératif.
- Liez la PR à l'issue correspondante (`Closes #XX`).
- En tant qu'**auteur** : relisez votre propre PR avant de demander une review. Annotez les parties qui nécessitent une attention particulière ou qui sont des choix délibérés.

**En tant que reviewer :**

- Vérifiez que le code respecte les guidelines du projet. C'est l'un des objectifs explicites de la code review.
- Les remarques portent sur le **code**, pas sur la personne.
- Distinguez ce qui est **bloquant** de ce qui est une **suggestion** — un préfixe comme `[bloquant]` ou `[nit]` évite les ambiguïtés.
- **N'hésitez pas à signaler tout ce qui ne va pas, même s'il y a beaucoup à dire.** Une review complète fait progresser le développeur. Ce n'est pas une attaque personnelle — c'est ce qui fait monter le niveau collectif. Une remarque passée sous silence pour ne pas vexer est un problème qui reviendra.

### Ce qu'on ne commite jamais

- Des **secrets** : clés d'API, mots de passe, tokens, chaînes de connexion
- Des **fichiers de configuration locale** : `.env`, fichiers d'IDE personnels (`.idea/`, `.vscode/` si non partagé intentionnellement)
- Des **artefacts de build** : `bin/`, `obj/`, fichiers générés

En cas de commit accidentel d'un secret, **révoquer la clé immédiatement** — effacer le commit ne suffit pas si la branche a été poussée.
