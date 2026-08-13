# Headcrab — Guidelines de développement

Ce dossier contient l'ensemble des bonnes pratiques de développement chez Headcrab.

Les guidelines sont découpées par domaine pour qu'un développeur n'ait à lire que ce qui le concerne.

---

## Par où commencer ?

**Tout le monde lit [general.md](general.md).**

Ensuite, selon ton projet :

| Tu travailles sur… | Tu lis aussi… |
|---|---|
| Un projet C# | [csharp.md](csharp.md) |
| Un projet Unity | [csharp.md](csharp.md) + [unity.md](unity.md) |
| Un projet Swift / iOS | [swift.md](swift.md) |
| Une API | [api.md](api.md) |

---

## Contenu

### [general.md](general.md) — Pratiques communes à tous les langages
Langue · Nommage · Accolades · Early Return · Taille des fonctions et classes · Scope · Booléens · Constantes · Documentation · Commentaires · Imports · Programmation défensive · Strings · Types abstraits · Flags de compilation · **Async/Await** · **Git**

### [csharp.md](csharp.md) — Spécificités C#
Convention de nommage · `var` · Getters/Setters · Propriétés · `sealed`

### [unity.md](unity.md) — Spécificités Unity
`==` vs `is` · SerializeField · Inspecteur · Templates · AutoSave · `??` et `?.` avec MonoBehaviour · Cloud Build

### [swift.md](swift.md) — Spécificités Swift
Convention de nommage · Optionals · Error handling · Protocoles

### [api.md](api.md) — Design d'API
*(à venir)*

---

## Philosophie

Ces guidelines existent pour que n'importe quel développeur de l'équipe puisse lire du code qu'il n'a pas écrit et le comprendre sans effort.

Chaque règle est accompagnée de son **pourquoi**. Une règle sans explication est une règle qu'on ne respecte pas dès qu'on a le dos tourné.

Ces guidelines ne sont pas gravées dans le marbre. Si tu penses qu'une règle est mauvaise, discutes-en — le seul mauvais scénario est de l'ignorer en silence.
