

# Chapitre 3 — Déduction de types, `auto`, `decltype` et templates modernes

> **Objectif :** comprendre comment le compilateur déduit les types afin d'écrire un code plus simple, plus sûr et plus générique.

------

# Pourquoi ce chapitre ?

Le C++ moderne repose énormément sur la **déduction de types**.

En C++98, il fallait pratiquement toujours écrire le type complet :

```cpp
std::vector<std::pair<std::string, std::unique_ptr<Motor>>>::iterator it =
    motors.begin();
```

Aujourd'hui :

```cpp
auto it = motors.begin();
```

Le compilateur connaît déjà le type.

L'idée fondamentale est la suivante :

> **Ne répétez pas une information que le compilateur connaît déjà.**

Cela rend le code :

- plus lisible ;
- plus robuste ;
- plus facile à maintenir ;
- moins sensible aux refactorings.

La déduction de types est utilisée partout en C++ moderne :

- avec `auto` ;
- avec les templates ;
- avec les lambdas ;
- avec les structured bindings ;
- avec les fonctions retournant `auto`.

Comprendre ce mécanisme est donc indispensable.

------

# 1. `auto`

## Déduction simple

Le mot-clé `auto` demande au compilateur de déduire le type d'une variable à partir de son initialisation.

```cpp
auto a = 42;
```

équivaut à :

```cpp
int a = 42;
```

------

```cpp
auto pi = 3.14159;
```

↓

```cpp
double pi = 3.14159;
```

------

```cpp
auto s = std::string("Hello");
```

↓

```cpp
std::string s("Hello");
```

------

Le type est déterminé **à la compilation**.

Il ne pourra plus changer.

```cpp
auto x = 42;

x = 100;      // OK

// x = "Hello";   // Erreur
```

Contrairement à certains langages de script, `auto` ne signifie donc pas « type dynamique ».

------

# 2. Pourquoi utiliser `auto` ?

Le principal intérêt de `auto` est de supprimer les types inutilement verbeux.

Prenons un exemple réaliste :

```cpp
std::unordered_map<
    std::string,
    std::vector<std::unique_ptr<Motor>>
> motors;
```

Sans `auto` :

```cpp
std::unordered_map<
    std::string,
    std::vector<std::unique_ptr<Motor>>
>::iterator it = motors.begin();
```

Avec `auto` :

```cpp
auto it = motors.begin();
```

Le code est plus court, mais surtout plus lisible.

------

Autre exemple très courant :

```cpp
auto motor = std::make_unique<Motor>();
```

au lieu de :

```cpp
std::unique_ptr<Motor> motor =
    std::make_unique<Motor>();
```

Ici encore, le compilateur connaît parfaitement le type.

Le répéter n'apporte aucune information supplémentaire.

------

# 3. `auto` ne signifie pas "type dynamique"

Les développeurs venant du C# assimilent souvent `auto` à `var`.

L'idée est proche, mais il est important de comprendre ce que fait réellement le compilateur.

Prenons :

```cpp
auto x = 42;
```

Pendant la compilation, le compilateur remplace simplement cette ligne par :

```cpp
int x = 42;
```

Une fois le programme compilé, il n'existe plus aucune trace du mot `auto`.

Le type réel est parfaitement connu.

Il n'y a donc :

- aucune perte de performances ;
- aucune réflexion (reflection) ;
- aucun typage dynamique.

------

# 4. `auto`, copies et références

C'est probablement la partie la plus importante du chapitre.

Le comportement de `auto` dépend de la façon dont il est déclaré.

## Cas n°1 : copie

```cpp
std::string name = "John";

auto s = name;
```

Le compilateur déduit :

```cpp
std::string
```

Une copie est créée.

```
name ------> "John"

s ---------> "John"
```

Les deux objets sont indépendants.

```cpp
s = "Paul";
```

↓

```
name == "John"

s == "Paul"
```

------

## Cas n°2 : référence

```cpp
std::string name = "John";

auto& s = name;
```

Cette fois :

```
        +------+
name -->|John  |
        +------+
            ^
            |
            s
```

Il n'existe qu'un seul objet.

Modifier `s` modifie également `name`.

```cpp
s = "Paul";
```

↓

```
name == "Paul"
```

------

## Cas n°3 : référence constante

Très fréquent :

```cpp
const auto& s = name;
```

Cette forme :

- évite toute copie ;
- garantit qu'aucune modification ne sera effectuée.

C'est la manière standard de parcourir des objets volumineux.

------

# 5. Les pièges de `auto`

Le compilateur applique plusieurs règles de déduction.

Les connaître permet d'éviter de nombreux bugs.

------

## Piège n°1 : les références disparaissent

Supposons :

```cpp
std::string name = "John";

std::string& ref = name;
```

Puis :

```cpp
auto x = ref;
```

Beaucoup de débutants pensent obtenir :

```cpp
std::string&
```

En réalité :

```cpp
std::string
```

