Voilà une roadmap visuelle d'abord, puis on détaille tout point par point.
---

## Niveau 1 — Socle du langage

### Variables et immutabilité

En Python, tu peux faire :
```python
x = 1
x = x + 1  # x vaut maintenant 2
```

En Elixir, les variables sont **immutables** — tu ne modifies pas, tu **rebindes** :
```elixir
x = 1
x = x + 1  # légal en Elixir, mais c'est un nouveau binding, pas une mutation
```

La différence est importante en concurrence : deux processus ne peuvent jamais se marcher dessus sur une variable, puisqu'aucune variable n'est partagée.

---

### Pattern matching — c'est le concept central

En Python, `=` est une assignation. En Elixir, `=` est un **match**. C'est fondamentalement différent.

```elixir
# Ça marche
{a, b} = {1, 2}       # a = 1, b = 2

# Ça plante si ça ne matche pas
{:ok, result} = {:ok, 42}    # result = 42
{:ok, result} = {:error, "oops"}  # crash ! :error ≠ :ok
```

Le pattern matching est partout : dans les `case`, les fonctions, les `with`. Si tu ne comprends pas ça, tu ne comprendras rien à Elixir.

---

### Fonctions et modules

```elixir
defmodule Calculatrice do
  def additionne(a, b), do: a + b

  def divise(_a, 0), do: {:error, "division par zéro"}
  def divise(a, b),  do: {:ok, a / b}
end

Calculatrice.additionne(2, 3)   # 5
Calculatrice.divise(10, 2)      # {:ok, 5.0}
Calculatrice.divise(10, 0)      # {:error, "division par zéro"}
```

La fonction `divise` a deux **clauses**. Elixir les teste dans l'ordre et prend la première qui matche. C'est du pattern matching appliqué aux arguments. Il n'y a pas de `if/else` ici — les clauses font le travail.

---

### Pipe operator `|>`

```elixir
# Sans pipe — tu lis de l'intérieur vers l'extérieur
String.upcase(String.trim("  hello  "))

# Avec pipe — tu lis dans l'ordre d'exécution
"  hello  "
|> String.trim()
|> String.upcase()
```

---

## Niveau 2 — Structures de données

| Structure | Exemple | Quand l'utiliser |
|---|---|---|
| List | `[1, 2, 3]` | Séquence ordonnée |
| Tuple | `{:ok, 42}` | Résultat de fonction, données liées |
| Map | `%{nom: "Alice", age: 30}` | Clé-valeur, comme `dict` Python |
| Keyword list | `[color: :red, size: 10]` | Options de fonction |
| Struct | `%User{nom: "Alice"}` | Modèle de données typé |

Le tuple `{:ok, valeur}` / `{:error, raison}` est une **convention** omniprésente. En Python tu aurais levé une exception — en Elixir tu retournes un tuple et tu traites les deux cas explicitement.

---

## Niveau 3 — Fonctionnel avancé

### Récursion à la place des boucles

Il n'y a pas de `for i in range(n)` en Elixir. Tu utilises la récursion ou `Enum`.

```elixir
# Récursion manuelle
def somme([]), do: 0
def somme([tete | reste]), do: tete + somme(reste)

# Avec Enum (à préférer en pratique)
Enum.sum([1, 2, 3, 4])  # 10
Enum.map([1, 2, 3], fn x -> x * 2 end)  # [2, 4, 6]
Enum.filter([1, 2, 3, 4], fn x -> rem(x, 2) == 0 end)  # [2, 4]
```

### `with` — gestion d'erreurs propre

```elixir
with {:ok, user}  <- get_user(id),
     {:ok, order} <- get_order(user),
     {:ok, paid}  <- charge(order) do
  {:ok, paid}
else
  {:error, reason} -> {:error, reason}
end
```

En Python tu aurais un try/except ou des `if` imbriqués. `with` chaîne les opérations qui peuvent échouer et court-circuite dès le premier `{:error, _}`.

---

## Niveau 4 — Concurrence et BEAM

C'est ce qui rend Elixir unique. Chaque tâche tourne dans un **processus léger** (pas un thread OS — des milliers peuvent tourner simultanément).

```elixir
# Lancer un processus
spawn(fn -> IO.puts("je tourne en parallèle") end)

# Envoyer et recevoir des messages
pid = spawn(fn ->
  receive do
    {:bonjour, nom} -> IO.puts("Salut #{nom}")
  end
end)

send(pid, {:bonjour, "Alice"})
```

### `Task` — pour du parallélisme simple

```elixir
task = Task.async(fn -> appel_api_lent() end)
result = Task.await(task)
```

Équivalent de `asyncio` Python, mais sans `async/await` partout dans le code.

---

## Niveau 5 — OTP et production

### `GenServer` — le serveur d'état

