# Données du serveur

Responsables : Mai, Albine. Ce document définit les constantes, structures et variables globales du serveur C. Il complète [architecture-serveur.md](architecture-serveur.md).

## 1. Constantes

```c
#define MAX_CLIENTS        20     /* connexions simultanées */
#define MAX_ACTIVE_CARDS   10     /* cartes actives possédées par utilisateur */
#define MIN_BATTLE_VA       5     /* VA minimale pour entrer en battle */
#define MIN_LEGIT_RECOS     3     /* recommandations actives pour être légitimé */
#define LEGIT_VA_BONUS     10     /* bonus d'une carte authentifiée */
#define BATTLE_DURATION_S  60     /* durée du vote en secondes */
#define DIFFICULTY          3     /* zéros de tête du hash */
#define MAX_LINE_LEN     1024     /* taille maximale d'une ligne du protocole */
#define HASH_HEX_LEN       65     /* 64 caractères + '\0' */
#define MAX_USERNAME_LEN   30
#define MAX_NAME_LEN       50
#define MAX_CREATOR_LEN   100
#define MAX_DESC_LEN      500
```

## 2. Énumérations

```c
typedef enum { DOMAIN_WEB, DOMAIN_GAMEDEV, DOMAIN_DATASCIENCE,
               DOMAIN_SYSTEMS, DOMAIN_MOBILE } domain_t;

typedef enum { CARD_ACTIVE, CARD_EN_BATTLE,
               CARD_AUTHENTIFIEE, CARD_INACTIVE } card_status_t;

typedef enum { KIND_HUMAN, KIND_BOT } user_kind_t;

typedef enum { BATTLE_ONGOING, BATTLE_CLOSING, BATTLE_FINISHED } battle_status_t;

typedef enum { TIEBREAK_NONE, TIEBREAK_VA, TIEBREAK_ID } tiebreak_t;
```

`BATTLE_CLOSING` est un état interne : les votes sont fermés et le résultat est en cours de minage. Il n'apparaît pas dans le protocole (vu comme `ONGOING`).

## 3. Structures métier

```c
typedef struct {
    int           id;
    char          username[MAX_USERNAME_LEN + 1];
    user_kind_t   kind;
    int           is_legitimate;      /* 0 ou 1 */
} user_t;

typedef struct {
    int           id;                 /* généré par le serveur */
    char          nom[MAX_NAME_LEN + 1];
    char          createur_historique[MAX_CREATOR_LEN + 1];
    int           annee_creation;
    domain_t      domaine;
    char          description[MAX_DESC_LEN + 1];
    int           proprietaire_id;
    int           createur_id;
    int           valeur;             /* VA, dérivée, recalculée à chaque action */
    card_status_t statut;
    int           est_authentifiee;   /* 0 ou 1 */
    int           victoires;
} card_t;

typedef struct {
    int  user_id;
    int  card_id;                     /* carte qui porte la recommandation */
    int  origin_card_id;              /* carte d'origine (différente après un transfert) */
    int  active;                      /* 0 après répudiation ou perte de la carte */
} recommendation_t;

typedef struct {
    int             id;
    int             card_a, card_b;
    int             starter_user_id;
    int             va_a, va_b;       /* VA initiales, pour le départage */
    long            ends_at;
    battle_status_t statut;
    int             winner_card;      /* 0 tant que non terminée */
    int            *voters;           /* tableau dynamique des user_id ayant voté */
    int            *voted_card;       /* carte choisie par chaque votant */
    int             n_votes;
    int             votes_a, votes_b;
    pthread_t       timer_thread;
    pthread_cond_t  cancel_cond;      /* réveil du thread de battle à l'arrêt */
} battle_t;
```

Les identifiants sont attribués par le serveur par incrémentation de compteurs (`next_user_id`, `next_card_id`, `next_battle_id`), démarrés à 1. Un identifiant n'est jamais réutilisé. Après restauration, les compteurs repartent du plus grand identifiant lu dans les blocs, plus un.

## 4. Structures de la blockchain

Voir [structures-blockchain.md](structures-blockchain.md).

```c
typedef struct block {
    long          id;
    long          timestamp;
    char          action_type[24];
    char          data[1024];
    char          prev_hash[HASH_HEX_LEN];
    long          nonce;
    char          hash[HASH_HEX_LEN];
    struct block *next;               /* hors calcul du hash */
} block_t;

typedef struct {
    block_t        *head;
    block_t        *tail;
    long            length;
    long            last_persisted_id; /* dernier bloc sauvegardé en base, -1 si aucun */
    int             difficulty;
    pthread_mutex_t mutex;             /* chain_mutex */
} blockchain_t;
```

