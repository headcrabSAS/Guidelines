# Guidelines Unity

Ce document couvre les bonnes pratiques spécifiques à Unity.

Consultez également :
- [general.md](general.md) — pratiques communes à tous les langages
- [csharp.md](csharp.md) — spécificités C#

---

## Sommaire

- [Setup d'un projet](#setup-dun-projet)
- [Architecture](#architecture)
- [Cycle de vie](#cycle-de-vie)
- [Null, == et is](#null--et-is)
- [Opérateurs null-conditionnels](#opérateurs-null-conditionnels)
- [Inspecteur](#inspecteur)
- [Performance](#performance)
- [Coroutines](#coroutines)
- [Git et Unity](#git-et-unity)
- [Build et compilation](#build-et-compilation)

---

## Setup d'un projet

Quand vous clonez ou créez un projet Unity, vérifiez que le fichier [setup.py](https://gist.github.com/Marsgames/1d4d745d9220b6f5e85dbb3724ea950e) est présent et exécutez-le **avant toute chose**. Il configure l'environnement de développement (template de script, paramètres de projet, etc.).

Téléchargez également le [template de script](https://gist.github.com/headcrab-company/2cc1e4679795422e16c6b6edbbdcb97d) et remplacez le script `81-C#` par défaut :
- **Windows** : `C:\Program Files\Unity\Editor\Data\Resources\ScriptTemplates`
- **Mac** : `/Applications/Unity/Unity.app/Contents/Resources/ScriptTemplates`

N'oubliez pas de mettre à jour la partie `Author` du template.

---

## Architecture

### MonoBehaviour vs classes C# pures

Un `MonoBehaviour` n'est pas juste une classe : c'est un composant Unity avec un cycle de vie géré par le moteur, qui s'attache à un `GameObject` et qui a un coût de performance non-négligeable.

**Ne créez pas de `MonoBehaviour` par défaut.** Posez-vous d'abord la question : est-ce que cette classe a besoin d'être un composant Unity ?

- Logique métier, data, utilitaires → **classe C# pure**
- Doit s'attacher à un GameObject, réagir aux événements Unity, être visible dans l'inspecteur → **MonoBehaviour**

```csharp
// ❌ — Un MonoBehaviour juste pour calculer un score
public class ScoreCalculator : MonoBehaviour
{
    public int ComputeScore(int baseScore, float multiplier) { ... }
}

// ✅ — Une classe C# pure, testable, sans overhead Unity
public sealed class ScoreCalculator
{
    public int ComputeScore(int baseScore, float multiplier) { ... }
}
```

### Singleton (GameManager)

Deux implémentations de GameManager sont disponibles [en suivant ce lien](https://gist.github.com/Marsgames/f5808a289f2ee135ac08930937787c68) :
- La première est un singleton classique avec une propriété `Instance` publique.
- La deuxième ajoute le minimum requis pour intégrer des publicités.

Le singleton doit rester **exceptionnel**. Avant d'en créer un, demandez-vous si l'injection de dépendance ne serait pas plus appropriée. Un singleton crée un couplage global difficile à tester.

### ScriptableObjects

Les `ScriptableObject` sont idéaux pour stocker des **données de configuration** partagées entre plusieurs GameObjects, sans passer par un singleton ni dupliquer des valeurs dans l'inspecteur.

```csharp
// ✅ — Données de configuration partagées via ScriptableObject
[CreateAssetMenu(fileName = "EnemyConfig", menuName = "Headcrab/EnemyConfig")]
public sealed class EnemyConfig : ScriptableObject
{
    public float MoveSpeed = 3f;
    public int MaxHealth = 100;
    public float AttackRange = 2f;
}
```

Avantages : modifiable dans l'éditeur, partageable entre prefabs, versionnable, rechargeable à chaud.

---

## Cycle de vie

L'ordre d'exécution des méthodes Unity sur un MonoBehaviour actif :

```
Awake → OnEnable → Start → Update / FixedUpdate / LateUpdate → OnDisable → OnDestroy
```

| Méthode | Quand | Usage recommandé |
|---|---|---|
| `Awake` | À l'instanciation, avant `Start`, même si le composant est désactivé | Initialisation interne, `GetComponent`, références self-contained |
| `OnEnable` | À chaque activation du composant | S'abonner aux événements |
| `Start` | Premier frame où le composant est actif, après tous les `Awake` | Initialisation qui dépend d'autres composants |
| `Update` | Chaque frame | Logique frame-by-frame, input |
| `FixedUpdate` | Intervalle fixe (indépendant du framerate) | Physique, `Rigidbody` |
| `LateUpdate` | Après tous les `Update` | Caméra, logique qui dépend des positions finales |
| `OnDisable` | À chaque désactivation | Se désabonner des événements |
| `OnDestroy` | À la destruction | Cleanup final |

**Règle pratique** : ce qui ne dépend que de soi-même va dans `Awake`. Ce qui dépend d'autres composants va dans `Start`. Les abonnements aux événements vont dans `OnEnable`/`OnDisable` pour éviter les appels vers des objets détruits.

---

## Null, `==` et `is`

En C# standard, `is null` est préféré à `== null` — voir [csharp.md](csharp.md).

**Unity est une exception.** Utilisez **toujours `== null`** avec les `MonoBehaviour`, `GameObject`, et tout objet qui hérite de `UnityEngine.Object`.

**Pourquoi ?** Unity surcharge l'opérateur `==` pour ses objets. Quand un `GameObject` est détruit, Unity le marque comme "mort" en interne — mais l'objet C# existe encore en mémoire jusqu'au prochain GC. `myObject == null` retourne `true` dans ce cas. `myObject is null` ne passe pas par cette surcharge et retourne `false` — l'objet semble vivant alors qu'il est détruit.

```csharp
// ✅ — Toujours avec les objets Unity
if (target == null) { return; }
if (gameObject == null) { return; }

// ❌ — Ne détecte pas les GameObjects détruits
if (target is null) { return; }

// ✅ — is peut être utilisé pour du pattern matching sur des types non-Unity
if (other is ICanFight fighter)
{
    fighter.InflictDamage(this);
}
```

---

## Opérateurs null-conditionnels

Les opérateurs `?.`, `??` et `??=` sont **interdits avec les MonoBehaviour et tout objet Unity**.

Ils utilisent l'opérateur `==` standard du CLR — pas la surcharge Unity. Un `GameObject` détruit peut passer leurs vérifications et provoquer des comportements indéfinis.

```csharp
// ❌ — Dangereux avec les objets Unity
var name = target?.name;
target ??= FindObjectOfType<Enemy>();

// ✅ — Vérification explicite avec ==
if (target == null)
{
    target = FindObjectOfType<Enemy>();
}

var name = target == null ? string.Empty : target.name;
```

Ces opérateurs restent utilisables sur des objets C# purs (classes métier, structs, etc.) qui ne dérivent pas de `UnityEngine.Object`.

---

## Inspecteur

### SerializeField

Les variables affichées dans l'inspecteur doivent être **`[SerializeField]` + `private`**, jamais `public`.

```csharp
// ✅
[SerializeField] private float moveSpeed = 5f;
[SerializeField] private Transform spawnPoint;

// ❌ — public expose inutilement la variable au code extérieur
public float moveSpeed = 5f;
```

**Pourquoi pas `public` ?** Une variable `public` dans Unity est automatiquement sérialisée — elle s'affiche dans l'inspecteur. Mais elle donne aussi accès en lecture ET en écriture à n'importe quelle autre classe. `[SerializeField] private` donne la visibilité dans l'inspecteur sans exposer la variable au reste du code.

Une variable sérialisée privée **doit toujours être initialisée à sa déclaration** (même à `null`, `0`, `false`...) pour éviter les warnings de compilation traités comme erreurs.

```csharp
[SerializeField] private int maxHealth = 0;
[SerializeField] private AudioClip hitSound = null;
```

### Visibility explicite

Même si `private` est implicite, on l'indique toujours — y compris pour les membres sérialisés. Voir [csharp.md](csharp.md).

### Organisation de l'inspecteur

Utilisez `[Header]` pour regrouper visuellement les variables dans l'inspecteur.

```csharp
[Header("Movement")]
[SerializeField] private float moveSpeed = 5f;
[SerializeField] private float jumpForce = 8f;

[Header("Combat")]
[SerializeField] private int maxHealth = 100;
[SerializeField] private float attackCooldown = 0.5f;
```

### Tooltip

Utilisez `[Tooltip]` pour documenter une variable directement dans l'inspecteur — au survol, Unity affiche la description.

```csharp
[Tooltip("Durée en secondes avant que le joueur récupère de la vie.\nValeur par défaut : 3")]
[SerializeField] private float healthRegenDelay = 3f;
```

### Range

Utilisez `[Range(min, max)]` pour créer un slider dans l'inspecteur. Applicable sur `int`, `float` et `double`.

```csharp
[Range(1, 10)]
[SerializeField] private int difficultyLevel = 3;

[Range(0f, 1f)]
[SerializeField] private float musicVolume = 0.8f;
```

### RequireComponent

Utilisez `[RequireComponent]` quand votre script dépend d'un autre composant. Unity empêchera la suppression du composant requis et l'ajoutera automatiquement si absent.

```csharp
[RequireComponent(typeof(Rigidbody))]
[RequireComponent(typeof(Collider))]
public sealed class PlayerMovement : MonoBehaviour
{
    private Rigidbody rb;

    private void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }
}
```

---

## Performance

### Cachez vos références dans Awake

`GetComponent<T>()`, `Find()`, `FindObjectOfType<T>()` sont des opérations coûteuses. Ne les appelez **jamais dans `Update`, `FixedUpdate` ou `LateUpdate`**.

```csharp
// ❌ — GetComponent appelé chaque frame
private void Update()
{
    GetComponent<Rigidbody>().AddForce(Vector3.up);
}

// ✅ — Référence cachée dans Awake, réutilisée chaque frame
private Rigidbody rb;

private void Awake()
{
    rb = GetComponent<Rigidbody>();
}

private void FixedUpdate()
{
    rb.AddForce(Vector3.up);
}
```

### Évitez les API string-based

`GameObject.Find("Player")`, `CompareTag("Enemy")` via string, `SendMessage()`... Ces API sont lentes, sujettes aux fautes de frappe, et ne produisent aucune erreur de compilation si le nom change.

```csharp
// ❌ — Lent, fragile, aucune garantie à la compilation
GameObject player = GameObject.Find("Player");
if (other.CompareTag("Enemy")) { ... }

// ✅ — Référence directe ou tag via constante
[SerializeField] private GameObject player;

private const string K_TAG_ENEMY = "Enemy";
if (other.CompareTag(K_TAG_ENEMY)) { ... }
```

### Évitez FindObjectOfType au runtime

`FindObjectOfType<T>()` parcourt tous les objets de la scène. Acceptable dans `Awake` ou `Start` — **jamais dans une boucle ou un Update**.

---

## Coroutines

Les coroutines sont le mécanisme natif de Unity pour les opérations étalées dans le temps. Elles s'exécutent sur le thread principal et sont liées au cycle de vie du GameObject qui les héberge.

```csharp
// ✅ — Démarrage et arrêt propres
private Coroutine regenCoroutine;

private void StartRegen()
{
    if (regenCoroutine != null)
    {
        StopCoroutine(regenCoroutine);
    }
    regenCoroutine = StartCoroutine(RegenHealth());
}

private IEnumerator RegenHealth()
{
    yield return new WaitForSeconds(regenDelay);

    while (currentHealth < maxHealth)
    {
        currentHealth += regenRate;
        yield return new WaitForSeconds(regenTickInterval);
    }
}
```

**Règles :**
- Toujours stocker la référence retournée par `StartCoroutine` pour pouvoir l'arrêter proprement.
- Ne pas utiliser `StopAllCoroutines()` sauf cas exceptionnel — vous arrêteriez toutes les coroutines du composant, y compris celles que vous n'avez pas intentionnellement lancées.
- Une coroutine s'arrête automatiquement si le GameObject ou le composant est désactivé. Évitez de vous appuyer sur ce comportement implicite — arrêtez-la explicitement dans `OnDisable`.

### Coroutines vs async/await

Unity supporte `async/await` depuis Unity 2017, mais avec des limitations. Par défaut, les tâches async Unity ne sont pas annulées quand le GameObject est détruit — ce qui peut provoquer des appels sur des objets détruits.

| | Coroutines | async/await |
|---|---|---|
| Lie au cycle de vie Unity | ✅ Automatique | ❌ Manuel (CancellationToken) |
| Accès à `yield` Unity (`WaitForSeconds`, etc.) | ✅ | ❌ |
| Gestion des exceptions | ❌ Silencieuses | ✅ try/catch |
| Lisibilité pour des flux complexes | ❌ Difficile | ✅ |

**Règle** : pour tout ce qui est lié au cycle de vie Unity et au timing (attendre N secondes, attendre la fin d'une animation...), utilisez les **coroutines**. Pour des opérations réseau ou I/O, utilisez **async/await** avec un `CancellationToken` lié à la destruction du composant.

---

## Git et Unity

### Les .meta sont sacrés

Les fichiers `.meta` **doivent toujours être versionnés**. Ils contiennent les GUIDs qui permettent à Unity de résoudre les références entre assets. Un `.meta` manquant ou modifié casse les références dans les scènes et les prefabs.

Ne les ajoutez **jamais** à `.gitignore`. Utilisez [ce .gitignore](https://gist.github.com/Marsgames/6b84af0d8dc2f6dd2080addc9213f87e) qui exclut ce qu'il faut sans toucher aux `.meta`.

Référence utile : [gitignore.io](https://www.toptal.com/developers/gitignore) pour générer un `.gitignore` adapté.

### Travail en équipe sur les scènes

Les fichiers de scène Unity (`.unity`) sont des fichiers binaires ou YAML difficiles à merger. Pour éviter les conflits :
- Une seule personne travaille sur une scène à la fois.
- Communiquez avant d'ouvrir une scène partagée.
- Configurez Unity pour le mode **Force Text** (Edit > Project Settings > Editor > Asset Serialization) pour que les scènes soient en YAML lisible — les conflits sont plus faciles à résoudre.

---

## Build et compilation

### AutoSave

Ajoutez un script [AutoSave](https://gist.github.com/Marsgames/40635c471ea962dd55a7afac5348c385) dans `Assets/Editor`. Il sauvegarde automatiquement la scène à chaque fois qu'on appuie sur Play.

**Pourquoi ?** Unity peut crasher sur une boucle infinie, un StackOverflow ou un bug dans le code natif. Sans AutoSave, toutes les modifications non sauvegardées depuis la dernière sauvegarde manuelle sont perdues.

### Warnings as errors (csc.rsp)

Créez un fichier `csc.rsp` dans le dossier `Assets` contenant :

```
-warnaserror+
```

Cela configure le compilateur C# d'Unity pour traiter les warnings comme des erreurs. Voir [general.md](general.md) pour la justification générale.

Si un warning ne peut pas être corrigé (API obsolète imposée, dépendance tierce...), supprimez-le localement :

```csharp
#pragma warning disable CS0612 // CS0612 : membre obsolète
    [SerializeField, Obsolete] private int legacyField = 0;
#pragma warning restore CS0612
```

Le numéro du code d'erreur s'affiche dans la console Unity avec le warning. N'oubliez pas le `restore`.

### Fichiers UnityEditor

Tout script contenant `using UnityEditor` **doit être placé dans un dossier `Assets/Editor`**, sauf s'il est entouré d'un `#if UNITY_EDITOR`.

```csharp
// ✅ — Placé dans Assets/Editor/, pas de guard nécessaire
using UnityEditor;

// ✅ — N'importe où, guard présent
#if UNITY_EDITOR
using UnityEditor;
// ...
#endif
```

**Pourquoi ?** Les assemblies dans `Assets/Editor` ne sont pas incluses dans le build de production. Un script `UnityEditor` hors de ce dossier et sans guard fait échouer le cloud build — `UnityEditor` n'existe pas dans le runtime de production.
