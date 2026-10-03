# Annexe C — Diagrammes de classes

Cette annexe donne une vue d'ensemble des données du client (classes Java) et du serveur (structures et modules C). Les attributs détaillés sont dans [donnees-client.md](donnees-client.md), [donnees-serveur.md](donnees-serveur.md) et [structures-blockchain.md](structures-blockchain.md).

Le serveur est écrit en C : il n'a pas de classes au sens strict. Le diagramme serveur représente chaque `struct` comme une classe et chaque module comme une classe sans attribut qui regroupe ses fonctions. Ce choix sert uniquement à lire les dépendances.

## C.1 Classes du client Java (`diagrammes-client/02-diagramme-classes.png`)

```mermaid
classDiagram
    direction LR

    class Card {
        +int id
        +String nom
        +String createurHistorique
        +int anneeCreation
        +Domain domaine
        +String description
        +IntegerProperty proprietaireId
        +int createurId
        +IntegerProperty valeur
        +ObjectProperty~CardStatus~ statut
        +isAuthenticated() boolean
        +isRecommendable() boolean
        +fromRow(String[]) Card
    }
    class UserInfo {
        +int id
        +String username
        +UserKind kind
        +BooleanProperty legitimate
    }
    class Battle {
        +int id
        +int cardA
        +int cardB
        +IntegerProperty votesA
        +IntegerProperty votesB
        +long endsAt
        +ObjectProperty~BattleStatus~ status
        +int winnerCard
        +BooleanProperty hasVoted
        +remainingSeconds(long) int
        +canVote(UserInfo, ClientModel) boolean
    }
    class Recommendation {
        +int cardId
        +int userId
        +String username
    }
    class ClientModel {
        +ObjectProperty~UserInfo~ me
        +ObservableList~Card~ allCards
        +ObservableList~UserInfo~ onlineUsers
        +ObservableList~Battle~ battles
        +ObservableList~Recommendation~ receivedRecommendations
        +ObservableList~String~ eventLog
        +ownedCards() ObservableList
        +createdCards() ObservableList
        +applyEvent(ProtocolMessage)
        +reloadFromServer(ServerConnection)
    }
    class ProtocolMessage {
        +Type type
        +String command
        +List~String~ fields
        +List~List~String~~ rows
        +String errorCode
    }
    class ProtocolParser {
        +parse(String)$ ProtocolMessage
        +expectedRows(ProtocolMessage)$ int
        +buildRequest(String, Object...)$ String
    }
    class ServerConnection {
        -Socket socket
        -Thread readerThread
        -CompletableFuture pending
        +connect(String, int)
        +send(String) CompletableFuture
        +close()
    }
    class ServerEventListener {
        <<interface>>
        +onEvent(ProtocolMessage)
        +onDisconnected(String)
    }
    class SceneNavigator {
        +goTo(Screen)
    }
    class LoginController
    class MainController
    class CardsController
    class CreateCardController
    class BattlesController
    class ProfileController
    class BotClient {
        +run(BotScenario)
    }

    ClientModel "1" o-- "*" Card
    ClientModel "1" o-- "*" UserInfo
    ClientModel "1" o-- "*" Battle
    ClientModel "1" o-- "*" Recommendation
    ServerConnection ..> ProtocolParser : utilise
    ServerConnection ..> ProtocolMessage : produit
    ServerConnection --> ServerEventListener : notifie
    ClientModel ..|> ServerEventListener
    LoginController --> ServerConnection
    LoginController --> ClientModel
    MainController --> ClientModel
    CardsController --> ServerConnection
    CardsController --> ClientModel
    CreateCardController --> ServerConnection
    BattlesController --> ServerConnection
    BattlesController --> ClientModel
    ProfileController --> ServerConnection
    ProfileController --> ClientModel
    BotClient --> ServerConnection
    SceneNavigator ..> LoginController
    SceneNavigator ..> MainController
```

<!-- png: diagrammes-client/02-diagramme-classes.png -->

Points de conception :

