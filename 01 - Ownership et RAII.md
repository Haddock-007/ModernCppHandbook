

# Modern C++ Handbook

# Chapitre 1 — Smart Pointers et RAII

> **Objectif**
>
> Comprendre le changement de philosophie le plus important apparu depuis C++98 :
> **la gestion automatique de la durée de vie des objets.**
>
> C'est probablement LE concept à maîtriser avant tout le reste du C++ moderne.

---

# Pourquoi ce changement ?

En C++98, la majorité des objets étaient créés avec `new` puis détruits avec `delete`.

```cpp
MyObject* obj = new MyObject();

// ...

delete obj;
```

Cette approche fonctionne…

…jusqu'au jour où l'on oublie un `delete`.

Ou pire :

- plusieurs `delete`
- une exception
- un retour prématuré
- une fuite mémoire
- un pointeur devenu invalide

Une grande partie des bugs C++ provenaient simplement de cette gestion manuelle.

Le C++ moderne adopte une philosophie différente :

> **La durée de vie d'une ressource doit être liée à la durée de vie d'un objet.**

Cette idée est appelée **RAII**.

---

# Le principe RAII

RAII signifie :

> **Resource Acquisition Is Initialization**

Autrement dit :

- le constructeur acquiert une ressource
- le destructeur la libère

Exemple :

```cpp
{
    std::fstream file("test.txt");

    // utilisation du fichier

} // fermeture automatique
```

Aucun appel à `close()` n'est nécessaire.

Le destructeur s'en charge automatiquement.

Cette philosophie est utilisée partout :

- fichiers
- mutex
- sockets
- mémoire
- handles Windows
- connexions réseau
- etc.

---

# Les Smart Pointers

Les smart pointers sont simplement des objets RAII spécialisés dans la gestion de mémoire.

Au lieu de ceci :

```cpp
MyObject* p = new MyObject();
```

on écrit :

```cpp
auto p = std::make_unique<MyObject>();
```

Lorsque `p` disparaît :

- le destructeur est appelé
- la mémoire est libérée automatiquement

Impossible d'oublier le `delete`.

---

# std::unique_ptr

Le smart pointer le plus utilisé.

Il possède **un propriétaire unique**.

```cpp
auto motor = std::make_unique<Motor>();
```

Lorsque `motor` disparaît :

```cpp
motor.reset();
```

ou

```cpp
}
```

l'objet est détruit.

---

## Transfert de propriété

Un `unique_ptr` ne peut pas être copié.

Ceci est interdit :

```cpp
auto a = std::make_unique<Motor>();

auto b = a;      // erreur
```

Il faut transférer la propriété :

```cpp
auto b = std::move(a);
```

Après :

```cpp
a == nullptr
```

et

```cpp
b
```

possède l'objet.

C'est extrêmement important.

Un objet ne possède qu'un seul propriétaire.

---

# std::shared_ptr

Parfois plusieurs objets doivent partager la même ressource.

Exemple :

```
Renderer
     \
      \
       Mesh
      /
Scene
```

Les deux utilisent le même objet.

On utilise alors :

```cpp
auto mesh = std::make_shared<Mesh>();
```

Chaque copie incrémente un compteur.

Lorsque le compteur atteint zéro :

l'objet est détruit.

---

# std::weak_ptr

Le problème du `shared_ptr` est le cycle de références.

Exemple :

```
Parent ----> Child
   ^            |
   |            |
   +------------+
```

Les deux se possèdent mutuellement.

Le compteur ne peut jamais atteindre zéro.

Résultat :

fuite mémoire.

Le `weak_ptr` casse ce cycle.

Il ne possède pas l'objet.

Il permet seulement de l'observer.

---

# Pourquoi make_unique ?

On pourrait écrire :

```cpp
std::unique_ptr<MyObject> p(new MyObject());
```

mais aujourd'hui on écrit :

```cpp
auto p = std::make_unique<MyObject>();
```

Pourquoi ?

Parce que :

- plus lisible
- plus sûr
- meilleure gestion des exceptions
- une seule allocation d'expression

C'est devenu la norme.

Même idée pour :

```cpp
std::make_shared()
```

---

# Que reste-t-il des pointeurs bruts ?

Les pointeurs classiques existent toujours.

Ils servent principalement à :

- observer un objet
- interagir avec du code C
- API système
- zones mémoire particulières

Mais ils ne doivent généralement plus représenter la propriété.

Exemple :

```cpp
void draw(const Mesh* mesh);
```

Le pointeur indique simplement :

> "je regarde cet objet"

pas

> "je suis responsable de sa destruction"

---

# Les références

Pour un paramètre obligatoire :

```cpp
void update(Motor& motor);
```

Pour un paramètre optionnel :

```cpp
void update(Motor* motor);
```

Le C++ moderne préfère largement les références lorsqu'un objet doit exister.

---

# Ce qu'il ne faut presque plus écrire

Éviter :

```cpp
new
delete
```

dans le code métier.

Ils existent encore…

mais deviennent très rares.

On les rencontre principalement :

- implémentation de bibliothèques
- allocateurs personnalisés
- moteurs temps réel
- systèmes embarqués
- wrappers bas niveau

---

# Application à notre Motion Engine

Notre architecture utilise déjà cette philosophie.

Par exemple :

```cpp
std::unique_ptr<MotionNode>

std::unique_ptr<IGroupRunner>

std::unique_ptr<IMaster>

std::unique_ptr<IValueSource>
```

Chaque composant possède clairement les objets qu'il crée.

Lorsqu'un `MotionProgram` est détruit, toute l'arborescence est automatiquement libérée.

Aucun `delete` n'est nécessaire.

C'est exactement l'esprit du C++ moderne.

---

# Erreurs classiques

### Copier un unique_ptr

```cpp
auto b = a;
```

Impossible.

Toujours utiliser :

```cpp
auto b = std::move(a);
```

---

### Utiliser un objet après move

```cpp
auto b = std::move(a);

a->run();          // erreur
```

Après un `move`, le pointeur source est considéré vide.

---

### Utiliser shared_ptr partout

C'est probablement l'erreur la plus fréquente.

Un `shared_ptr` coûte plus cher :

- compteur atomique
- allocations supplémentaires
- durée de vie moins claire

Toujours commencer par :

```
unique_ptr
```

Passer au `shared_ptr` uniquement lorsqu'un véritable partage de propriété est nécessaire.

---

# Bonnes pratiques

✔ Utiliser des objets automatiques lorsque c'est possible.

✔ Utiliser `unique_ptr` par défaut.

✔ Utiliser `shared_ptr` seulement si plusieurs propriétaires existent.

✔ Utiliser `weak_ptr` pour casser les cycles.

✔ Éviter `new` et `delete`.

✔ Utiliser `std::make_unique()`.

✔ Utiliser `std::make_shared()`.

---

# Résumé

Depuis C++11, la mémoire n'est plus gérée manuellement.

Le principe est simple :

- un objet possède une ressource ;
- son destructeur la libère automatiquement.

Cette approche (RAII) constitue la base de pratiquement toutes les bibliothèques modernes C++.

Les smart pointers sont simplement l'application de cette philosophie à la mémoire dynamique.

En pratique :

- **`std::unique_ptr`** est le choix par défaut ;
- **`std::shared_ptr`** ne doit être utilisé qu'en cas de propriété réellement partagée ;
- **`std::weak_ptr`** sert à observer un objet sans en prolonger la durée de vie.