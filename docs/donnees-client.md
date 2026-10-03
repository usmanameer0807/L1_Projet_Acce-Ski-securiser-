# Données du client

Responsables : Usman, Omar. Ce document décrit les classes Java du client, leurs attributs et leurs responsabilités. Il complète [architecture-client.md](architecture-client.md).

## 1. Énumérations

```java
public enum Domain { Web, GameDev, DataScience, Systems, Mobile }

public enum CardStatus { ACTIVE, EN_BATTLE, AUTHENTIFIEE, INACTIVE }

public enum UserKind { HUMAN, BOT }

public enum BattleStatus { ONGOING, FINISHED }
```

## 2. Classes du modèle

Les propriétés destinées à l'interface sont des propriétés JavaFX (`StringProperty`, `IntegerProperty`…) pour que les vues se mettent à jour automatiquement.

### `Card`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `id` | `int` | Identifiant donné par le serveur |
| `nom` | `String` | Nom du langage |
| `createurHistorique` | `String` | Concepteur d'origine |
| `anneeCreation` | `int` | Année de première parution |
| `domaine` | `Domain` | Clé de proximité |
| `description` | `String` | Description |
| `proprietaireId` | `IntegerProperty` | Propriétaire courant |
| `createurId` | `int` | Créateur de la carte |
| `valeur` | `IntegerProperty` | VA courante |
| `statut` | `ObjectProperty<CardStatus>` | Statut courant |

