# Règles Métier — Plateforme SAÉ Recommender

> **Projet :** SAÉ BUT2 S3 (2026-2027)  
> **Thème retenu :** Langages de programmation  
> **Auteur :** Équipe SAÉ (Groupe 1-C)  
> **Statut :** Validé — Référence de conception pour la Phase 1  

---

## 1. Thème et structure d'une carte

### 1.1 Nature de l'objet
Chaque carte représente un **langage de programmation**. La plateforme permet à la communauté d'évaluer, de recommander et de confronter la popularité et la pertinence des langages selon leur domaine d'application.

### 1.2 Attributs d'une carte
Chaque carte est caractérisée par les données suivantes :

| Champ | Type | Description |
| :--- | :--- | :--- |
| `id` | Entier | Identifiant unique généré par le serveur |
| `nom` | Chaîne (50) | Nom du langage (ex. `"Python"`, `"Rust"`, `"C++"`) |
| `createur_historique` | Chaîne (100) | Concepteur d'origine (ex. `"Guido van Rossum"`, `"Bjarne Stroustrup"`) |
| `annee_creation` | Entier | Année de première parution (ex. `1991`) |
| `domaine` | Énumération | **Clé de proximité** pour les battles (`Web`, `GameDev`, `DataScience`, `Systems`, `Mobile`) |
| `description` | Chaîne (500) | Présentation synthétique du langage et de ses usages |
| `proprietaire_id` | Entier | Identifiant de l'utilisateur propriétaire courant de la carte |
| `createur_id` | Entier | Identifiant de l'utilisateur ayant créé la carte sur la plateforme |
| `valeur` (VA) | Entier | Valeur d'Appréciation courante de la carte |
| `statut` | Énumération | État courant : `ACTIVE`, `EN_BATTLE`, `AUTHENTIFIEE`, `INACTIVE` |

Par défaut, à la création, le propriétaire d'une carte est l'utilisateur qui l'a créée (`proprietaire_id = createur_id`).

---

## 2. Recommandations et calcul de la valeur (VA)

### 2.1 Recommandation (« Like »)
* Un utilisateur connecté peut recommander une carte avec le statut `ACTIVE` ou `AUTHENTIFIEE`.
* Une recommandation apporte un soutien direct à la carte.
* **Règle d'unicité :** Un utilisateur ne peut émettre qu'**une seule recommandation active** par carte.

### 2.2 Répudiation (« Annulation du like »)
* Il n'existe **pas de vote négatif (dislike)** sur la plateforme.
* Un utilisateur peut uniquement **annuler (répudier)** un like qu'il a précédemment attribué.
* Lors d'une répudiation, la contribution active est déduite de la carte, mais l'historique complet de l'action reste immuable dans la blockchain.

### 2.3 Formule de calcul de la VA (Valeur d'Appréciation)
La valeur d'une carte mesure son attractivité globale :

$$\text{VA} = \text{recommandations\_actives} + (10 \times \text{est\_authentifiee})$$

* **1 recommandation active :** `+1 VA`
* **1 répudiation :** `-1 VA`
* **Authentification de la carte :** bonus fixe permanent de `+10 VA`

---

## 3. Critères et règles des confrontations (« Battles »)

### 3.1 Critère de proximité
Deux cartes peuvent s'affronter en battle **si et seulement si elles partagent le même `domaine`**.
* *Exemple autorisé :* `C++` vs `C#` (tous deux dans le domaine `GameDev`).
* *Exemple autorisé :* `Python` vs `R` (tous deux dans le domaine `DataScience`).
* *Exemple refusé :* `Rust` (`Systems`) vs `PHP` (`Web`) $\rightarrow$ Rejet (`CARDS_NOT_COMPATIBLE`).

### 3.2 Conditions de lancement
* Les deux cartes doivent avoir le statut `ACTIVE` ou `AUTHENTIFIEE`.
* Le demandeur doit être le **propriétaire d'au moins une des deux cartes**.
* Chaque carte doit posséder une valeur minimale de départ : **$\text{VA} \ge 5$**.
* Une carte ne peut participer qu'à **une seule battle à la fois** (le serveur bascule son statut à `EN_BATTLE`).

### 3.3 Déroulement et règles de vote
* **Durée du vote :** **60 secondes** à compter de l'acceptation par le serveur.
* **Éligibilité des votants :** Tout utilisateur connecté peut voter, **à l'exception des propriétaires des deux cartes engagées**.
* **Pondération du vote :** **1 utilisateur = 1 vote** (vote égalitaire, simple et transparent).
* **Unicité :** 1 seul vote par utilisateur par battle.

### 3.4 Règle de départage en cas d'égalité (Tie-break)
Si à la fin des 60 secondes les deux cartes ont le même nombre de votes :
1. La carte possédant la **VA initiale la plus élevée** avant le lancement de la battle est déclarée gagnante.
2. Si l'égalité persiste, la carte la plus ancienne (identifiant `id` le plus petit) l'emporte.
Il y a donc **toujours un vainqueur désigné**.

### 3.5 Résolution de la battle et sort du perdant
Conformément aux exigences du cahier des charges national :
1. **Carte gagnante :** Récupère l'intégralité des **recommandations actives** associées à la carte perdante. Son compteur de victoires est incrémenté et son statut redevient `ACTIVE` (ou `AUTHENTIFIEE`).
2. **Carte perdante :** Perd ses recommandations actives et bascule immédiatement au statut **`INACTIVE`** (archivée).
3. **Immuabilité :** L'historique et les blocs antérieurs de la carte perdante restent définitivement inscrits dans la blockchain.
4. **Interdictions :** Une carte `INACTIVE` ne peut plus être recommandée ni engagée dans une nouvelle battle.