```elixir
defmodule Compteur do
  use GenServer

  def init(valeur_initiale), do: {:ok, valeur_initiale}

  def handle_call(:lire, _from, état),    do: {:reply, état, état}
  def handle_cast({:incrémenter, n}, état), do: {:noreply, état + n}
end

{:ok, pid} = GenServer.start_link(Compteur, 0)
GenServer.cast(pid, {:incrémenter, 5})
GenServer.call(pid, :lire)  # 5
```

Un `GenServer` est un processus qui garde un état et répond à des messages. C'est la brique de base de toute application Elixir sérieuse.

### `Supervisor` — tolérance aux pannes

```elixir
children = [
  {Compteur, 0}
]
Supervisor.start_link(children, strategy: :one_for_one)
```

Si `Compteur` plante, le Supervisor le **relance automatiquement**. C'est la philosophie "let it crash" — tu ne défends pas chaque ligne de code contre les exceptions, tu construis des arbres de supervision qui se rétablissent seuls.

---

## Ce que tu dois installer

```bash
# 1. Installer Erlang + Elixir via asdf (recommandé)
asdf plugin add erlang
asdf plugin add elixir
asdf install erlang latest
asdf install elixir latest

# 2. Créer un projet
mix new mon_projet
cd mon_projet

# 3. Lancer le REPL interactif
iex -S mix
```

`mix` est l'équivalent de `pip` + `poetry` + `pytest` réunis : gestion des dépendances, compilation, tests, tâches custom.

---

## Ressources dans l'ordre

1. **Elixir School** (elixirschool.com) — gratuit, traduit en français, linéaire
2. **Programming Elixir** de Dave Thomas — le livre de référence
3. **Elixir in Action** de Saša Jurić — indispensable pour OTP
4. **Phoenix LiveView** — quand tu veux faire du web

Ne commence pas Phoenix avant d'avoir une vraie maîtrise d'OTP. C'est l'erreur classique.


## Setup Elixir pour l'apprentissage

### 1. Installer Erlang + Elixir

La meilleure façon est via **asdf** — ça évite les conflits de versions.

```bash
# Installer asdf (si pas déjà fait)
brew install asdf  # macOS
# ou sur Linux : voir https://asdf-vm.com/guide/getting-started.html

# Ajouter les plugins
asdf plugin add erlang
asdf plugin add elixir

# Installer les dernières versions stables
asdf install erlang latest
asdf install elixir latest

# Définir les versions globales
asdf global erlang latest
asdf global elixir latest

# Vérifier
elixir --version
```

---

### 2. Créer un projet d'apprentissage

```bash
mix new apprentissage_elixir
cd apprentissage_elixir
```

Structure générée :
```
apprentissage_elixir/
├── lib/
│   └── apprentissage_elixir.ex   ← ton code ici
├── test/
│   └── apprentissage_elixir_test.exs
├── mix.exs                        ← config du projet (dépendances etc.)
└── README.md
```

---

### 3. Éditeur

**VSCode** — installe l'extension **ElixirLS** (identifiant : `jakebecker.elixir-ls`). C'est la référence : autocomplétion, go-to-definition, formatage, inline diagnostics.

**Zed** — le support Elixir est natif via Tree-sitter pour la syntaxe, et ElixirLS pour le LSP. Dans `settings.json` de Zed :

```json
{
  "languages": {
    "Elixir": {
      "language_servers": ["elixir-ls"]
    }
  }
}
```

Zed télécharge ElixirLS automatiquement. Si ça ne se lance pas, vérifie que `elixir` est bien dans ton `PATH`.

---

### 4. Workflow quotidien

```bash
# Le REPL — ton meilleur ami pour expérimenter
iex -S mix

# Lancer les tests
mix test

# Formater le code (à faire souvent)
mix format
```

Dans `iex`, tu peux recharger ton code sans quitter :
```elixir
recompile()
```

---

### Conseil pratique

Pour l'apprentissage, **travaille directement dans `iex`** pour tester des petites choses, et écris tes exercices dans `lib/`. Évite de créer une tonne de fichiers — un seul module par concept étudié suffit largement au début.

Bonne décision de bloquer là-dessus — c'est le cœur d'Elixir. On reprend depuis zéro avec des analogies Python.

---

## Niveau 4 — Concurrence et BEAM

### Le problème en Python

En Python, la concurrence est compliquée. Tu as trois options, toutes imparfaites :

```python
import threading  # threads OS, partagent la mémoire → risque de race condition
import asyncio    # coroutines, mais tout ton code doit être async
import multiprocessing  # vrais process, mais lourds et communication complexe
```

Le **GIL** (Global Interpreter Lock) empêche deux threads Python de tourner vraiment en parallèle.

### La solution BEAM