* `ClientModel` est le seul état partagé. Les contrôleurs le lisent et le modifient, jamais l'un l'autre.
* `ServerConnection` ne connaît pas la vue : elle notifie un `ServerEventListener`, implémenté par `ClientModel`. Le bot réutilise la même connexion sans modèle graphique.
* `ProtocolParser` est statique et sans état : il se teste sans réseau.

## C.2 Structures et modules du serveur C (`diagrammes-serveur/02-classes-serveur.png`)

```mermaid
classDiagram
    direction TB

    class server_ctx_t {
        +char ip[46]
        +int port
        +int listen_sock
        +client_slot_t clients[20]
        +int n_clients
        +user_t* users
        +card_t* cards
        +recommendation_t* recos
        +battle_t* battles
        +blockchain_t chain
        +PGconn* pg_conn
        +pthread_mutex_t clients_mutex
        +pthread_rwlock_t state_lock
        +pthread_mutex_t db_mutex
    }
    class client_slot_t {
        +int active
        +int sock
        +pthread_t thread
        +int user_id
        +user_kind_t kind
    }
    class user_t {
        +int id
        +char username[31]
        +user_kind_t kind
        +int is_legitimate
    }
    class card_t {
        +int id
        +char nom[51]
        +domain_t domaine
        +int proprietaire_id
        +int createur_id
        +int valeur
        +card_status_t statut
        +int est_authentifiee
        +int victoires
    }
    class recommendation_t {
        +int user_id
        +int card_id
        +int origin_card_id
        +int active
    }
    class battle_t {
        +int id
        +int card_a
        +int card_b
        +int va_a
        +int va_b
        +long ends_at
        +battle_status_t statut
        +int winner_card
        +int votes_a
        +int votes_b
        +pthread_t timer_thread
    }
    class blockchain_t {
        +block_t* head
        +block_t* tail
        +long length
        +long last_persisted_id
        +int difficulty
        +pthread_mutex_t mutex
    }
    class block_t {
        +long id
        +long timestamp
        +char action_type[24]
        +char data[1024]
        +char prev_hash[65]
        +long nonce
        +char hash[65]
        +block_t* next
    }

    class server { +accept_loop() +free_slot() }
    class client_thread { +client_main() }
    class protocol { +parse_request() +reply_ok() +reply_err() +send_event() }
    class actions { +validate_*() +apply_*() }
    class pipeline { +run_action() }
    class battle { +battle_timer() +battle_pick_winner() }
    class block { +block_compute_hash() +block_mine() }
    class blockchain { +blockchain_append() +blockchain_verify() }
    class db { +db_save_block() +db_load_chain() +db_rebuild_state() }

    server_ctx_t "1" *-- "20" client_slot_t
    server_ctx_t "1" *-- "*" user_t
    server_ctx_t "1" *-- "*" card_t
    server_ctx_t "1" *-- "*" recommendation_t
    server_ctx_t "1" *-- "*" battle_t
    server_ctx_t "1" *-- "1" blockchain_t
    blockchain_t "1" o-- "*" block_t
    card_t "1" <-- "*" recommendation_t : card_id
    user_t "1" <-- "*" recommendation_t : user_id
    card_t "2" <-- "*" battle_t : card_a, card_b

    server ..> client_thread : crée
    client_thread ..> protocol
    client_thread ..> pipeline
    battle ..> pipeline
    pipeline ..> actions
    pipeline ..> block
    pipeline ..> blockchain
    pipeline ..> db
    actions ..> server_ctx_t
    blockchain ..> block
```

<!-- png: diagrammes-serveur/02-classes-serveur.png -->

Points de conception :

* `server_ctx_t` regroupe tout l'état global ; les verrous qui le protègent sont dans la même structure.
* `pipeline` est le seul module qui enchaîne validation, minage, ajout et sauvegarde. Les modules `actions`, `block`, `blockchain` et `db` ne connaissent pas cet enchaînement.
* `battle` appelle `pipeline` comme un client : le résultat d'une battle suit exactement le même chemin qu'une action utilisateur.

## C.3 Structure de la blockchain

Le diagramme de classes `00-blockchain.png` (blocs et chaîne) est dans [structures-blockchain.md](structures-blockchain.md) §1. Le schéma relationnel `01-schema-bdd.png` est dans [schema-bdd.md](schema-bdd.md) §2.