---

## 4. Légitimation, Revendication et Authentification

### 4.1 Légitimation d'un utilisateur (Processus simulé)
La légitimation permet à un utilisateur d'obtenir une reconnaissance officielle de son autorité vis-à-vis d'un langage :
* **Condition d'obtention :** Tout utilisateur ayant émis au moins **3 recommandations valides** sur la plateforme peut demander la légitimation sur un langage.
* **Effet :** L'utilisateur acquiert le statut de représentant légitime (`is_legitimate = true`), enregistré dans un bloc de la blockchain.

### 4.2 Revendication d'une carte
* **Objectif :** Permettre au représentant légitime de prendre le contrôle d'une carte créée initialement par un tiers.
* **Condition :** Seul un utilisateur légitimé peut revendiquer une carte qui appartient à un propriétaire non légitime.
* **Modalités de la transaction :** Le transfert de propriété est validé par le serveur de manière directe :
  * Le champ `proprietaire_id` devient l'identifiant du demandeur légitime.
  * Le champ `createur_id` reste inchangé pour préserver la mémoire de l'auteur d'origine.
  * L'action est inscrite dans la blockchain (`ACTION_CLAIM_CARD`).

### 4.3 Authentification d'une carte (Certification officielle)
* **Condition :** Seul le propriétaire légitime d'une carte peut déclencher son authentification.
* **Effets concrets :**
  1. Le statut de la carte passe à **`AUTHENTIFIEE`** (affichage d'un badge officiel sur le client JavaFX).
  2. La carte reçoit un bonus permanent de **`+10 VA`**.
  3. Les caractéristiques de la carte (`nom`, `domaine`, `createur_historique`, `annee_creation`) deviennent **verrouillées** et ne peuvent plus être modifiées ni contestées.

---

## 5. Règles système et gestion des erreurs

### 5.1 Limites techniques de la plateforme
* **Connexions simultanées :** Plafond fixé à **`MAX_CLIENTS = 20`** utilisateurs.
* **Limite de possession :** Un utilisateur peut posséder au maximum **10 cartes actives** simultanément.

### 5.2 Règle d'absence et déconnexion inattendue
* Si un utilisateur se déconnecte pendant une battle, le vote continue jusqu'à l'expiration des 60 secondes.
* Toute action non validée ou en cours d'envoi lors de la coupure d'un socket est abandonnée sans corrompre l'état mémoire ni la blockchain.

---

## 6. Précisions de conception (ajoutées pour la mise en œuvre)

Les règles ci-dessus restent la référence. Les points suivants ne les modifient pas : ils fixent la manière dont l'implémentation applique les cas que le texte laisse implicites. Le groupe doit les valider.

| Réf. | Précision | Justification |
| :--- | :--- | :--- |
| P1 | `est_authentifiee` est un booléen distinct de `statut`. Pendant une battle (`EN_BATTLE`), il permet de retrouver l'état `AUTHENTIFIEE` ou `ACTIVE` à la fin. | La formule de VA et la règle 3.5 en ont besoin. |
| P2 | Lors du transfert des recommandations (3.5), si un utilisateur recommande déjà la carte gagnante, sa recommandation transférée est fusionnée avec l'existante : elle n'est pas comptée deux fois. | Préserve la règle d'unicité de 2.1. Le gain de VA du gagnant peut donc être inférieur au nombre de recommandations du perdant. |
| P3 | Les votes sont inscrits dans la blockchain un par un (`ACTION_BATTLE_VOTE`), puis le résultat (`ACTION_BATTLE_RESULT`). | Les votes restent traçables et la battle peut être restaurée après un redémarrage. |
| P4 | Si le serveur redémarre pendant une battle, la battle non terminée est clôturée au démarrage avec les votes déjà enregistrés, selon les règles de 3.4. | Aucune carte ne reste bloquée à `EN_BATTLE`. |
| P5 | La « VA initiale » du départage (3.4) est la VA de chaque carte relevée au lancement et conservée dans le bloc `ACTION_BATTLE_START`. | Une répudiation pendant la battle ne change pas le départage. |
| P6 | Les « 3 recommandations valides » de 4.1 sont des recommandations **actives** de l'utilisateur au moment de la demande. | Une recommandation répudiée n'est plus valide. |
| P7 | La demande de légitimation désigne une carte (le langage visé). Le statut `is_legitimate` est porté par l'utilisateur et vaut pour toute la plateforme. | Cohérent avec 4.2 et 4.3, qui testent seulement « légitimé ou non ». |
| P8 | Une carte `EN_BATTLE` ne peut être ni revendiquée ni authentifiée. | Évite de changer propriétaire ou VA en pleine confrontation. |
| P9 | Les champs texte ne contiennent ni `\|` ni retour à la ligne (séparateurs du protocole). | Voir [protocole-applicatif-commun.md](protocole-applicatif-commun.md). |

### Points laissés ouverts

Ces questions ne sont pas tranchées par les règles actuelles. Elles sont notées pour décision avant la phase 2.

* Un utilisateur peut-il recommander sa propre carte ? Les règles ne l'interdisent pas.
* Faut-il refuser deux cartes de même `nom` ? Les règles ne l'imposent pas.
* Une carte `INACTIVE` peut-elle redevenir active ? Le cahier des charges l'autorise « selon une règle explicite » : ici, non, une carte `INACTIVE` le reste.
