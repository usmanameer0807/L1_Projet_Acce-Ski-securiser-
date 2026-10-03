# Protocole applicatif commun

Ce document est le contrat entre les clients Java et le serveur C. Toute la communication passe par des sockets TCP : aucune autre interface n'est utilisée (pas d'API REST).

## 1. Choix de conception

| Question | Choix retenu | Alternatives écartées |
| :--- | :--- | :--- |
| Transport | TCP, un socket par client | UDP : pas de fiabilité ni d'ordre garantis |
| Format des messages | Texte UTF-8, une ligne par message, champs séparés par `\|` | JSON : analyse lourde en C sans bibliothèque autorisée. Binaire : difficile à déboguer. |
| Délimitation | Fin de ligne `\n` | Préfixe de longueur : plus fragile à écrire à la main pour tester |
| Dialogue | Le client envoie une requête et attend la réponse avant d'en envoyer une autre | Requêtes en parallèle avec identifiant : complexité inutile pour la SAÉ |
| Notifications | Le serveur peut envoyer à tout moment des lignes `EVT\|…` | Interrogation périodique par le client : trafic inutile |

Le format texte permet de tester le serveur à la main avec `nc <ip> <port>`, ce qui sert aux démonstrations et aux tests.

## 2. Connexion

* Le serveur écoute sur une adresse IP et un port (par défaut `5555`), fournis au démarrage.
* Un client doit connaître cette adresse et ce port pour se connecter (configuration du client).
* Le serveur accepte au plus `MAX_CLIENTS = 20` connexions simultanées. Au-delà, il répond `ERR|LOGIN|SERVER_FULL|…` puis ferme le socket.
* Après la connexion TCP, le premier message valide est `LOGIN`. Tant que le client n'est pas identifié, toute autre commande reçoit `ERR|<CMD>|NOT_LOGGED_IN|…`.

## 3. Format général

* Une ligne = un message, terminée par `\n` (un `\r` final éventuel est ignoré).
* Longueur maximale d'une ligne : **1024 octets**. Au-delà, le serveur répond `ERR|…|LINE_TOO_LONG|…`.
* Champs séparés par `|`. Un champ texte ne contient ni `|` ni retour à la ligne. Le serveur refuse ces caractères avec `INVALID_FIELD`.
* Les noms de commandes sont en majuscules, sans espace.

### 3.1 Trois types de messages serveur

| Préfixe | Sens | Forme |
| :--- | :--- | :--- |
| `OK` | Réponse à une requête réussie | `OK\|<CMD>\|<données…>` |
| `ERR` | Réponse à une requête refusée | `ERR\|<CMD>\|<CODE>\|<message lisible>` |
| `EVT` | Notification envoyée sans requête | `EVT\|<TYPE>\|<données…>` |
| `ROW` | Ligne de données qui suit un `OK` de liste | `ROW\|<champs…>` |

Le client distingue une réponse d'un événement par le préfixe : `OK` et `ERR` répondent à la dernière requête, `EVT` est asynchrone et peut arriver entre une requête et sa réponse.

### 3.2 Réponses de liste

Les commandes `LIST_*` répondent par une ligne d'en-tête qui donne le nombre `n` de lignes, puis exactement `n` lignes `ROW` :

```text
OK|LIST_CARDS|2
ROW|1|Python|Guido van Rossum|1991|DataScience|ACTIVE|12|3|3|Langage généraliste
ROW|2|Rust|Graydon Hoare|2010|Systems|AUTHENTIFIEE|20|4|4|Langage système sûr
```

## 4. Commandes client → serveur

Les types sont : `int` entier décimal ; `texte` chaîne sans `|` ni retour à la ligne ; `domaine` parmi `Web`, `GameDev`, `DataScience`, `Systems`, `Mobile`.

### 4.1 Session

| Commande | Arguments | Réponse en cas de succès |
| :--- | :--- | :--- |
| `LOGIN` | `username` (texte, 1 à 30 caractères alphanumériques ou `_`), `kind` (`HUMAN` ou `BOT`) | `OK\|LOGIN\|user_id\|is_legitimate\|nb_cartes_actives` |
| `DISCONNECT` | aucun | `OK\|DISCONNECT`, puis le serveur ferme le socket |

À la connexion, le serveur reconstitue le contexte de l'utilisateur en parcourant la blockchain. Si aucune donnée ne le concerne, son contexte est vide. Le client lit ensuite ce contexte avec les commandes `LIST_*`.

Un nom inconnu crée l'utilisateur (action `ACTION_REGISTER_USER`). Un nom connu retrouve son identifiant et son historique. Il n'y a pas de mot de passe : l'authentification forte est hors périmètre de la SAÉ.

### 4.2 Actions (enregistrées dans la blockchain)

| Commande | Arguments | Réponse en cas de succès |
| :--- | :--- | :--- |
| `CREATE_CARD` | `nom`, `createur_historique`, `annee_creation` (int), `domaine`, `description` | `OK\|CREATE_CARD\|card_id\|block_id` |
| `RECOMMEND` | `card_id` | `OK\|RECOMMEND\|card_id\|nouvelle_va\|block_id` |
| `REPUDIATE` | `card_id` | `OK\|REPUDIATE\|card_id\|nouvelle_va\|block_id` |
| `START_BATTLE` | `card_id_a`, `card_id_b` | `OK\|START_BATTLE\|battle_id\|ends_at\|block_id` |
| `VOTE` | `battle_id`, `card_id` | `OK\|VOTE\|battle_id\|block_id` |
| `REQUEST_LEGITIMATION` | `card_id` (le langage visé) | `OK\|REQUEST_LEGITIMATION\|block_id` |
| `CLAIM_CARD` | `card_id` | `OK\|CLAIM_CARD\|card_id\|block_id` |
| `AUTHENTICATE_CARD` | `card_id` | `OK\|AUTHENTICATE_CARD\|card_id\|nouvelle_va\|block_id` |