Une copie est créée.

Si l'on souhaite conserver la référence :

```cpp
auto& x = ref;
```

------

## Piège n°2 : le `const` disparaît

```cpp
const int value = 42;

auto x = value;
```

Le type obtenu est :

```cpp
int
```

Pourquoi ?

Parce qu'une copie est créée.

Cette copie est indépendante de l'objet d'origine.

Pour conserver le caractère constant :

```cpp
const auto x = value;
```

ou

```cpp
const auto& x = value;
```

------

## Piège n°3 : copier toute une collection sans le vouloir

Exemple classique :

```cpp
std::vector<int> values = {1,2,3};

for (auto v : values)
{
    v *= 2;
}
```

Le résultat est surprenant :

```
1
2
3
```

Pourquoi ?

Parce que chaque élément est copié.

```
values

1
2
3

↓

copies

v
```

Pour modifier réellement le tableau :

```cpp
for (auto& v : values)
{
    v *= 2;
}
```

Résultat :

```
2
4
6
```

------

## Piège n°4 : copier de gros objets

Supposons :

```cpp
std::vector<Motor> motors;
```

Puis :

```cpp
for (auto motor : motors)
{
    ...
}
```

Chaque itération copie un objet `Motor`.

Si `Motor` est volumineux, cela peut coûter cher.

Préférez :

```cpp
for (const auto& motor : motors)
{
    ...
}
```

Aucune copie n'est effectuée.

------

## Règle pratique

Lorsque vous écrivez :

```cpp
auto
```

posez-vous toujours cette question :

> **Est-ce que je veux une copie ou une référence ?**

En pratique, cela conduit presque toujours à choisir parmi ces quatre formes :

| Déclaration   | Signification              |
| ------------- | -------------------------- |
| `auto`        | Copie                      |
| `const auto`  | Copie constante            |
| `auto&`       | Référence modifiable       |
| `const auto&` | Référence en lecture seule |

------

# 6. `decltype`

Il arrive que l'on souhaite récupérer **exactement** le type d'une expression.

C'est le rôle de `decltype`.

Exemple :

```cpp
int x = 5;

decltype(x) y = 10;
```

Le compilateur remplace simplement :

```cpp
decltype(x)
```

par :

```cpp
int
```

------

Autre exemple :

```cpp
std::vector<int> values;

decltype(values.begin()) it = values.begin();
```

Le type de `it` est exactement celui retourné par `begin()`.

Aujourd'hui, on écrirait plus simplement :

```cpp
auto it = values.begin();
```

mais `decltype` reste indispensable dans certains templates où le type n'est pas connu à l'avance.

------

À retenir :

- `auto` déduit le type **à partir d'une initialisation** ;
- `decltype` déduit le type **à partir d'une expression existante**.

Ces deux mécanismes sont complémentaires et constituent la base de la déduction de types en C++ moderne.



# 7. `decltype(auto)`

`decltype(auto)` est un cas particulier.

Il combine les avantages de `auto` et de `decltype`.

Avec `auto`, certaines informations sont perdues (comme les références ou les `const`).

Avec `decltype(auto)`, le compilateur conserve **exactement** le type de l'expression.

Exemple :

```cpp
int value = 42;

int& getValue()
{
    return value;
}
```

Si l'on écrit :

```cpp
auto x = getValue();
```

Le type déduit est :

```cpp
int
```

Une copie est effectuée.

En revanche :

```cpp
decltype(auto) x = getValue();
```

Le type devient :

```cpp
int&
```

La référence est conservée.

------

Dans la pratique, `decltype(auto)` est surtout utilisé dans les bibliothèques génériques et dans certaines fonctions qui souhaitent retourner exactement ce qu'elles reçoivent.

Ce n'est pas un outil utilisé quotidiennement dans du code applicatif.

------

# 8. Les fonctions avec `auto`

Depuis C++14, le compilateur peut également déduire le type de retour d'une fonction.

Exemple :

```cpp
auto square(int x)
{
    return x * x;
}
```

Le compilateur déduit automatiquement :

```cpp
int
```

------

Autre exemple :

```cpp
auto createMotor()
{
    return std::make_unique<Motor>();
}
```

Le type réel est :

```cpp
std::unique_ptr<Motor>
```

Il n'est plus nécessaire de l'écrire.

Cela simplifie énormément les fonctions retournant des types complexes.

------

# 9. Les lambdas

Nous avons brièvement rencontré les lambdas.

Une lambda est une **fonction anonyme**.

Exemple :

```cpp
auto square =
    [](int x)
    {
        return x * x;
    };
```

Puis :

```cpp
int y = square(5);
```

Le résultat vaut :

```text
25
```

Le type réel d'une lambda est généré par le compilateur.

Il n'a pas de nom.

C'est pourquoi on utilise presque toujours :

```cpp
auto
```

pour stocker une lambda.

Les captures (`[]`, `[=]`, `[&]`, etc.) feront l'objet d'un chapitre dédié.

