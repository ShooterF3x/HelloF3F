# Be F3F — Classement Elo pour le F3F

**Be F3F** est une application web (un seul fichier HTML, sans serveur) qui calcule un classement saisonnier des pilotes de F3F à partir des résultats exportés de **F3X Vault**. Au lieu d'additionner des points de concours, elle s'appuie sur un système de **cote Elo** : chaque pilote est évalué par rapport à ceux qu'il a battus ou perdus contre, concours après concours.

Ce document explique **entièrement** le mode de calcul.

---

## Sommaire

1. [Pourquoi un classement Elo ?](#1-pourquoi-un-classement-elo-)
2. [Les données d'entrée](#2-les-données-dentrée)
3. [Principe général](#3-principe-général)
4. [Étape 1 — Le score de chaque pilote dans un concours](#4-étape-1--le-score-de-chaque-pilote-dans-un-concours)
5. [Étape 2 — Les duels : un concours = un tournoi de duels](#5-étape-2--les-duels--un-concours--un-tournoi-de-duels)
6. [Étape 3 — Calcul de la cote finale (stabilisation globale)](#6-étape-3--calcul-de-la-cote-finale-stabilisation-globale)
7. [Étape 4 — Courbe d'évolution chronologique](#7-étape-4--courbe-dévolution-chronologique)
8. [Étape 5 — Règles de classement officiel](#8-étape-5--règles-de-classement-officiel)
9. [L'indice de difficulté du concours (IDC)](#9-lindice-de-difficulté-du-concours-idc)
10. [Exemple chiffré](#10-exemple-chiffré)
11. [Paramètres du moteur](#11-paramètres-du-moteur)
12. [Utilisation de l'application](#12-utilisation-de-lapplication)
13. [Limites et points d'attention](#13-limites-et-points-dattention)

---

## 1. Pourquoi un classement Elo ?

Un classement par somme de points pose plusieurs problèmes :

- un pilote qui fait un seul concours avec un très bon plateau peut être mal comparé à un pilote qui en fait cinq ;
- un concours très relevé et un concours plus modeste rapportent les mêmes points « bruts » ;
- le résultat dépend du nombre de concours disputés.

Le système Elo, utilisé aux échecs, au go ou au tennis, répond à une question simple :

> **Contre qui a-t-on gagné, et contre qui a-t-on perdu ?**

Battre un pilote très fort rapporte beaucoup. Battre un pilote faible rapporte peu. Perdre contre un pilote faible coûte cher. Le classement reflète donc la **force relative** des pilotes, pas seulement leurs points.

---

## 2. Les données d'entrée

L'application lit des fichiers **CSV exportés de F3X Vault**. Pour chaque fichier, elle extrait :

| Champ CSV | Utilisation |
|---|---|
| `Event_Name` | Nom du concours (sinon : nom du fichier) |
| `Number_Rounds` | Nombre de manches (12 par défaut) |
| `End_Date` ou `Start_Date` | Date du concours (détermine la saison et l'ordre chronologique) |
| `Pilot_Name` | Identifiant du pilote (converti en **majuscules**) |
| `Percent` | Pourcentage du pilote (base du score) |
| `Total Points` | Points totaux (utilisé uniquement si `Percent` est absent) |
| `Position` | Position affichée dans la page du concours |

Un pilote est identifié **uniquement par son nom en majuscules** : `Dupont Jean` et `DUPONT JEAN` sont le même pilote, mais `DUPONT J.` serait un pilote différent. Il faut donc veiller à une orthographe cohérente d'un concours à l'autre.

Un même concours (même nom et même date) réimporté **remplace** l'ancien : il n'y a pas de doublon.

---

## 3. Principe général

Le calcul se fait **saison par saison** (année civile de la date du concours). Pour une saison donnée :

```
CSV des concours
      │
      ▼
 Score de chaque pilote (Percent × 10)
      │
      ▼
 Tous les pilotes d'un concours s'affrontent en duels
      │
      ├──► Cote Elo FINALE (stabilisation globale, 15 itérations)  ──► CLASSEMENT
      │
      └──► Courbe d'ÉVOLUTION (concours après concours, K = 24)   ──► GRAPHIQUES
```

Il y a donc **deux calculs distincts** :

- l'un, **global**, donne la cote qui sert au classement ;
- l'autre, **chronologique**, sert uniquement à tracer l'évolution de chaque pilote.

---

## 4. Étape 1 — Le score de chaque pilote dans un concours

Pour chaque pilote ayant participé à un concours :

```
score = Percent × 10          (si la colonne Percent existe)
score = Total Points          (sinon)
```

Exemple : un pilote à `95.40 %` obtient un score de `954`. Le vainqueur d'un concours, à 100 %, obtient `1000`.

Seuls les scores **strictement positifs** sont retenus. Un pilote à 0 est considéré comme absent du concours.

> **À retenir :** pour la cote Elo, seul l'**ordre** des scores dans un concours compte, pas leur valeur absolue. Finir devant quelqu'un, c'est « gagner un duel » ; l'écart de points n'est pas utilisé dans le calcul. Les scores bruts sont en revanche affichés sur les graphiques de profil.

---

## 5. Étape 2 — Les duels : un concours = un tournoi de duels

Dans un concours de `n` pilotes, les pilotes sont classés du meilleur score au plus faible. Puis **chaque pilote est comparé à chacun des autres** : cela fait `n × (n − 1) / 2` duels.

Pour chaque duel, le mieux classé est le **gagnant**, l'autre le **perdant**.

### Probabilité de victoire attendue

Pour un duel entre un pilote A (cote `Ra`) et un pilote B (cote `Rb`), la probabilité attendue que A batte B est la formule Elo classique :

```
Ea = 1 / (1 + 10^((Rb − Ra) / 400))
Eb = 1 / (1 + 10^((Ra − Rb) / 400))      (Ea + Eb = 1)
```

- Deux pilotes de même cote : `E = 0,5` (50 %).
- Un pilote avec 400 points d'avance : `E ≈ 0,91` (91 %).

### Gain et perte

- Le **gagnant** reçoit : `+ K × (1 − E_gagnant)`
- Le **perdant** reçoit : `− K × E_perdant`

Conséquences :

| Situation | Effet |
|---|---|
| Le favori bat l'outsider | Petit gain, petite perte |
| L'outsider bat le favori | **Gros gain**, grosse perte |
| Duel entre pilotes de même niveau | Gain et perte moyens |

Chaque duel est **à somme nulle** : ce que le gagnant gagne, le perdant le perd exactement. La cote moyenne de l'ensemble des pilotes reste donc autour de **1200**, qui est la cote de départ.

---

## 6. Étape 3 — Calcul de la cote finale (stabilisation globale)

C'est le cœur de la méthode et ce qui la distingue d'un Elo « classique ».

### Le problème d'un Elo classique

Dans un Elo classique, on traite les concours **un par un dans l'ordre de leur date**. Le résultat final dépend alors de l'ordre dans lequel on a traité les données : les premiers concours sont calculés avec des cotes encore imprécises (tout le monde à 1200). Changer l'ordre d'import ou l'ordre des dates changerait le classement.

### La solution Be F3F : tout calculer ensemble

Be F3F traite **tous les concours de la saison simultanément**, de façon répétée, jusqu'à stabilisation :

```
Tous les pilotes démarrent à 1200

Répéter 15 fois :
    1. Pour TOUS les concours de la saison, pour TOUS les duels :
         calculer le gain/perte avec les cotes actuelles
         (K effectif = 32 × 0,1 = 3,2 par duel)
    2. Additionner les gains/pertes de chaque pilote
    3. Appliquer tous les changements EN MÊME TEMPS
       (avec un plancher à 800 points)
```

Détails importants :

- **Mise à jour simultanée.** À chaque itération, les gains et pertes sont calculés à partir des cotes de **début d'itération**, puis appliqués d'un seul coup. Aucun pilote n'est avantagé par l'ordre de traitement.
- **Petit pas (`K = 32 × 0,1 = 3,2`).** Chaque duel ne déplace la cote que très légèrement, mais comme il y a 15 itérations et de nombreux duels, la cote progresse par petites touches vers une valeur stable.
- **Plancher à 800.** Aucune cote ne peut descendre sous 800 points.
- **Aucune dépendance à la chronologie.** Résultat : la cote finale est **identique quel que soit l'ordre d'import des fichiers**. C'est ce qu'annonce l'application : « classement stable et indépendant de l'ordre d'importation ».

La cote obtenue à la fin de la 15ᵉ itération est la **cote Elo officielle** du pilote pour la saison, arrondie à l'entier.

### Ce que la cote reflète

Un pilote monte quand il **bat régulièrement des pilotes eux-mêmes bien classés**, sur l'ensemble des concours de la saison. Comme les duels sont calculés entre tous les participants d'un même concours, les pilotes qui se sont croisés dans des concours communs servent de « pont » pour comparer des pilotes qui ne se sont jamais rencontrés directement.

---

## 7. Étape 4 — Courbe d'évolution chronologique

La cote globale ne dit pas **comment** le pilote y est arrivé. Pour cela, un second calcul, purement chronologique, reconstitue l'historique :

```
Tous les pilotes démarrent à 1200

Pour chaque concours, dans l'ordre des dates :
    - classer les participants par score
    - pour chaque duel (gagnant / perdant) :
          gain/perte = 24 × (résultat − probabilité attendue)
      calculé à partir des cotes AVANT le concours
    - appliquer tous les changements du concours en même temps
    - enregistrer la cote de chaque pilote après ce concours
```

Ici `K = 24` et il n'y a **pas d'itération** : on suit la progression réelle concours après concours.

Ces valeurs alimentent le **graphique de profil** de chaque pilote :

- **Courbe bleue pleine (axe de droite)** : cote Elo après chaque concours ;
- **Courbe rouge pointillée (axe de gauche)** : score brut F3X Vault (`Percent × 10`) du pilote à chaque concours.

> ⚠️ La courbe d'évolution et la cote globale utilisent des paramètres différents (`K = 24` sans itération contre `K = 3,2` sur 15 itérations). Le dernier point de la courbe n'est donc **pas forcément égal** à la cote affichée dans le classement. La courbe montre une **tendance** ; le classement officiel repose sur la cote globale.

---

## 8. Étape 5 — Règles de classement officiel

### Seuil de participation

Un pilote doit avoir disputé **au moins 3 concours** dans la saison (`MIN_CONTESTS_REQUIRED = 3`) pour être **classé officiellement**.

### Tri

1. Les pilotes **officiels** apparaissent en premier, triés par **cote Elo décroissante** ;
2. puis les pilotes **non classés (NC)**, également triés par cote décroissante.

### Affichage

| Statut | Rang | Affichage |
|---|---|---|
| ≥ 3 concours | 1, 2, 3… | « Officiel » |
| < 3 concours | `-` | « Non classé (< 3) », ligne grisée |

Les pilotes non classés **participent quand même aux calculs** (ils comptent comme adversaires et influencent les cotes des autres). Ils n'apparaissent simplement pas dans le rang officiel.

---

## 9. L'indice de difficulté du concours (IDC)

Chaque concours se voit attribuer un indicateur de **force de plateau** :

```
Elo du concours   = moyenne des cotes Elo des pilotes participants
Elo moyen saison  = moyenne des « Elo du concours » de tous les concours
IDC               = Elo du concours / Elo moyen saison
```

| IDC | Interprétation |
|---|---|
| > 1 | Plateau plus fort que la moyenne de la saison |
| ≈ 1 | Plateau dans la moyenne |
| < 1 | Plateau plus faible que la moyenne |

Il est visible dans l'onglet **Audit & Transparence** et dans la page de chaque concours.

**Remarque d'honnêteté :** dans la version actuelle, l'IDC est un **indicateur de transparence**. Il n'est pas réinjecté comme coefficient multiplicateur dans le calcul de la cote. La difficulté du plateau est déjà prise en compte **implicitement** par le système Elo lui-même : battre des pilotes à forte cote rapporte plus, donc un plateau relevé rapporte naturellement plus. L'IDC permet simplement de **vérifier et d'expliquer** ce phénomène.

---

## 10. Exemple chiffré

Un concours avec trois pilotes, tous à 1200 au départ :

| Pilote | Percent | Score |
|---|---|---|
| A | 100,0 % | 1000 |
| B | 96,5 % | 965 |
| C | 91,0 % | 910 |

Ordre : A > B > C. Duels : A bat B, A bat C, B bat C.

**Première itération de la stabilisation** (tous à 1200, donc `E = 0,5` pour chaque duel, `K = 3,2`) :

| Duel | Gagnant | Perdant |
|---|---|---|
| A bat B | A : +3,2 × 0,5 = **+1,6** | B : −1,6 |
| A bat C | A : **+1,6** | C : −1,6 |
| B bat C | B : **+1,6** | C : −1,6 |

Total après l'itération 1 :

- A : `+3,2` → 1203,2
- B : `−1,6 + 1,6 = 0` → 1200,0
- C : `−3,2` → 1196,8

À l'itération 2, A étant devenu favori, ses victoires rapportent un peu moins et celles de C, plus rare, un peu plus : l'écart se creuse de moins en moins vite. Au bout de 15 itérations, les écarts sont stabilisés et reflètent la hiérarchie A > B > C.

Avec de vrais concours à 20 ou 30 pilotes et plusieurs concours dans la saison, les cotes se répartissent de façon beaucoup plus fine, en tenant compte de tous les croisements entre pilotes.

---

## 11. Paramètres du moteur

| Paramètre | Valeur | Rôle |
|---|---|---|
| Cote de départ | **1200** | Cote de tout pilote avant calcul |
| Échelle Elo | **400** | Dans la formule de probabilité |
| K global (pas par duel) | **32 × 0,1 = 3,2** | Amplitude d'un duel dans la stabilisation |
| Itérations de stabilisation | **15** | Nombre de passes sur tous les concours |
| Plancher de cote | **800** | Cote minimale possible |
| K chronologique | **24** | Amplitude d'un duel dans la courbe d'évolution |
| Concours minimum pour être classé | **3** | Seuil du classement officiel |
| Score brut | **Percent × 10** | Conversion du pourcentage F3X Vault |

Ces valeurs sont des constantes en tête de code (`MIN_CONTESTS_REQUIRED`) ou dans la fonction `renderActiveSeason()`, et peuvent être ajustées.

---

## 12. Utilisation de l'application

### Chargement des données

L'application cherche les concours dans cet ordre :

1. **Mémoire du navigateur** (`localStorage`) : données déjà importées lors d'une précédente visite ;
2. **Dépôt GitHub** : renseigner `GITHUB_OWNER`, `GITHUB_REPO` et `GITHUB_FOLDER` (par défaut `data/2026`) pour charger automatiquement tous les `.csv` du dossier ;
3. **Fichiers locaux** : liste `LOCAL_CSV_FILES` ;
4. **Import manuel** : onglet *Gestion des Concours*, en cliquant ou en déposant des CSV.

Le bouton **Actualiser** relance le chargement automatique.

### Onglets

| Onglet | Contenu |
|---|---|
| Classement Général Elo | Tableau officiel de la saison |
| Profils & Évolution | Fiche de chaque pilote avec graphique cote + score brut |
| Audit & Transparence | Elo moyen et IDC de chaque concours |
| Notice & Formules | Philosophie du système |
| Gestion des Concours | Import de CSV, suppression d'un concours erroné |
| *Un onglet par concours* | Résultats détaillés (position, points, pourcentage, IDC) |

### Export

Le bouton **Exporter Excel** génère un fichier `.xls` multi-feuilles : classement Elo, coefficients IDC, et une feuille par concours.

### Saisons

Le sélecteur de saison permet de consulter chaque année séparément. **Chaque saison repart de zéro** : tous les pilotes recommencent à 1200.

---

## 13. Limites et points d'attention

- **Seul l'ordre compte.** Gagner de 0,1 point ou de 50 points est équivalent pour la cote Elo.
- **Orthographe des noms.** Un même pilote écrit différemment est compté comme deux pilotes.
- **Cote relative.** Une cote de 1300 n'a de sens que par rapport aux autres pilotes de la même saison ; elle n'est pas comparable d'une année sur l'autre.
- **Effet du nombre d'adversaires.** Les gains s'additionnent sur tous les duels : les pilotes qui affrontent beaucoup d'adversaires voient leur cote évoluer davantage que ceux qui participent à de petits concours.
- **Cote globale ≠ courbe.** Les deux calculs sont volontairement distincts (voir [section 7](#7-étape-4--courbe-dévolution-chronologique)).
- **IDC informatif.** Il n'intervient pas directement dans le calcul de la cote (voir [section 9](#9-lindice-de-difficulté-du-concours-idc)).
- **Données locales.** Les données importées manuellement sont stockées dans le navigateur ; vider le cache les efface. Pour un classement partagé et durable, utiliser le chargement GitHub.

---

## En résumé

> Be F3F transforme chaque concours en une série de duels entre tous les pilotes présents, puis calcule, sur l'ensemble de la saison et de façon indépendante de l'ordre d'import, une cote Elo qui mesure la force relative de chacun. Un classement officiel exige au moins 3 concours, et l'évolution chronologique de chaque pilote reste consultable en parallèle.
