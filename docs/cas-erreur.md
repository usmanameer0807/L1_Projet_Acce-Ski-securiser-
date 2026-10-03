# Cas d'erreur

Ce document liste les cas d'erreur traités par la plateforme, leur code protocolaire et le comportement attendu. Une requête refusée n'est **jamais** inscrite dans la blockchain.

## 1. Erreurs de protocole

| Situation | Code | Comportement serveur |
| :--- | :--- | :--- |
| Commande inconnue | `UNKNOWN_COMMAND` | Réponse `ERR`, la connexion reste ouverte |
| Nombre d'arguments incorrect | `BAD_ARGUMENTS` | Réponse `ERR` |
| Champ vide, trop long, non numérique ou contenant `\|` | `INVALID_FIELD` | Réponse `ERR` |
| Domaine hors liste | `INVALID_DOMAIN` | Réponse `ERR` |
| Ligne de plus de 1024 octets | `LINE_TOO_LONG` | Réponse `ERR`, ligne ignorée |
| Commande avant `LOGIN` | `NOT_LOGGED_IN` | Réponse `ERR` |
| Second `LOGIN` | `ALREADY_LOGGED_IN` | Réponse `ERR` |

## 2. Erreurs de connexion

| Situation | Code | Comportement |
| :--- | :--- | :--- |
| 20 clients déjà connectés | `SERVER_FULL` | `ERR\|LOGIN\|SERVER_FULL\|…` puis fermeture du socket |
| Nom d'utilisateur déjà connecté | `USERNAME_TAKEN` | Réponse `ERR`, socket conservé pour un nouvel essai |
| Client coupé brutalement (socket fermé sans `DISCONNECT`) | — | Le thread libère le socket et l'emplacement. La requête en cours est abandonnée, la blockchain n'est pas modifiée. |

## 3. Erreurs métier

### 3.1 Cartes

| Situation | Code |
| :--- | :--- |
| Carte inexistante | `CARD_NOT_FOUND` |
| L'utilisateur possède déjà 10 cartes actives (création ou revendication) | `CARD_LIMIT_REACHED` |
| Recommandation d'une carte `INACTIVE` ou `EN_BATTLE` | `CARD_NOT_RECOMMENDABLE` |
| Recommandation d'une carte déjà recommandée par cet utilisateur | `ALREADY_RECOMMENDED` |
| Répudiation sans recommandation active de cet utilisateur sur la carte | `NO_ACTIVE_RECOMMENDATION` |

### 3.2 Battles

| Situation | Code |
| :--- | :--- |
| Même carte donnée deux fois | `INVALID_FIELD` |
| Cartes de domaines différents | `CARDS_NOT_COMPATIBLE` |
| Carte `INACTIVE` ou déjà `EN_BATTLE` | `CARD_NOT_AVAILABLE` |
| Demandeur propriétaire d'aucune des deux cartes | `NOT_CARD_OWNER` |
| Une des cartes a une VA inférieure à 5 | `VA_TOO_LOW` |
| Battle inexistante | `BATTLE_NOT_FOUND` |
| Vote après la fin des 60 secondes | `BATTLE_CLOSED` |
| Vote pour une carte absente de la battle | `CARD_NOT_IN_BATTLE` |
| Vote d'un propriétaire d'une des deux cartes | `VOTE_FORBIDDEN_OWNER` |
| Second vote du même utilisateur | `ALREADY_VOTED` |

### 3.3 Légitimation, revendication, authentification

| Situation | Code |
| :--- | :--- |
| Légitimation demandée par un utilisateur déjà légitimé | `ALREADY_LEGITIMATE` |
| Moins de 3 recommandations actives | `NOT_ENOUGH_RECOMMENDATIONS` |
| Action réservée aux légitimés demandée par un non-légitimé | `NOT_LEGITIMATE` |
| Revendication d'une carte dont le propriétaire est déjà légitimé | `OWNER_IS_LEGITIMATE` |
| Revendication d'une carte qu'on possède déjà | `ALREADY_OWNER` |
| Authentification d'une carte dont on n'est pas propriétaire | `NOT_CARD_OWNER` |
| Authentification d'une carte déjà authentifiée | `ALREADY_AUTHENTICATED` |
| Revendication ou authentification d'une carte `INACTIVE` ou `EN_BATTLE` | `CARD_NOT_AVAILABLE` |

## 4. Erreurs système

| Situation | Comportement |
| :--- | :--- |
| Base PostgreSQL indisponible au démarrage | Le serveur considère qu'aucune sauvegarde n'existe : il crée une blockchain avec son bloc genesis et fonctionne en mémoire (mode dégradé), avec un avertissement. |
| Base indisponible pendant le fonctionnement | L'action reste valide : le bloc est ajouté à la blockchain en mémoire. Le serveur mémorise le dernier bloc sauvegardé et réessaie la sauvegarde des blocs en attente à l'action suivante, dans l'ordre. |
| Échec de l'écriture en base au milieu d'une transaction | La transaction est annulée : ni le bloc ni l'état dérivé ne sont écrits en base. Le bloc reste en attente de sauvegarde. |
| Conflit pendant le minage (un autre bloc a été ajouté entre-temps) | Le minage reprend avec le nouveau hash précédent, sans erreur visible pour le client. |
| Action devenue invalide entre la validation et l'ajout du bloc | Le serveur revalide au moment d'ajouter le bloc. Si l'action n'est plus valide, il répond avec l'erreur métier correspondante et le bloc n'est pas ajouté. |
| Bloc altéré détecté par `VERIFY_CHAIN` | Réponse `OK\|VERIFY_CHAIN\|INVALID\|…` avec le numéro du bloc et la raison. Le serveur continue de fonctionner. |
| Mémoire insuffisante | Réponse `INTERNAL_ERROR`, aucun changement d'état |
| Arrêt du serveur demandé | Les clients reçoivent `EVT\|SERVER_SHUTDOWN`. Les threads sont arrêtés proprement, les sockets et la connexion à la base sont fermés. |

## 5. Absence d'un utilisateur

Conformément aux règles métier (5.2) :

* Si un propriétaire se déconnecte pendant une battle, la battle continue jusqu'à la fin des 60 secondes et son résultat est appliqué.
* Une action qui nécessite l'intervention d'un utilisateur absent est refusée. Il n'existe pas d'action en attente d'acceptation dans le périmètre actuel, car la revendication est validée directement par le serveur.

## 6. Messages affichés dans l'interface

Le client traduit chaque code en un message lisible, sans montrer le code brut à l'utilisateur. La table de correspondance est détaillée dans [architecture-client.md](architecture-client.md).
