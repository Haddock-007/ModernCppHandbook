

# Modern C++ Handbook

# Chapitre 2 — Références, Move Semantics et `std::move`

---

# Pourquoi ce chapitre ?

L'une des plus grandes évolutions du C++ moderne est l'apparition des **move semantics** (C++11).

Avant C++11, lorsqu'un objet devait être transféré, il était généralement **copié**.

Aujourd'hui, le compilateur cherche au contraire à **déplacer** les ressources lorsqu'il sait que l'objet d'origine ne sera plus utilisé.

Cette évolution est à la base de nombreuses fonctionnalités modernes :

- `std::unique_ptr`
- les conteneurs STL performants
- les retours de fonctions efficaces
- l'absence de copies inutiles

---

# Les trois façons de passer un objet

```cpp
class Motor
{
public:
    std::string name;
};
```

## Par valeur

```cpp
void f(Motor m)
{
}
```

Une copie est créée.

```
Motor A
   │
copie
   │
Motor B
```

---

## Par référence

```cpp
void f(Motor& m)
{
}
```

Aucune copie.

La fonction manipule directement l'objet original.

---

## Par référence constante

```cpp
void f(const Motor& m)
{
}
```

Toujours aucune copie.

La fonction promet simplement de ne pas modifier l'objet.

C'est aujourd'hui le mode de passage recommandé pour les gros objets en lecture.

---

# Pourquoi éviter les copies ?

Considérons :

```cpp
class Image
{
public:
    std::vector<uint8_t> pixels;
};
```

Passer cette classe par valeur implique de recopier tout son contenu.

```cpp
void display(Image img)
```

À chaque appel :

```
50 MB
   │
copie
   │
50 MB
```

Avec :

```cpp
void display(const Image& img)
```

aucune copie n'est réalisée.

---

# Copier ou déplacer ?

Une copie signifie :

```
Objet A

AAAAAA
```

↓

```
Objet B

AAAAAA
```

Chaque objet possède ses propres données.

---

Un déplacement signifie :

```
Objet A
   │
   └──── mémoire
```

↓

```
Objet B
   │
   └──── mémoire

Objet A devient vide
```

On ne recopie pas les données.

On transfère simplement leur propriété.

Cette opération est généralement très peu coûteuse.

---

# Une intuition sur les lvalues et les rvalues

Il n'est pas nécessaire de connaître toute la théorie.

Une règle simple suffit.

## Une lvalue

Une variable nommée est une **lvalue**.

```cpp
Motor m;

foo(m);
```

Après l'appel, on peut encore utiliser `m`.

Le compilateur ne peut donc pas déplacer automatiquement son contenu.

---

## Une rvalue

Un objet temporaire est une **rvalue**.

```cpp
foo(Motor());
```

ou

```cpp
foo(createMotor());
```

Ces objets n'ont pas de nom.

Ils vont disparaître juste après l'expression.

Le compilateur est donc libre d'en déplacer les ressources.

---

Une règle très utile est :

| Expression      | Catégorie       |
| --------------- | --------------- |
| `m`             | lvalue          |
| `Motor()`       | rvalue          |
| `createMotor()` | rvalue          |
| `std::move(m)`  | rvalue (xvalue) |

Retenez surtout qu'une **variable nommée est toujours une lvalue**, même si elle a été créée par déplacement.

---

# Le rôle de `std::move`

Considérons :

```cpp
Motor m;
```

Le compilateur considère que `m` pourra encore être utilisé.

Si l'on souhaite transférer définitivement son contenu :

```cpp
std::move(m)
```

Attention :

`std::move()` **ne déplace rien**.

Il indique simplement au compilateur :

> "Tu peux considérer cet objet comme temporaire."

Le déplacement sera ensuite réalisé par le constructeur ou l'opérateur de déplacement.

---

# Constructeur de copie

Considérons :

```cpp
Motor a;

Motor b = a;
```

Le compilateur appelle :

```cpp
Motor(const Motor&);
```

Schéma :

```
a

AAAAAA
   │
copie
   ▼
b

AAAAAA
```

Les deux objets sont indépendants.

---

# Constructeur de déplacement

Cette fois :

```cpp
Motor a;

Motor b = std::move(a);
```

Le compilateur appelle :

```cpp
Motor(Motor&&);
```

Schéma :

```
a
   │
   └──── mémoire
          │
          ▼
b
```

La mémoire est transférée.

`a` reste valide mais son contenu n'est plus spécifié.

---

# Affectation de copie

```cpp
Motor a;
Motor b;

b = a;
```

Appelle :

```cpp
Motor& operator=(const Motor&);
```

---

# Affectation par déplacement

```cpp
Motor a;
Motor b;

b = std::move(a);
```

Appelle :

```cpp
Motor& operator=(Motor&&);
```

L'ancien contenu de `b` est libéré, puis les ressources de `a` sont transférées.

---

# Ce qui est réellement appelé

| Code                      | Fonction appelée            |
| ------------------------- | --------------------------- |
| `Motor b = a;`            | constructeur de copie       |
| `Motor b = std::move(a);` | constructeur de déplacement |
| `b = a;`                  | affectation de copie        |
| `b = std::move(a);`       | affectation par déplacement |

Cette distinction est fondamentale.

La première ligne construit un nouvel objet.

Les deux dernières modifient un objet existant.

---

# Pourquoi `unique_ptr` exige un déplacement

```cpp
auto a = std::make_unique<Motor>();
```

Puis :

```cpp
auto b = a;
```

Impossible.

Deux propriétaires seraient créés.

En revanche :

```cpp
auto b = std::move(a);
```

fonctionne.

```
a ---> nullptr

b ---> Motor
```

Il n'y a toujours qu'un seul propriétaire.

---

# Retour de fonction

```cpp
Image load()
{
    Image img;

    ...

    return img;
}
```

Aujourd'hui, il ne faut généralement **pas** écrire :

```cpp
return std::move(img);
```

Il suffit d'écrire :

```cpp
return img;
```

Le compilateur appliquera automatiquement le RVO ou un déplacement si nécessaire.

---

# Erreurs classiques

## Utiliser un objet déplacé

```cpp
std::string a = "abc";

std::string b = std::move(a);

std::cout << a;
```

Le programme est valide.

Mais le contenu de `a` n'est plus garanti.

---

## Déplacer trop tôt

```cpp
foo(std::move(data));

bar(data);
```

Très probablement une erreur.

Après un `move`, l'objet doit être considéré comme vide.

---

# Bonnes pratiques

✔ Passer les gros objets en `const&` lorsqu'on les lit.

✔ Utiliser `&` lorsqu'on souhaite les modifier.

✔ Utiliser `std::move` uniquement lorsqu'on abandonne réellement l'objet.

✔ Considérer un objet déplacé comme valide mais vide.

✔ Laisser le compilateur optimiser les retours de fonctions.

---

# À retenir

- Une copie duplique les ressources.
- Un déplacement transfère leur propriété.
- Une variable nommée est toujours une lvalue.
- Un objet temporaire est une rvalue.
- `std::move()` ne déplace rien : il autorise simplement un déplacement.
- Le constructeur de copie et le constructeur de déplacement sont deux mécanismes distincts.
- Les opérateurs d'affectation suivent exactement la même logique.
- Les move semantics sont l'une des clés des performances du C++ moderne.