Méthodes : `isAuthenticated()` (vrai si `AUTHENTIFIEE`), `isRecommendable()` (vrai si `ACTIVE` ou `AUTHENTIFIEE`), `fromRow(String[])` (construction à partir d'une ligne `ROW`).

### `UserInfo`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `id` | `int` | Identifiant |
| `username` | `String` | Nom |
| `kind` | `UserKind` | Humain ou bot |
| `legitimate` | `BooleanProperty` | Statut de légitimation |

### `Battle`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `id` | `int` | Identifiant |
| `cardA`, `cardB` | `int` | Cartes engagées |
| `votesA`, `votesB` | `IntegerProperty` | Compteurs de votes |
| `endsAt` | `long` | Fin du vote, secondes Unix |
| `status` | `ObjectProperty<BattleStatus>` | En cours ou terminée |
| `winnerCard` | `int` | 0 tant que non terminée |
| `hasVoted` | `BooleanProperty` | Vrai si l'utilisateur local a déjà voté |

Méthodes : `remainingSeconds(long now)`, `canVote(UserInfo me, ClientModel model)` (faux pour un propriétaire des cartes engagées, après un vote ou si la battle est terminée).

### `Recommendation`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `cardId` | `int` | Carte recommandée |
| `userId` | `int` | Utilisateur qui recommande |
| `username` | `String` | Nom, pour l'affichage des recommandations reçues |

### `ClientModel`

État unique de l'application, partagé entre les contrôleurs.

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `me` | `ObjectProperty<UserInfo>` | Utilisateur local |
| `allCards` | `ObservableList<Card>` | Toutes les cartes connues |
| `onlineUsers` | `ObservableList<UserInfo>` | Utilisateurs connectés |
| `battles` | `ObservableList<Battle>` | Battles en cours et terminées |
| `receivedRecommendations` | `ObservableList<Recommendation>` | Recommandations reçues par mes cartes |
| `eventLog` | `ObservableList<String>` | Journal affiché sur l'écran principal |

Méthodes principales : `ownedCards()` et `createdCards()` (listes filtrées dérivées de `allCards`, sans copie), `applyEvent(ProtocolMessage)`, `reloadFromServer(ServerConnection)` (enchaîne `GET_PROFILE`, `LIST_CARDS`, `LIST_RECEIVED`, `LIST_USERS`, `LIST_BATTLES` après la connexion).

## 3. Classes réseau

### `ProtocolMessage`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `type` | `enum` (`OK`, `ERR`, `EVT`) | Type de message |
| `command` | `String` | Commande concernée (pour `OK`/`ERR`) ou type d'événement |
| `fields` | `List<String>` | Champs suivant la commande |
| `rows` | `List<List<String>>` | Lignes `ROW` d'une réponse de liste |
| `errorCode` | `String` | Code d'erreur (pour `ERR`) |

### `ProtocolParser`

Méthodes statiques : `parse(String line)` renvoie un message partiel ; `expectedRows(ProtocolMessage)` renvoie le nombre de lignes `ROW` attendues après un `OK|LIST_…|n` ; `buildRequest(String cmd, Object... args)` assemble une requête en refusant tout champ contenant `|` ou un retour à la ligne.

### `ServerConnection`

| Attribut | Type | Description |
| :--- | :--- | :--- |
| `socket` | `Socket` | Connexion TCP |
| `out` | `PrintWriter` | Envoi (UTF-8, `\n`) |
| `reader` | `BufferedReader` | Lecture (UTF-8) |
| `readerThread` | `Thread` | Thread de lecture (démon) |
| `pending` | `CompletableFuture<ProtocolMessage>` | Requête en attente de réponse |
| `listener` | `ServerEventListener` | Destinataire des événements `EVT` et des coupures |

Méthodes : `connect(host, port)`, `send(String line)` (renvoie un `CompletableFuture`), `close()`.

Interface `ServerEventListener` : `onEvent(ProtocolMessage)` et `onDisconnected(String reason)`.

## 4. Contrôleurs

| Classe | Écran | Rôle |
| :--- | :--- | :--- |
| `LoginController` | Login | Contrôle les champs, ouvre la connexion, envoie `LOGIN`, charge le contexte puis navigue |
| `MainController` | Principal | Affiche utilisateurs connectés et journal, gère la navigation et la déconnexion |
| `CardsController` | Cartes | Filtres, détail, boutons Recommander, Répudier, Revendiquer, Authentifier, Lancer une battle |
| `CreateCardController` | Création | Contrôle du formulaire, envoie `CREATE_CARD` |
| `BattlesController` | Battles | Liste, compte à rebours, boutons de vote |
| `ProfileController` | Profil | Informations, `REQUEST_LEGITIMATION`, `VERIFY_CHAIN` |

Les boutons sont activés ou désactivés par des liaisons sur le modèle : par exemple « Revendiquer » est actif si l'utilisateur est légitimé, que la carte est disponible et qu'il n'en est pas propriétaire. Ces règles sont un confort d'interface ; le serveur refuse de toute façon une action interdite.

## 5. Correspondance interface → protocole

| Action de l'utilisateur | Requête envoyée | Mise à jour du modèle à la réponse `OK` |
| :--- | :--- | :--- |
| Se connecter | `LOGIN\|nom\|HUMAN` | `me`, puis rechargement complet du contexte |
| Créer une carte | `CREATE_CARD\|…` | La carte arrive aussi par `EVT\|CARD_CREATED` |
| Recommander | `RECOMMEND\|id` | VA de la carte, liste « recommandées » |
| Répudier | `REPUDIATE\|id` | VA de la carte, liste « recommandées » |
| Lancer une battle | `START_BATTLE\|a\|b` | La battle arrive par `EVT\|BATTLE_STARTED` |
| Voter | `VOTE\|battle\|carte` | `hasVoted = true` |
| Demander la légitimation | `REQUEST_LEGITIMATION\|id` | `me.legitimate = true` |
| Revendiquer | `CLAIM_CARD\|id` | Propriétaire de la carte |
| Authentifier | `AUTHENTICATE_CARD\|id` | Statut et VA de la carte |
| Vérifier la blockchain | `VERIFY_CHAIN` | Message « valide » ou numéro du bloc altéré |
| Se déconnecter | `DISCONNECT` | Retour au login |

Les cartes, la VA et les statuts sont mis à jour par les événements `CARD_UPDATED` autant que par les réponses, pour que tous les clients voient les mêmes valeurs.

## 6. Configuration

L'adresse IP et le port du serveur sont saisis dans l'écran de login. Les dernières valeurs saisies sont mémorisées dans un fichier de préférences local (`java.util.prefs`) pour éviter de les ressaisir.

## 7. Images des cartes

Des images peuvent illustrer les cartes. Elles seront choisies avec une licence qui autorise cet usage, et leurs crédits seront indiqués dans un fichier `CREDITS.md` du client (phase 2). Elles ne sont pas nécessaires au fonctionnement et ne transitent pas par le protocole : chaque client affiche une image locale associée au domaine de la carte.
