
## Exercice 1 : Équations différentielles du premier ordre
### 1. (1 + e^x)yy' = e^x ; \quad y(0) = 1
Il s'agit d'une équation à variables séparables.


En intégrant les deux membres par rapport à leurs variables respectives :


Appliquons la condition initiale y(0) = 1 pour déterminer C :


L'équation devient :


Puisque la condition initiale exige y(0) = 1 > 0, nous devons retenir la racine positive.
**Solution :**

### 2. xy' = \sqrt{x^2 - y^2} + y
Cette équation est homogène. Supposons une résolution sur un intervalle où x > 0. On pose y(x) = x u(x), ce qui implique y' = u'x + u.
En substituant dans l'équation :


Puisque x \neq 0 par hypothèse, on divise par x (et x^2) :


Cette nouvelle forme est à variables séparables. En intégrant :


**Solution :**

### 3. y' + 2xy = y^2e^{x^2}
Il s'agit d'une équation de Bernoulli avec n=2. On effectue le changement de variable u = y^{1-2} = y^{-1}, ce qui donne u' = -y^{-2}y'.
Divisons l'équation initiale par y^2 :


Substituons avec u et u' :


Il s'agit maintenant d'une équation différentielle linéaire du premier ordre. Le facteur intégrant est \mu(x) = e^{\int -2x \, dx} = e^{-x^2}. En multipliant l'équation par \mu(x) :


En intégrant :


En revenant à y = u^{-1} :
**Solution :**

## Exercice 2 : Équations du deuxième ordre
### 1. y'' + 2y' + 5y = e^{-x}\cos(2x)
**Équation homogène :** y'' + 2y' + 5y = 0.
L'équation caractéristique est r^2 + 2r + 5 = 0. Son discriminant est \Delta = 4 - 20 = -16 = (4i)^2.
Les racines complexes sont r = -1 \pm 2i.
La solution homogène est :


**Solution particulière :**
Le second membre est \text{Re}(e^{(-1+2i)x}). Nous résolvons z'' + 2z' + 5z = e^{(-1+2i)x} et prendrons y_p = \text{Re}(z_p).
Puisque -1+2i est racine simple de l'équation caractéristique (cas de résonance), on cherche z_p sous la forme z_p(x) = Ax e^{(-1+2i)x}.
Ses dérivées sont :


En substituant dans l'équation complexe et en factorisant par A e^{(-1+2i)x} :


Donc z_p(x) = -\frac{i}{4} x e^{-x} (\cos(2x) + i\sin(2x)) = \frac{x e^{-x}}{4} ( -i\cos(2x) + \sin(2x) ).
La partie réelle est y_p(x) = \frac{x}{4} e^{-x} \sin(2x).
**Solution :**

### 2. y'' - 2y' + y = \frac{e^x}{x^2 + 1}
**Équation homogène :** r^2 - 2r + 1 = 0 \implies (r-1)^2 = 0. La racine double est r=1.


**Solution particulière par variation des constantes :**
On pose y_p(x) = u_1(x)e^x + u_2(x)xe^x. Les fonctions u_1' et u_2' satisfont le système :


En soustrayant la première équation à la seconde :


En remplaçant u_2' dans la première équation :


On obtient y_p(x) = -\frac{1}{2}\ln(x^2+1)e^x + x\arctan(x)e^x.
**Solution :**

## Exercice 3 : Systèmes d'équations différentielles
### 1) Système de dimension 2

Par dérivation de la première équation : x'' = y' + 1.
En substituant y' = x - t :


L'équation homogène associée est x'' - x = 0 \implies x_h(t) = C_1 e^t + C_2 e^{-t}.
Pour la solution particulière, on pose x_p(t) = At + B. En identifiant -At - B = -t + 1, on trouve A = 1 et B = -1.
Donc x(t) = C_1 e^t + C_2 e^{-t} + t - 1.
En utilisant la première équation y = x' - t, on trouve y(t) = C_1 e^t - C_2 e^{-t} + 1 - t.
**Solution :**

### 2) Système de dimension 3
Le système s'écrit X' = MX avec M = \begin{pmatrix} -1 & 1 & 1 \\ 1 & -1 & 1 \\ 1 & 1 & -1 \end{pmatrix}.
Le polynôme caractéristique est P(\lambda) = \det(M - \lambda I) = 0.
En remarquant que la somme des éléments de chaque ligne est 1, \lambda_1 = 1 est une valeur propre évidente avec pour vecteur propre associé V_1 = (1, 1, 1)^T.
La trace de la matrice est -3, donc la somme des valeurs propres est \lambda_1 + \lambda_2 + \lambda_3 = -3. Puisque la matrice est symétrique (valeurs propres réelles) et par identification, \lambda_2 = \lambda_3 = -2 (racine double).
Pour \lambda = -2, l'espace propre est défini par x+y+z = 0. Une base de vecteurs propres est V_2 = (1, -1, 0)^T et V_3 = (1, 0, -1)^T.
**Solution :**

## Exercice 4 : Séries entières
Résolution de y'' - xy = 0 (Équation d'Airy).
On suppose une solution de la forme y(x) = \sum_{n=0}^{\infty} a_n x^n.
Ses dérivées sont :


On injecte ces séries dans l'équation différentielle :


En décalant l'indice de la seconde somme (k = n+1) :


On extrait le terme n=0 de la première somme pour unifier les indices à partir de n=1 :


Par identification avec le polynôme nul, on obtient :
 1.  2. Posons n+2 = 3m, 3m+1, ou 3m+2 pour déterminer les suites :
 * Si l'indice est un multiple de 3 (3m) : a_3 = \frac{a_0}{2 \cdot 3}, a_6 = \frac{a_3}{5 \cdot 6}, d'où a_{3m} = a_0 \prod_{j=1}^m \frac{1}{3j(3j-1)}.
 * Si l'indice est de la forme 3m+1 : a_4 = \frac{a_1}{3 \cdot 4}, a_7 = \frac{a_4}{6 \cdot 7}, d'où a_{3m+1} = a_1 \prod_{j=1}^m \frac{1}{3j(3j+1)}.
 * Si l'indice est de la forme 3m+2 : puisque a_2 = 0, par récurrence, a_{3m+2} = 0.
Le rayon de convergence est infini.
**Solution :**