## 5. Structures de communication

```c
typedef struct {
    int         active;                /* emplacement occupé ? */
    int         sock;
    pthread_t   thread;
    int         user_id;               /* 0 tant que non identifié */
    user_kind_t kind;
    char        remote_ip[46];
} client_slot_t;

typedef struct {
    char  *fields[16];                 /* champs découpés sur '|' */
    int    n_fields;
    char   line[MAX_LINE_LEN + 1];
} request_t;
```

## 6. État global du serveur

Les variables globales sont regroupées dans une seule structure pour que leurs verrous et leur cycle de vie soient lisibles.

```c
typedef struct {
    /* Configuration */
    char             ip[46];
    int              port;
    int              difficulty;

    /* Réseau */
    int              listen_sock;
    client_slot_t    clients[MAX_CLIENTS];
    int              n_clients;
    pthread_mutex_t  clients_mutex;

    /* État de la plateforme (protégé par state_lock) */
    user_t          *users;      int n_users;     int cap_users;
    card_t          *cards;      int n_cards;     int cap_cards;
    recommendation_t *recos;     int n_recos;     int cap_recos;
    battle_t        *battles;    int n_battles;   int cap_battles;
    int              next_user_id, next_card_id, next_battle_id;
    pthread_rwlock_t state_lock;

    /* Blockchain et base */
    blockchain_t     chain;
    void            *pg_conn;    /* PGconn* */
    int              db_available;
    pthread_mutex_t  db_mutex;
} server_ctx_t;

extern server_ctx_t g_server;               /* unique instance */
extern volatile sig_atomic_t g_shutdown;    /* drapeau d'arrêt */
```

Les tableaux d'utilisateurs, de cartes, de recommandations et de battles sont des **tableaux dynamiques** (`realloc` par doublement de capacité) indexés par position. L'identifiant `id` d'une carte est égal à sa position + 1, ce qui donne un accès direct. Les cartes perdantes restent dans le tableau avec le statut `INACTIVE` : rien n'est supprimé, comme dans la blockchain.

## 7. Calcul de la VA

```c
int card_compute_va(const card_t *c)
{
    int n = count_active_recos(c->id);
    return n + LEGIT_VA_BONUS * c->est_authentifiee;
}
```

Cette fonction est la seule source de la VA. Elle est rappelée à chaque recommandation, répudiation, transfert et authentification. La formule est celle des [règles métier](regles-metier.md), section 2.3.

## 8. Résolution d'une battle

```c
/* Renvoie l'identifiant de la carte gagnante et renseigne le critère utilisé */
int battle_pick_winner(const battle_t *b, tiebreak_t *how)
{
    if (b->votes_a != b->votes_b) {
        *how = TIEBREAK_NONE;
        return b->votes_a > b->votes_b ? b->card_a : b->card_b;
    }
    if (b->va_a != b->va_b) {
        *how = TIEBREAK_VA;
        return b->va_a > b->va_b ? b->card_a : b->card_b;
    }
    *how = TIEBREAK_ID;
    return b->card_a < b->card_b ? b->card_a : b->card_b;
}
```

Transfert des recommandations : pour chaque recommandation active de la carte perdante, si son utilisateur n'a pas déjà de recommandation active sur la carte gagnante, la recommandation passe à la carte gagnante (`card_id` change, `origin_card_id` garde la carte d'origine). Sinon elle est désactivée (précision P2 des règles métier). La perdante passe à `INACTIVE`, la gagnante à `ACTIVE` ou `AUTHENTIFIEE` selon `est_authentifiee`.

## 9. Accès concurrent aux données

| Donnée | Lecture | Écriture |
| :--- | :--- | :--- |
| `users`, `cards`, `recos`, `battles` | `state_lock` en lecture | `state_lock` en écriture, uniquement dans `apply_block` |
| `chain` | `chain.mutex` | `chain.mutex`, uniquement dans le pipeline |
| `clients`, `n_clients` | `clients_mutex` | `clients_mutex` |
| `pg_conn` | `db_mutex` | `db_mutex` |

Règle d'or : **l'état de la plateforme ne change que dans `apply_block`**. C'est le même code en fonctionnement normal et à la restauration.

## 10. Mémoire

Chaque structure allouée dynamiquement a un propriétaire clair : le serveur alloue les tableaux d'état au démarrage et les libère à l'arrêt ; chaque thread libère sa propre `request_t` ; les tableaux de votes d'une battle sont libérés avec la battle. Le test de phase 2 inclut une exécution sous `valgrind` pour contrôler l'absence de fuite.
