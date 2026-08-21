# Guidelines API REST

Ce document couvre les bonnes pratiques de conception et de consommation d'API REST chez Headcrab.

---

## Sommaire

- [Principes fondamentaux](#principes-fondamentaux)
- [Nommage des endpoints](#nommage-des-endpoints)
- [Verbes HTTP](#verbes-http)
- [Codes de statut](#codes-de-statut)
- [Format des réponses](#format-des-réponses)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Versioning](#versioning)
- [Authentification](#authentification)
- [CORS](#cors)
- [Pagination](#pagination)
- [Filtrage et tri](#filtrage-et-tri)
- [Côté client](#côté-client)

---

## Principes fondamentaux

Une API REST manipule des **ressources**, pas des actions. L'URL identifie _ce sur quoi_ on agit ; le verbe HTTP identifie _ce qu'on fait_.

```
// ❌ — Orienté action (RPC-style)
POST /getUser
POST /createUser
POST /deleteUser
POST /updateUserEmail

// ✅ — Orienté ressource (REST)
GET    /users
GET    /users/{id}
POST   /users
DELETE /users/{id}
PATCH  /users/{id}
```

Une API bien conçue est **prévisible** : quelqu'un qui connaît la convention peut deviner les endpoints sans lire la documentation.

---

## Nommage des endpoints

### Règles de base

- **Noms au pluriel** : `/users`, `/orders`, `/products` — une collection est toujours plurielle.
- **kebab-case** : `/user-profiles`, `/payment-methods` — pas de camelCase ni snake_case dans les URLs.
- **Minuscules uniquement**.
- **Pas de verbes** dans les URLs — c'est le rôle du verbe HTTP.
- **Pas d'extension de fichier** : `/users`, pas `/users.json`.

### Hiérarchie des ressources

Exprimez les relations avec des sous-ressources, mais limitez la profondeur à deux niveaux maximum.

```
// ✅ — Relation claire, profondeur raisonnable
GET /users/{userId}/orders
GET /users/{userId}/orders/{orderId}

// ❌ — Trop profond, difficile à maintenir
GET /users/{userId}/orders/{orderId}/items/{itemId}/reviews/{reviewId}
```

Au-delà de deux niveaux, aplatissez et filtrez par paramètre :

```
// ✅ — Plus simple, aussi expressif
GET /reviews?orderId={orderId}
```

### Actions non-CRUD

Parfois une opération ne rentre pas dans les verbes HTTP standard (envoi d'un email, validation, publication...). Utilisez un sous-chemin en forme de verbe d'action, en le documentant explicitement.

```
POST /users/{id}/verify-email
POST /orders/{id}/cancel
POST /reports/{id}/publish
```

---

## Verbes HTTP

| Verbe    | Usage                                     | Idempotent | Body de requête |
| -------- | ----------------------------------------- | ---------- | --------------- |
| `GET`    | Lire une ou plusieurs ressources          | ✅         | ❌              |
| `POST`   | Créer une ressource                       | ❌         | ✅              |
| `PUT`    | Remplacer complètement une ressource      | ✅         | ✅              |
| `PATCH`  | Mettre à jour partiellement une ressource | ✅         | ✅              |
| `DELETE` | Supprimer une ressource                   | ✅         | ❌              |

> **Idempotent** : appeler la même requête plusieurs fois produit le même résultat. Un `GET` ou un `DELETE` répété ne change rien après le premier appel. Un `POST` répété crée plusieurs ressources.

### PUT vs PATCH

- `PUT` remplace **entièrement** la ressource. Tous les champs doivent être envoyés — les champs absents sont réinitialisés.
- `PATCH` met à jour **uniquement les champs envoyés**. Les champs absents restent inchangés.

```json
// PATCH /users/42 — met à jour uniquement l'email
{ "email": "new@email.com" }

// PUT /users/42 — remplace tout l'objet, l'email est supprimé
{
  "name": "Alice",
  "role": "admin"
}
```

Préférez `PATCH` pour les mises à jour partielles — c'est le cas le plus courant et il évite d'envoyer inutilement tout l'objet.

---

## Codes de statut

N'utilisez **pas uniquement** `200` et `500`. Les codes HTTP communicent l'intention — un client bien conçu peut adapter son comportement sans lire le corps de la réponse.

### 2xx — Succès

| Code                | Quand l'utiliser                                                                         |
| ------------------- | ---------------------------------------------------------------------------------------- |
| `200 OK`            | Requête réussie avec corps de réponse (GET, PATCH, PUT)                                  |
| `201 Created`       | Ressource créée (POST). Inclure un header `Location` pointant vers la nouvelle ressource |
| `202 Accepted`      | Traitement async d'une requête en cours                                                  |
| `204 No Content`    | Requête réussie sans corps de réponse (DELETE, certains PUT)                             |
| `207 Multi-Status`  | Opération bulk où chaque item a son propre statut (création ou modification par lot)     |

Le `207` retourne un tableau de résultats individuels, chacun avec son propre code de statut :

```json
// POST /users/bulk
// HTTP 207 Multi-Status
{
  "results": [
    { "id": "1", "status": 201, "data": { "id": "1", "name": "Alice" } },
    { "id": "2", "status": 409, "error": { "code": "EMAIL_ALREADY_EXISTS", "message": "..." } },
    { "id": "3", "status": 201, "data": { "id": "3", "name": "Charlie" } }
  ]
}
```

Ne retournez pas `200` pour une opération bulk partielle — le client ne saurait pas qu'une partie a échoué sans lire tout le corps.

### 3xx — Redirections

| Code                       | Quand l'utiliser                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------- |
| `301 Moved Permanently`    | Endpoint déplacé définitivement. Préférez `308` si le verbe doit être préservé         |
| `308 Permanent Redirect`   | Endpoint déplacé définitivement, avec préservation du verbe HTTP (POST reste POST)     |

Le `301` a un comportement hérité des navigateurs : beaucoup de clients HTTP convertissent automatiquement `POST` en `GET` sur la redirection. Le `308` corrige ce problème. Pour les endpoints dépréciés qui acceptent du `POST`, `PATCH` ou `PUT`, utilisez toujours `308`.

Accompagnez toujours une redirection du header `Location` pointant vers le nouvel endpoint, et documentez la date de suppression prévue de l'ancien.

### 4xx — Erreur client

| Code                       | Quand l'utiliser                                                                           |
| -------------------------- | ------------------------------------------------------------------------------------------ |
| `400 Bad Request`          | Requête malformée, paramètres manquants ou invalides                                       |
| `401 Unauthorized`         | Non authentifié — token absent ou invalide                                                 |
| `403 Forbidden`            | Authentifié mais sans permission sur cette ressource                                       |
| `404 Not Found`            | Ressource introuvable — son existence passée est inconnue                                  |
| `409 Conflict`             | Conflit d'état (email déjà utilisé, ressource déjà dans cet état...)                       |
| `410 Gone`                 | Ressource définitivement supprimée — le serveur sait qu'elle a existé et ne reviendra pas  |
| `422 Unprocessable Entity` | Requête syntaxiquement valide mais sémantiquement incorrecte (validation métier)           |
| `429 Too Many Requests`    | Rate limiting — inclure un header `Retry-After`                                            |

La différence entre `404` et `410` : un `404` peut être temporaire (mauvaise URL, ressource pas encore créée...) ; un `410` est définitif. Un client ou un crawler qui reçoit `410` sait qu'il peut arrêter de retenter. Utilisez `410` pour les endpoints supprimés après dépréciation.

### 5xx — Erreur serveur

| Code                        | Quand l'utiliser                                             |
| --------------------------- | ------------------------------------------------------------ |
| `500 Internal Server Error` | Erreur serveur inattendue                                    |
| `503 Service Unavailable`   | Service temporairement indisponible (maintenance, surcharge) |

**Ne jamais retourner `200` avec une erreur dans le corps.** C'est une des erreurs les plus répandues et elle force les clients à parser le corps pour détecter les erreurs.

```json
// ❌ — Trompe le client et les outils de monitoring
HTTP 200 OK
{ "success": false, "error": "User not found" }

// ✅
HTTP 404 Not Found
{ "error": { "code": "USER_NOT_FOUND", "message": "No user found with id 42." } }
```

---

## Format des réponses

### Enveloppe standard

Toutes les réponses utilisent une enveloppe JSON cohérente.

**Ressource unique :**

```json
{
  "data": {
    "id": "42",
    "name": "Alice",
    "email": "alice@example.com",
    "createdAt": "2024-03-15T10:30:00Z"
  }
}
```

**Collection :**

```json
{
  "data": [
    { "id": "1", "name": "Alice" },
    { "id": "2", "name": "Bob" }
  ],
  "pagination": {
    "cursor": "eyJpZCI6Mn0=",
    "hasMore": true
  }
}
```

**Pourquoi une enveloppe ?** Elle permet d'ajouter des métadonnées (pagination, liens, version...) sans casser les clients existants. Une réponse qui retourne directement un tableau rend toute évolution impossible sans breaking change.

### Conventions de nommage

- **camelCase** pour les clés JSON : `createdAt`, `userId`, `firstName`.
- **ISO 8601** pour les dates : `"2024-03-15T10:30:00Z"`. Toujours en UTC.
- **Chaînes pour les IDs** : même si l'ID est un entier côté serveur, retournez-le en string — les entiers 64-bit dépassent la précision de certains parseurs JSON JavaScript.
- **Pas de `null` inutile** : omettez les champs absents plutôt que de les retourner à `null`, sauf si `null` a une signification sémantique propre (champ optionnel explicitement vidé).

```json
// ❌
{ "id": 42, "nickname": null, "deletedAt": null }

// ✅
{ "id": "42" }
```

---

## Gestion des erreurs

Toutes les erreurs suivent le même format, quel que soit le code de statut.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request contains invalid fields.",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address."
      },
      {
        "field": "age",
        "message": "Must be at least 18."
      }
    ]
  }
}
```

- **`code`** : identifiant machine-readable en UPPER_SNAKE_CASE. Stable entre les versions — les clients peuvent brancher de la logique dessus.
- **`message`** : message lisible par un humain. Peut changer entre les versions.
- **`details`** : liste optionnelle pour les erreurs de validation multi-champs.

Les codes d'erreur sont **documentés et exhaustifs** — un client ne doit jamais recevoir un code qu'il ne connaît pas et ne peut pas gérer.

---

## Versioning

Versionnez par l'URL, préfixé sur toutes les routes.

```
https://api.headcrab.io/v1/users
https://api.headcrab.io/v2/users
```

**Règles :**

- Une nouvelle version uniquement pour des **breaking changes** (suppression de champ, changement de type, changement de comportement incompatible).
- Les ajouts de champs ou de nouveaux endpoints sont **non-breaking** — pas besoin de nouvelle version.
- Maintenez l'ancienne version au minimum **6 mois** après la mise en production de la nouvelle, avec une date de dépréciation communiquée.
- Documentez les migrations entre versions.

**Pourquoi pas le header ?** `Accept: application/vnd.headcrab.v2+json` est plus "pur" REST mais moins visible, moins testable dans un navigateur, et moins bien supporté par les outils de monitoring et de cache. L'URL est pragmatique et universelle.

---

## Authentification

Utilisez **Bearer token** via le header `Authorization`.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Règles :**

- Le token n'est **jamais** dans l'URL (il apparaîtrait dans les logs serveur et l'historique navigateur).
- Les endpoints publics sont explicitement documentés comme tels — le défaut est privé.
- `401` = token absent ou invalide (pas d'informations sur pourquoi, pour ne pas aider un attaquant).
- `403` = token valide mais permissions insuffisantes.
- Les tokens ont une durée de vie courte (15 min à 1h). Utilisez un refresh token pour renouveler sans ré-authentification.

---

## CORS

Le CORS (Cross-Origin Resource Sharing) est un mécanisme de sécurité du navigateur qui bloque les requêtes HTTP vers un domaine différent de celui de la page appelante. Il ne concerne pas les clients mobiles natifs (iOS, Android) — seulement les appels depuis un navigateur web.

**Pourquoi ça existe ?** Sans CORS, n'importe quelle page web malveillante pourrait faire des requêtes vers votre API en se faisant passer pour l'utilisateur connecté (attaque CSRF). Le navigateur bloque par défaut ; CORS permet au serveur de dire explicitement quelles origines il autorise.

### Headers à configurer côté serveur

```
Access-Control-Allow-Origin: https://app.headcrab.io
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

- **`Allow-Origin`** : liste explicite des origines autorisées. Ne jamais mettre `*` sur un endpoint authentifié — `*` est incompatible avec les cookies et le header `Authorization` en mode credentials.
- **`Allow-Methods`** : liste des verbes HTTP autorisés depuis une origine externe.
- **`Allow-Headers`** : headers custom que le client a le droit d'envoyer. Inclure `Authorization` si vous utilisez des Bearer tokens depuis le web.
- **`Max-Age`** : durée en secondes pendant laquelle le navigateur peut mettre en cache la réponse preflight. Évite de refaire une requête OPTIONS à chaque appel.

### Preflight

Pour toute requête non-simple (verbes autres que `GET`/`POST`, headers custom comme `Authorization`, body JSON...), le navigateur envoie automatiquement une requête `OPTIONS` préalable pour vérifier les permissions. Le serveur doit répondre `200` ou `204` à cette requête.

```
OPTIONS /users/42
Origin: https://app.headcrab.io
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: Authorization, Content-Type

→ HTTP 204 No Content
Access-Control-Allow-Origin: https://app.headcrab.io
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

Configurez votre serveur pour répondre à `OPTIONS` sur toutes les routes — ne le traitez pas comme un verbe métier.

### Plusieurs origines autorisées

Si l'API est consommée depuis plusieurs domaines (app web, dashboard admin, staging...), gérez la liste côté serveur : lisez l'header `Origin` de la requête et retournez-le dans `Allow-Origin` s'il est dans votre liste blanche. Ne concaténez pas plusieurs origines dans le header — il n'accepte qu'une seule valeur.

```python
ALLOWED_ORIGINS = {
    "https://app.headcrab.io",
    "https://admin.headcrab.io",
    "https://staging.headcrab.io",
}

def cors_origin(request_origin: str) -> str | None:
    return request_origin if request_origin in ALLOWED_ORIGINS else None
```

---

## Pagination

Pour toute collection pouvant dépasser quelques centaines d'éléments, la pagination est **obligatoire**. Une API qui retourne des listes non paginées est une bombe à retardement.

### Comparaison des techniques

|                                     | Offset                                             | Curseur / Keyset     | Temporel                    |
| ----------------------------------- | -------------------------------------------------- | -------------------- | --------------------------- |
| Stable sous insertions/suppressions | ❌                                                 | ✅                   | ✅                          |
| Saut direct à une page arbitraire   | ✅                                                 | ❌                   | ❌                          |
| Navigation arrière simple           | ✅                                                 | ⚠️                   | ⚠️                          |
| Nombre total de résultats           | ✅                                                 | ❌ (coûteux)         | ❌ (coûteux)                |
| Performant sur de grands datasets   | ❌                                                 | ✅                   | ✅                          |
| Risque de doublons                  | ❌                                                 | ❌                   | ⚠️                          |
| Idéal pour                          | Interfaces paginées, back-office, datasets stables | Scroll infini, feeds | Logs, événements, timelines |

### Offset

Le serveur saute N éléments et retourne les M suivants (`OFFSET N LIMIT M`). Simple à implémenter et à comprendre, il donne accès direct à n'importe quelle page et permet de retourner un total.

```
GET /users?page=3&limit=20
```

```json
{
  "data": [...],
  "pagination": { "page": 3, "limit": 20, "total": 347, "hasMore": true }
}
```

**Limitations :** si des éléments sont insérés ou supprimés entre deux requêtes, les pages se décalent — un élément peut apparaître en double ou être sauté. Sur de grands datasets, les performances se dégradent : `OFFSET 50000 LIMIT 20` oblige la base à parcourir 50 000 lignes pour les ignorer, même si elles ne sont pas retournées.

### Curseur / Keyset

Au lieu de sauter N éléments, le serveur reprend là où il s'est arrêté : après le dernier élément retourné. Le curseur encode la position de ce dernier élément (son `id`, ou un composite `id + createdAt` si le tri est sur une colonne non-unique) et est retourné au client, qui le renvoie à la requête suivante.

```
GET /users?cursor=eyJpZCI6MTAwfQ==&limit=20
```

```json
{
  "data": [...],
  "pagination": { "cursor": "eyJpZCI6MTIwfQ==", "hasMore": true }
}
```

Le curseur est **opaque** pour le client — il ne doit pas en interpréter le contenu, seulement le renvoyer tel quel. Cela permet au serveur de changer son implémentation interne sans breaking change.

**Limitations :** pas de saut direct à une page arbitraire — la navigation est séquentielle. La navigation arrière est possible mais nécessite que le client conserve une pile des curseurs précédents. Le total n'est pas retourné : un `COUNT(*)` sur une grande table est lent et se dégrade avec le temps ; on retourne `hasMore` à la place.

### Temporel

Variante du keyset où la clé de position est un timestamp. Naturel pour tout ce qui est ordonné chronologiquement : logs, événements, timelines.

```
GET /events?since=2024-03-15T10:30:00Z&limit=50
GET /messages?before=2024-03-15T10:30:00Z&limit=20
```

**Limitations :** si deux enregistrements ont exactement le même timestamp (fréquent sur des événements générés en rafale), certains peuvent être ignorés ou retournés en double. Pour l'éviter, combinez le timestamp avec un `id` secondaire comme tiebreaker — ce qui revient à un keyset composite.

---

**Taille maximale :** quelle que soit la technique, fixez une limite supérieure côté serveur (ex : 100). Un client qui demande `limit=10000` ne doit pas l'obtenir.

---

## Filtrage et tri

### Filtrage

Les filtres passent en query parameters.

```
GET /orders?status=pending&userId=42
GET /products?minPrice=10&maxPrice=100&category=electronics
```

Pour les filtres complexes ou multi-valeurs :

```
GET /products?tags=swift,ios,mobile
GET /orders?createdAfter=2024-01-01T00:00:00Z&createdBefore=2024-03-01T00:00:00Z
```

### Tri

```
GET /users?sort=createdAt&order=desc
GET /products?sort=price&order=asc
```

- `sort` : le champ sur lequel trier (nom de champ de la ressource).
- `order` : `asc` ou `desc`.
- Un tri par défaut doit toujours exister côté serveur — une liste non triée est non-déterministe.

---

## Côté client

### Toujours gérer les codes d'erreur explicitement

Ne traitez pas toutes les erreurs de la même façon. Un `401` appelle une ré-authentification, un `403` un message d'accès refusé, un `429` une attente avant retry.

```swift
// ❌ — Traitement uniforme, perte d'information
func handleResponse(_ response: HTTPURLResponse) {
    if response.statusCode != 200 {
        showGenericError()
    }
}

// ✅ — Traitement différencié
func handleResponse(_ response: HTTPURLResponse) throws {
    switch response.statusCode {
    case 200...299: return
    case 401: throw APIError.unauthorized
    case 403: throw APIError.forbidden
    case 404: throw APIError.notFound
    case 429: throw APIError.rateLimited
    case 500...599: throw APIError.serverError
    default: throw APIError.unexpected(response.statusCode)
    }
}
```

### Timeouts

Définissez toujours un timeout. Une requête sans timeout peut bloquer indéfiniment.

```swift
// ✅
var request = URLRequest(url: url)
request.timeoutInterval = 30  // 30 secondes maximum
```

Valeurs recommandées : 10-15s pour les requêtes courantes, 30-60s pour les uploads ou opérations longues.

### Retry et idempotence

Ne relancez automatiquement que les requêtes **idempotentes** (`GET`, `PUT`, `PATCH`, `DELETE`). Ne relancez jamais un `POST` automatiquement — vous risquez de créer des doublons.

```swift
// ✅ — Retry acceptable sur GET
func fetchWithRetry(url: URL, maxAttempts: Int = 3) async throws -> Data {
    for attempt in 1...maxAttempts {
        do {
            return try await fetch(url: url)
        } catch APIError.serverError where attempt < maxAttempts {
            try await Task.sleep(nanoseconds: UInt64(attempt) * 1_000_000_000)
        }
    }
    throw APIError.maxRetriesExceeded
}
```

### Ne jamais logguer les tokens ou données sensibles

Les headers `Authorization`, les tokens, et les données personnelles ne doivent jamais apparaître dans les logs, même en debug.

```swift
// ❌
print("Request headers: \(request.allHTTPHeaderFields)")  // Logue le token

// ✅
print("Request: \(request.httpMethod ?? "") \(request.url?.path ?? "")")
```