La BEAM (la VM d'Elixir) a ses propres **processus légers**, complètement séparés des threads OS.

- Tu peux en lancer **des millions** sans problème
- Ils **ne partagent aucune mémoire** entre eux
- Ils communiquent uniquement par **messages**
- Si l'un plante, **les autres continuent**

Pense à des **onglets de navigateur** : chaque onglet est isolé, s'il plante il n'emporte pas les autres.

---

### `spawn` — lancer un processus
```elixir
# spawn lance une fonction dans un nouveau processus
# et retourne son identifiant (pid)
pid = spawn(fn ->
  IO.puts("je tourne dans un processus séparé")
end)

IO.puts("moi je continue sans attendre")
```

Contrairement à Python où `threading.Thread` partage la mémoire, ici les deux processus sont **totalement isolés**.

---

### `send` / `receive` — les messages

Chaque processus a une **boîte aux lettres**. Tu envoies des messages dedans, le processus les lit quand il est prêt.

```elixir
# On crée un processus qui attend un message
pid = spawn(fn ->
  receive do
    {:bonjour, nom} -> IO.puts("Salut #{nom} !")
    :stop            -> IO.puts("Au revoir")
  end
end)

# On lui envoie un message depuis le processus principal
send(pid, {:bonjour, "Alice"})
# → affiche "Salut Alice !"
```

En Python l'équivalent serait une `Queue` entre threads — mais ici c'est natif dans le langage.

---

### `Task` — parallélisme simple

Pour les cas courants (lancer un calcul en parallèle et attendre le résultat), tu n'as pas besoin de `spawn` manuellement :

```elixir
# Equivalent Python de : asyncio.create_task()
task = Task.async(fn -> appel_api_lent() end)

# On fait autre chose pendant ce temps...
calcul_local()

# On attend le résultat (bloque ici si pas encore prêt)
resultat = Task.await(task)
```

```elixir
# Plusieurs tâches en parallèle — équivalent de asyncio.gather()
resultats = [url1, url2, url3]
|> Enum.map(fn url -> Task.async(fn -> fetch(url) end) end)
|> Task.await_many()
```

---

## Niveau 5 — OTP

OTP c'est un ensemble de **patterns et de bibliothèques** pour construire des applications robustes. Trois concepts à maîtriser dans l'ordre.

---

### `GenServer` — un processus avec état

Le problème : un processus `spawn` normal meurt après avoir exécuté sa fonction. Comment garder un **état persistant** dans un processus ?

Réponse : `GenServer` (Generic Server).

Pense à lui comme à un **objet Python qui tourne dans son propre thread**, mais sans partage de mémoire.

```python
# Ce que tu ferais en Python
class Compteur:
    def __init__(self):
        self.valeur = 0

    def incrementer(self, n):
        self.valeur += n

    def lire(self):
        return self.valeur
```

```elixir
# L'équivalent en Elixir avec GenServer
defmodule Compteur do
  use GenServer

  # Démarrage — définit l'état initial
  def init(valeur_initiale) do
    {:ok, valeur_initiale}
  end

  # handle_call = appel synchrone (tu attends la réponse)
  def handle_call(:lire, _from, état) do
    {:reply, état, état}
    #         ↑        ↑
    #    réponse   nouvel état
  end

  # handle_cast = appel asynchrone (tu n'attends pas)
  def handle_cast({:incrementer, n}, état) do
    {:noreply, état + n}
  end
end

# Utilisation
{:ok, pid} = GenServer.start_link(Compteur, 0)

GenServer.cast(pid, {:incrementer, 5})   # asynchrone
GenServer.cast(pid, {:incrementer, 3})   # asynchrone
GenServer.call(pid, :lire)               # synchrone → 8
```

La différence clé avec Python : l'état est **dans le processus**, pas dans une variable partagée. Impossible d'avoir deux threads qui modifient `self.valeur` en même temps.

---

### `Supervisor` — relancer automatiquement.
En Python, si une exception n'est pas attrapée dans un thread, le thread meurt silencieusement. Tu dois gérer ça manuellement.

En Elixir, le `Supervisor` surveille ses processus enfants et les **relance automatiquement** si l'un d'eux plante — sans affecter les autres.

```elixir
defmodule MonApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MonApp.Compteur,   # un GenServer
      MonApp.Cache,      # un autre GenServer
      MonApp.Logger      # encore un autre
    ]

    opts = [strategy: :one_for_one]
    #                  ↑
    # si un enfant plante → on relance uniquement lui
    # :one_for_all → on relance TOUS les enfants
    # :rest_for_one → on relance lui + tous ceux déclarés après

    Supervisor.start_link(children, opts)
  end
end
```

C'est la philosophie **"let it crash"** : au lieu de défendre chaque ligne de code avec des `try/except`, tu laisses planter et tu laisses le Supervisor réparer. C'est contre-intuitif venant de Python, mais c'est plus robuste en production.

---

### Les trois en résumé

```
Application (point d'entrée de ton programme)
└── Supervisor (surveille et relance)
    ├── GenServer A (processus avec état)
    ├── GenServer B (processus avec état)
    └── Supervisor enfant (tu peux imbriquer)
        └── GenServer C
```

Pour l'apprentissage, le bon ordre est :

1. Comprendre `spawn` + `send`/`receive` — les briques de base
2. Écrire un `GenServer` simple à la main
3. Mettre ce `GenServer` sous un `Supervisor`

---

Exemple concret : un **cache en mémoire**. Exactement comme Redis, mais intégré dans ton app Elixir. Tu stockes des résultats d'appels API pour ne pas les refaire à chaque requête.

---

## Le problème sans GenServer

```python
# En Python, tu ferais probablement ça
cache = {}  # variable globale partagée — dangereux avec des threads

def get_user(id):
    if id in cache:
        return cache[id]
    user = appel_api(id)  # lent
    cache[id] = user
    return user
```

Le problème : si deux threads accèdent à `cache` en même temps, tu as une **race condition**. Tu dois ajouter des locks manuellement. C'est là que ça devient pénible.

---

## La solution avec GenServer

Le GenServer **sérialise** tous les accès à l'état. Un seul message traité à la fois — plus de race condition, par construction.

```elixir
defmodule MonApp.Cache do
  use GenServer

  # --- API publique (ce que le reste de l'app appelle) ---

  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def get(clé) do
    GenServer.call(__MODULE__, {:get, clé})
  end

  def set(clé, valeur) do
    GenServer.cast(__MODULE__, {:set, clé, valeur})
  end

  def delete(clé) do
    GenServer.cast(__MODULE__, {:delete, clé})
  end

  # --- Callbacks internes (le GenServer lui-même) ---

  def init(_) do
    {:ok, %{}}  # état initial : une map vide
  end

  def handle_call({:get, clé}, _from, état) do
    {:reply, Map.get(état, clé), état}
    #          ↑ réponse renvoyée    ↑ état inchangé
  end

  def handle_cast({:set, clé, valeur}, état) do
    {:noreply, Map.put(état, clé, valeur)}
    #                   ↑ nouvel état
  end

  def handle_cast({:delete, clé}, état) do
    {:noreply, Map.delete(état, clé)}
  end
end
```

---

## Le Supervisor qui le surveille

```elixir
defmodule MonApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MonApp.Cache   # le Supervisor démarre et surveille le Cache
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

---

## Ce que ça donne en utilisation

```elixir
# Depuis n'importe où dans ton app

# Mettre en cache le résultat d'un appel API
user = appel_api_lent(42)
MonApp.Cache.set("user:42", user)

# Le récupérer plus tard (instantané)
MonApp.Cache.get("user:42")   # → %{id: 42, nom: "Alice"}

# Clé absente
MonApp.Cache.get("user:99")   # → nil

# Supprimer
MonApp.Cache.delete("user:42")
```

---

## Pourquoi le Supervisor change tout

Voilà le scénario qui illustre l'intérêt :

```elixir
# Imaginons que le Cache reçoit une valeur corrompue
# et lève une exception → le processus plante

# Sans Supervisor → ton app perd le cache définitivement,
# ou pire, elle crashe entièrement

# Avec Supervisor → automatiquement :
# 1. le processus Cache est détecté comme mort
# 2. il est relancé avec init(%{}) → cache vide, mais fonctionnel
# 3. le reste de l'app n'a rien vu, n'a pas planté
```---

## Un deuxième exemple pour ancrer : compteur de visiteurs

Même pattern, cas d'usage différent — un compteur global de visiteurs en temps réel.

```elixir
defmodule MonApp.Visiteurs do
  use GenServer

  def start_link(_),      do: GenServer.start_link(__MODULE__, 0, name: __MODULE__)
  def incrementer,        do: GenServer.cast(__MODULE__, :incrementer)
  def total,              do: GenServer.call(__MODULE__, :total)

  def init(n),                         do: {:ok, n}
  def handle_cast(:incrementer, n),    do: {:noreply, n + 1}
  def handle_call(:total, _from, n),   do: {:reply, n, n}
end
```

```elixir
# Quand un utilisateur arrive sur ton site
MonApp.Visiteurs.incrementer()

# Quand tu affiches le compteur
MonApp.Visiteurs.total()  # → 1042
```

---

## Ce qu'il faut retenir

`GenServer` = **une boîte qui garde un état**, accessible par messages, sans race condition possible.

`Supervisor` = **un gardien** qui relance cette boîte si elle tombe.

En Python tu aurais :
- une variable globale (fragile)
- des locks manuels (pénible)
- pas de relance automatique (tu dois le coder)

En Elixir, tout ça est résolu par le pattern `GenServer` + `Supervisor`, qui sont des briques standard du langage.