Pour l'instant, retenez simplement qu'une lambda permet d'écrire une petite fonction directement à l'endroit où elle est utilisée.

------

# 10. Structured bindings (C++17)

Les structured bindings permettent de décomposer un objet composé.

Supposons une fonction :

```cpp
std::pair<std::string, int> getMotorInfo();
```

Avant C++17 :

```cpp
auto info = getMotorInfo();

std::cout << info.first;
std::cout << info.second;
```

Aujourd'hui :

```cpp
auto [name, id] = getMotorInfo();
```

Les deux variables sont créées automatiquement.

------

Autre exemple très fréquent :

```cpp
for (auto& [id, graph] : motionGraphs)
{
    ...
}
```

où `motionGraphs` est une `std::unordered_map`.

Les structured bindings améliorent énormément la lisibilité.

------

# 11. Déduction de types dans les templates

Nous avons déjà rencontré :

```cpp
template<typename T>
```

Comment le compilateur choisit-il `T` ?

------

## Cas n°1 : passage par valeur

```cpp
template<typename T>
void f(T value)
{
}
```

Appel :

```cpp
int x = 5;

f(x);
```

Le compilateur déduit :

```text
T = int
```

Le paramètre est copié.

------

## Cas n°2 : passage par référence

```cpp
template<typename T>
void f(T& value)
{
}
```

Le compilateur déduit :

```text
T = int
```

mais le paramètre est une référence.

Aucune copie n'est réalisée.

------

## Cas n°3 : référence constante

```cpp
template<typename T>
void f(const T& value)
{
}
```

Cette forme est extrêmement fréquente.

Elle permet :

- d'éviter les copies ;
- d'accepter des objets constants ;
- d'accepter des temporaires.

Une grande partie de la STL utilise cette forme.

------

## Cas n°4 : forwarding reference

```cpp
template<typename T>
void f(T&& value)
{
}
```

Nous retrouvons ici les notions de lvalue et de rvalue vues au chapitre précédent.

Selon l'argument fourni :

- une lvalue donnera une référence ;
- une rvalue donnera une référence rvalue.

C'est ce mécanisme qui permet le **perfect forwarding**.

Nous lui consacrerons un chapitre complet.

Pour le moment, retenez simplement que ce comportement est propre aux templates.

------

# Tableau récapitulatif

| Déclaration   | Copie | Modification | Usage principal    |
| ------------- | ----- | ------------ | ------------------ |
| `auto`        | ✔     | ✔            | Petits types       |
| `const auto`  | ✔     | ✖            | Constantes locales |
| `auto&`       | ✖     | ✔            | Modifier un objet  |
| `const auto&` | ✖     | ✖            | Lire un objet      |
| `auto&&`      | ✖     | Dépend       | Templates avancés  |

------

# Bonnes pratiques

✔ Utiliser `auto` lorsque le type est évident.

✔ Utiliser `auto` avec `make_unique()` et `make_shared()`.

✔ Utiliser `auto` pour les itérateurs.

✔ Utiliser `const auto&` pour parcourir des objets volumineux.

✔ Utiliser `auto&` lorsque l'on souhaite modifier une collection.

✔ Éviter `auto` lorsque le type apporte une information importante au lecteur.

Par exemple :

```cpp
double distance = computeDistance();
```

est parfois plus lisible que :

```cpp
auto distance = computeDistance();
```

Le type `double` fait ici partie de la documentation du code.

------

# Résumé

Le C++ moderne cherche à supprimer le code répétitif sans perdre les avantages du typage statique.

Le compilateur est capable de déduire de nombreux types automatiquement.

`auto` améliore la lisibilité, simplifie les refactorings et réduit les risques d'erreur.

Cependant, il est essentiel de comprendre la différence entre :

- une copie (`auto`) ;
- une référence (`auto&`) ;
- une référence constante (`const auto&`).

Ce choix influence directement les performances et la sémantique du programme.

------

# À retenir

- `auto` demande au compilateur de déduire le type.
- Le type reste statique.
- `auto` crée généralement une copie.
- `auto&` conserve une référence.
- `const auto&` est la forme standard pour lire des objets.
- `decltype` récupère le type exact d'une expression.
- `decltype(auto)` conserve exactement le type retourné par une expression.
- Les templates utilisent les mêmes mécanismes de déduction que `auto`.
- Les structured bindings simplifient énormément la manipulation des paires, tuples et maps.

------

# Erreurs fréquentes

❌ Penser que `auto` est un type dynamique.

❌ Oublier que `auto` supprime les références et les `const` lors d'une copie.

❌ Modifier une copie dans une boucle :

```cpp
for (auto x : values)
```

au lieu de :

```cpp
for (auto& x : values)
```

❌ Copier de gros objets alors qu'une simple référence constante suffit.

Une bonne compréhension de ces règles permet d'écrire un code plus lisible, plus performant et plus idiomatique.