`ends_at` est un horodatage Unix en secondes (UTC). Une réponse `OK` n'est envoyée qu'une fois le bloc miné et ajouté à la blockchain. Une action refusée n'est jamais inscrite.

### 4.3 Consultations (non enregistrées)

| Commande | Arguments | Réponse |
| :--- | :--- | :--- |
| `LIST_CARDS` | `filter` : `ALL`, `OWNED`, `CREATED`, `RECOMMENDED` | `OK\|LIST_CARDS\|n` puis `ROW\|id\|nom\|createur_historique\|annee\|domaine\|statut\|va\|proprietaire_id\|createur_id\|description` |
| `LIST_RECEIVED` | aucun : recommandations reçues par mes cartes | `OK\|LIST_RECEIVED\|n` puis `ROW\|card_id\|user_id\|username` |
| `LIST_USERS` | aucun : utilisateurs connectés | `OK\|LIST_USERS\|n` puis `ROW\|user_id\|username\|kind\|is_legitimate` |
| `LIST_BATTLES` | `filter` : `ONGOING` ou `ALL` | `OK\|LIST_BATTLES\|n` puis `ROW\|battle_id\|card_a\|card_b\|statut\|votes_a\|votes_b\|ends_at\|winner_card` |
| `GET_PROFILE` | aucun | `OK\|GET_PROFILE\|user_id\|username\|is_legitimate\|nb_cartes_actives\|nb_recos_donnees\|va_totale` |
| `VERIFY_CHAIN` | aucun | `OK\|VERIFY_CHAIN\|VALID\|nb_blocs` ou `OK\|VERIFY_CHAIN\|INVALID\|block_id\|raison` |

`statut` d'une battle : `ONGOING` ou `FINISHED`. `raison` vaut `BAD_HASH`, `BAD_PREV_HASH` ou `BAD_POW`. Une chaîne altérée n'est pas une erreur de protocole : la réponse est donc un `OK` qui rapporte l'anomalie.

L'arrêt du serveur n'est pas une commande client. Il se fait depuis le serveur (signal `SIGINT` ou commande `quit` sur la console).

## 5. Événements serveur → clients

| Événement | Destinataires | Données |
| :--- | :--- | :--- |
| `EVT\|USER_CONNECTED` | tous les autres | `user_id\|username\|kind` |
| `EVT\|USER_DISCONNECTED` | tous | `user_id\|username` |
| `EVT\|CARD_CREATED` | tous | `card_id\|nom\|domaine\|proprietaire_id` |
| `EVT\|CARD_UPDATED` | tous | `card_id\|statut\|va\|proprietaire_id` |
| `EVT\|BATTLE_STARTED` | tous | `battle_id\|card_a\|card_b\|ends_at` |
| `EVT\|VOTE_CAST` | tous | `battle_id\|votes_a\|votes_b` |
| `EVT\|BATTLE_ENDED` | tous | `battle_id\|winner_card\|loser_card\|votes_a\|votes_b\|tiebreak` |
| `EVT\|SERVER_SHUTDOWN` | tous | `message` |

`tiebreak` vaut `NONE` (victoire aux votes), `VA` (égalité départagée par la VA initiale) ou `ID` (départagée par l'identifiant). Un événement ne contient jamais le détail des votes individuels.

## 6. Codes d'erreur

La liste complète, avec les situations qui les produisent, est dans [cas-erreur.md](cas-erreur.md). Codes généraux :

| Code | Signification |
| :--- | :--- |
| `UNKNOWN_COMMAND` | Commande inconnue |
| `BAD_ARGUMENTS` | Nombre d'arguments incorrect |
| `INVALID_FIELD` | Valeur absente, trop longue, mal typée ou contenant un caractère interdit |
| `LINE_TOO_LONG` | Ligne de plus de 1024 octets |
| `NOT_LOGGED_IN` | Commande reçue avant `LOGIN` |
| `ALREADY_LOGGED_IN` | `LOGIN` reçu une seconde fois |
| `INTERNAL_ERROR` | Erreur inattendue côté serveur |

## 7. Exemple de dialogue

```text
C: LOGIN|alice|HUMAN
S: OK|LOGIN|1|false|0
C: CREATE_CARD|Python|Guido van Rossum|1991|DataScience|Langage généraliste très utilisé en science des données
S: OK|CREATE_CARD|1|2
S: EVT|CARD_CREATED|1|Python|DataScience|1
C: RECOMMEND|1
S: OK|RECOMMEND|1|1|3
C: RECOMMEND|1
S: ERR|RECOMMEND|ALREADY_RECOMMENDED|Vous recommandez déjà la carte 1
C: DISCONNECT
S: OK|DISCONNECT
```

## 8. Évolution du protocole

Toute évolution du protocole se fait par modification de ce fichier, avec l'accord des équipes client et serveur. En phase 3, ce document décrit les échanges réellement implémentés.
