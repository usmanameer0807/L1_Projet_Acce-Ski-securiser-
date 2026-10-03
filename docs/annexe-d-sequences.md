# Annexe D — Diagrammes de séquence

Cette annexe décrit les échanges entre composants pour chaque action. La partie D.1 présente la vue côté client (utilisateur, interface, réseau, serveur) et la partie D.2 la vue côté serveur (threads, validation, blockchain, base). Les deux se complètent : le serveur `S` de la partie D.1 correspond au thread client `CT` de la partie D.2.

Le minage et la vérification, au niveau de l'algorithme, sont dans [structures-blockchain.md](structures-blockchain.md) (diagrammes `02-sequence-minage` et `03-sequence-verification` de `diagrammes-blockchain-bdd/`).

Convention : les noms de commandes et de réponses sont ceux du [protocole](protocole-applicatif-commun.md).

---

## D.1 Séquences côté client

Participants : **U** utilisateur, **V** vue JavaFX, **C** contrôleur, **N** couche réseau (`ServerConnection`), **S** serveur.

### D.1.1 Login (`diagrammes-client/03-sequence-login.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Login
    participant C as LoginController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: saisit IP, port, nom
    V->>C: clic « Se connecter »
    C->>C: contrôle les champs
    C->>N: connect(ip, port)
    N->>S: connexion TCP
    C->>N: send("LOGIN|alice|HUMAN")
    N->>S: LOGIN|alice|HUMAN
    alt serveur plein ou nom déjà pris
        S-->>N: ERR|LOGIN|SERVER_FULL / USERNAME_TAKEN
        N-->>C: erreur
        C-->>V: message lisible
    else accepté
        S-->>N: OK|LOGIN|1|false|0
        C->>N: GET_PROFILE, LIST_CARDS, LIST_RECEIVED, LIST_USERS, LIST_BATTLES
        N->>S: requêtes de consultation
        S-->>N: réponses OK et lignes ROW
        N-->>C: modèle mis à jour
        C-->>V: affiche l'écran principal
    end
```

<!-- png: diagrammes-client/03-sequence-login.png -->

### D.1.2 Création de carte (`diagrammes-client/04-sequence-create-card.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Création
    participant C as CreateCardController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: remplit le formulaire
    V->>C: clic « Créer »
    C->>C: contrôle longueurs, année, domaine
    C->>N: send("CREATE_CARD|Python|Guido van Rossum|1991|DataScience|…")
    N->>S: CREATE_CARD|…
    alt refusée
        S-->>N: ERR|CREATE_CARD|CARD_LIMIT_REACHED
        N-->>C: erreur
        C-->>V: message lisible
    else créée
        S-->>N: OK|CREATE_CARD|1|2
        S-->>N: EVT|CARD_CREATED|1|Python|DataScience|1
        N-->>C: carte ajoutée au modèle
        C-->>V: retour à la liste des cartes
    end
```

<!-- png: diagrammes-client/04-sequence-create-card.png -->

### D.1.3 Recommandation (`diagrammes-client/05-sequence-recommend.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Cartes
    participant C as CardsController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: sélectionne une carte
    V->>C: clic « Recommander »
    C->>N: send("RECOMMEND|3")
    N->>S: RECOMMEND|3
    alt refusée
        S-->>N: ERR|RECOMMEND|ALREADY_RECOMMENDED / CARD_NOT_RECOMMENDABLE
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|RECOMMEND|3|13|9
        S-->>N: EVT|CARD_UPDATED|3|ACTIVE|13|2
        N-->>C: VA mise à jour
        C-->>V: liste « recommandées » rafraîchie
    end
```

<!-- png: diagrammes-client/05-sequence-recommend.png -->

### D.1.4 Répudiation (`diagrammes-client/06-sequence-repudiate.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Cartes
    participant C as CardsController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: sélectionne une carte recommandée
    V->>C: clic « Répudier »
    C->>N: send("REPUDIATE|3")
    N->>S: REPUDIATE|3
    alt aucune recommandation active
        S-->>N: ERR|REPUDIATE|NO_ACTIVE_RECOMMENDATION
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|REPUDIATE|3|12|10
        S-->>N: EVT|CARD_UPDATED|3|ACTIVE|12|2
        N-->>C: VA mise à jour
        C-->>V: carte retirée de la liste « recommandées »
    end
```

<!-- png: diagrammes-client/06-sequence-repudiate.png -->

### D.1.5 Lancement d'une battle (`diagrammes-client/07-sequence-battle.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Cartes
    participant C as CardsController
    participant N as ServerConnection
    participant S as Serveur
    participant O as Autres clients
    U->>V: choisit deux cartes
    V->>C: clic « Lancer une battle »
    C->>N: send("START_BATTLE|1|2")
    N->>S: START_BATTLE|1|2
    alt refusée
        S-->>N: ERR|START_BATTLE|CARDS_NOT_COMPATIBLE / VA_TOO_LOW / NOT_CARD_OWNER / CARD_NOT_AVAILABLE
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|START_BATTLE|2|1790000060|14
        S-->>N: EVT|BATTLE_STARTED|2|1|2|1790000060
        S-->>O: EVT|BATTLE_STARTED|2|1|2|1790000060
        N-->>C: battle ajoutée au modèle
        C-->>V: compte à rebours de 60 s
    end
    Note over S,O: 60 s plus tard : EVT|BATTLE_ENDED (voir D.1.6)
```

<!-- png: diagrammes-client/07-sequence-battle.png -->

### D.1.6 Vote (`diagrammes-client/08-sequence-vote.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Battles
    participant C as BattlesController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: clic « Voter Python »
    V->>C: vote(battle 2, carte 1)
    C->>N: send("VOTE|2|1")
    N->>S: VOTE|2|1
    alt refusé
        S-->>N: ERR|VOTE|ALREADY_VOTED / VOTE_FORBIDDEN_OWNER / BATTLE_CLOSED
        N-->>C: erreur
        C-->>V: message lisible
    else accepté
        S-->>N: OK|VOTE|2|15
        S-->>N: EVT|VOTE_CAST|2|3|1
        N-->>C: compteurs mis à jour, hasVoted = true
        C-->>V: boutons de vote désactivés
    end
    S-->>N: EVT|BATTLE_ENDED|2|1|2|3|1|NONE
    N-->>C: battle terminée
    C-->>V: affiche le vainqueur
```

<!-- png: diagrammes-client/08-sequence-vote.png -->

### D.1.7 Légitimation (`diagrammes-client/09-sequence-legitimation.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Profil
    participant C as ProfileController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: clic « Demander la légitimation »
    V->>C: demande pour la carte choisie
    C->>N: send("REQUEST_LEGITIMATION|3")
    N->>S: REQUEST_LEGITIMATION|3
    alt moins de 3 recommandations actives ou déjà légitimé
        S-->>N: ERR|REQUEST_LEGITIMATION|NOT_ENOUGH_RECOMMENDATIONS / ALREADY_LEGITIMATE
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|REQUEST_LEGITIMATION|16
        N-->>C: me.legitimate = true
        C-->>V: badge « légitimé », boutons Revendiquer et Authentifier actifs
    end
```

<!-- png: diagrammes-client/09-sequence-legitimation.png -->

### D.1.8 Authentification d'une carte (`diagrammes-client/10-sequence-authentification.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue Cartes
    participant C as CardsController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: sélectionne sa carte
    V->>C: clic « Authentifier »
    C->>N: send("AUTHENTICATE_CARD|3")
    N->>S: AUTHENTICATE_CARD|3
    alt refusée
        S-->>N: ERR|AUTHENTICATE_CARD|NOT_LEGITIMATE / NOT_CARD_OWNER / ALREADY_AUTHENTICATED
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|AUTHENTICATE_CARD|3|22|17
        S-->>N: EVT|CARD_UPDATED|3|AUTHENTIFIEE|22|2
        N-->>C: statut et VA mis à jour
        C-->>V: badge « authentifiée » affiché
    end
```

<!-- png: diagrammes-client/10-sequence-authentification.png -->

### D.1.9 Revendication (`diagrammes-client/11-sequence-revendication.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur légitimé
    participant V as Vue Cartes
    participant C as CardsController
    participant N as ServerConnection
    participant S as Serveur
    participant P as Ancien propriétaire
    U->>V: sélectionne une carte d'un tiers
    V->>C: clic « Revendiquer »
    C->>N: send("CLAIM_CARD|4")
    N->>S: CLAIM_CARD|4
    alt refusée
        S-->>N: ERR|CLAIM_CARD|NOT_LEGITIMATE / OWNER_IS_LEGITIMATE / CARD_LIMIT_REACHED
        N-->>C: erreur
        C-->>V: message lisible
    else acceptée
        S-->>N: OK|CLAIM_CARD|4|18
        S-->>N: EVT|CARD_UPDATED|4|ACTIVE|9|1
        S-->>P: EVT|CARD_UPDATED|4|ACTIVE|9|1
        N-->>C: propriétaire mis à jour
        C-->>V: la carte apparaît dans « Possédées »
    end
```

<!-- png: diagrammes-client/11-sequence-revendication.png -->

### D.1.10 Déconnexion (`diagrammes-client/12-sequence-disconnect.png`)

```mermaid
sequenceDiagram
    actor U as Utilisateur
    participant V as Vue principale
    participant C as MainController
    participant N as ServerConnection
    participant S as Serveur
    U->>V: clic « Déconnexion »
    V->>C: logout()
    C->>N: send("DISCONNECT")
    N->>S: DISCONNECT
    S-->>N: OK|DISCONNECT
    S->>S: ferme le socket, libère l'emplacement
    N->>N: close() et arrêt du thread de lecture
    C->>C: vide le modèle
    C-->>V: retour à l'écran Login
    Note over S: les autres clients reçoivent EVT|USER_DISCONNECTED
```

<!-- png: diagrammes-client/12-sequence-disconnect.png -->

---

## D.2 Séquences côté serveur

Participants : **CL** client, **CT** thread client, **P** module `protocol`, **A** module `actions`, **PL** module `pipeline`, **BC** blockchain (`block` et `blockchain`), **ST** état en mémoire, **DB** PostgreSQL.

Les séquences D.2.2 à D.2.9 utilisent toutes le pipeline commun. Pour éviter de le répéter, D.2.2 le détaille en entier et les suivantes montrent ce qui change.

### D.2.1 Login (`diagrammes-serveur/03-sequence-login-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant M as Thread principal
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    CL->>M: connexion TCP
    alt 20 clients déjà connectés
        M-->>CL: ERR|LOGIN|SERVER_FULL
        M->>M: ferme le socket
    else place disponible
        M->>CT: crée le thread (emplacement réservé)
        CL->>CT: LOGIN|alice|HUMAN
        CT->>ST: lecture : nom connecté ?
        alt nom déjà connecté
            CT-->>CL: ERR|LOGIN|USERNAME_TAKEN
        else nom libre
            alt utilisateur inconnu
                CT->>PL: ACTION_REGISTER_USER
                PL-->>CT: bloc ajouté
            end
            CT->>ST: reconstitue le contexte de l'utilisateur
            CT-->>CL: OK|LOGIN|user_id|is_legitimate|nb_cartes
            CT-->>CL: (les autres clients reçoivent EVT|USER_CONNECTED)
        end
    end
```

<!-- png: diagrammes-serveur/03-sequence-login-serveur.png -->

### D.2.2 Création de carte, avec le pipeline complet (`diagrammes-serveur/04-sequence-create-card-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant P as protocol
    participant A as actions
    participant PL as pipeline
    participant BC as Blockchain
    participant ST as État
    participant DB as PostgreSQL
    CL->>CT: CREATE_CARD|Python|…
    CT->>P: parse_request(ligne)
    P-->>CT: champs contrôlés
    CT->>PL: run_action(CREATE_CARD, champs)
    PL->>A: validate_create_card()
    A->>ST: lecture sous state_lock (rdlock)
    alt refus (limite de 10 cartes, champ invalide)
        A-->>PL: erreur
        PL-->>CT: ERR
        CT-->>CL: ERR|CREATE_CARD|CARD_LIMIT_REACHED
    else valide
        PL->>BC: lire id et hash du dernier bloc (chain_mutex)
        PL->>BC: block_mine() sans verrou
        loop tant que le dernier bloc a changé
            PL->>BC: chain_mutex : dernier bloc inchangé ?
            alt changé
                PL->>BC: block_mine() avec le nouveau prev_hash
            end
        end
        PL->>A: revalider (state_lock en écriture)
        alt devenue invalide
            PL-->>CT: ERR (bloc abandonné)
        else toujours valide
            PL->>BC: ajouter le bloc
            PL->>ST: apply_block() : crée la carte, VA 0
            PL->>DB: BEGIN, INSERT blocks, INSERT cards, COMMIT
            PL-->>CT: succès (card_id, block_id)
            CT-->>CL: OK|CREATE_CARD|card_id|block_id
            CT-->>CL: EVT|CARD_CREATED (diffusé aux autres clients)
        end
    end
```

<!-- png: diagrammes-serveur/04-sequence-create-card-serveur.png -->

### D.2.3 Recommandation (`diagrammes-serveur/05-sequence-recommend-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    CL->>CT: RECOMMEND|3
    CT->>PL: run_action(RECOMMEND, 3)
    PL->>ST: carte existe ? ACTIVE ou AUTHENTIFIEE ? déjà recommandée ?
    alt refus
        PL-->>CL: ERR|RECOMMEND|CARD_NOT_FOUND / CARD_NOT_RECOMMENDABLE / ALREADY_RECOMMENDED
    else valide
        PL->>BC: miner ACTION_RECOMMEND (user_id|card_id)
        PL->>ST: revalider, ajouter le bloc, recommandation active, VA + 1
        PL->>DB: transaction : INSERT blocks, INSERT recommendations, UPDATE cards
        PL-->>CL: OK|RECOMMEND|3|nouvelle_va|block_id
        PL-->>CL: EVT|CARD_UPDATED (tous les clients)
    end
```

<!-- png: diagrammes-serveur/05-sequence-recommend-serveur.png -->

### D.2.4 Répudiation (`diagrammes-serveur/06-sequence-repudiate-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    CL->>CT: REPUDIATE|3
    CT->>PL: run_action(REPUDIATE, 3)
    PL->>ST: recommandation active de cet utilisateur sur la carte ?
    alt aucune
        PL-->>CL: ERR|REPUDIATE|NO_ACTIVE_RECOMMENDATION
    else existe
        PL->>BC: miner ACTION_REPUDIATE (user_id|card_id)
        PL->>ST: revalider, ajouter le bloc, recommandation inactive, VA − 1
        PL->>DB: transaction : INSERT blocks, UPDATE recommendations (revoked_block), UPDATE cards
        PL-->>CL: OK|REPUDIATE|3|nouvelle_va|block_id
        PL-->>CL: EVT|CARD_UPDATED (tous les clients)
    end
    Note over BC: le bloc ACTION_RECOMMEND d'origine reste intact dans la chaîne
```

<!-- png: diagrammes-serveur/06-sequence-repudiate-serveur.png -->

### D.2.5 Lancement d'une battle (`diagrammes-serveur/07-sequence-battle-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    participant BT as Thread de battle
    CL->>CT: START_BATTLE|1|2
    CT->>PL: run_action(START_BATTLE, 1, 2)
    PL->>ST: cartes distinctes, même domaine, statuts ACTIVE ou AUTHENTIFIEE,<br/>demandeur propriétaire d'une des deux, VA ≥ 5
    alt refus
        PL-->>CL: ERR|START_BATTLE|CARDS_NOT_COMPATIBLE / CARD_NOT_AVAILABLE / NOT_CARD_OWNER / VA_TOO_LOW
    else valide
        PL->>BC: miner ACTION_BATTLE_START (… va_a|va_b|ends_at)
        PL->>ST: revalider, ajouter le bloc, cartes EN_BATTLE, battle ONGOING
        PL->>DB: transaction : INSERT blocks, INSERT battles, UPDATE cards
        PL->>BT: crée le thread (échéance = maintenant + 60 s)
        PL-->>CL: OK|START_BATTLE|battle_id|ends_at|block_id
        PL-->>CL: EVT|BATTLE_STARTED (tous les clients)
    end
```

<!-- png: diagrammes-serveur/07-sequence-battle-serveur.png -->

### D.2.6 Vote (`diagrammes-serveur/08-sequence-vote-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    CL->>CT: VOTE|2|1
    CT->>PL: run_action(VOTE, 2, 1)
    PL->>ST: battle existe et ouverte ? carte engagée ?<br/>votant non propriétaire ? pas déjà voté ?
    alt refus
        PL-->>CL: ERR|VOTE|BATTLE_NOT_FOUND / BATTLE_CLOSED / CARD_NOT_IN_BATTLE / VOTE_FORBIDDEN_OWNER / ALREADY_VOTED
    else valide
        PL->>BC: miner ACTION_BATTLE_VOTE (user_id|battle_id|card_id)
        PL->>ST: revalider (battle encore ouverte ?), ajouter le bloc, compteur + 1
        alt la battle s'est fermée pendant le minage
            PL-->>CL: ERR|VOTE|BATTLE_CLOSED (bloc abandonné)
        else
            PL->>DB: transaction : INSERT blocks, INSERT battle_votes
            PL-->>CL: OK|VOTE|2|block_id
            PL-->>CL: EVT|VOTE_CAST|2|votes_a|votes_b (tous les clients)
        end
    end
```

<!-- png: diagrammes-serveur/08-sequence-vote-serveur.png -->

### D.2.7 Résultat d'une battle (`diagrammes-serveur/09-sequence-battle-result-serveur.png`)

```mermaid
sequenceDiagram
    participant BT as Thread de battle
    participant ST as État
    participant PL as pipeline
    participant BC as Blockchain
    participant DB as PostgreSQL
    participant CL as Tous les clients
    BT->>BT: attend 60 s (pthread_cond_timedwait)
    BT->>ST: ferme les votes (statut CLOSING), lit les compteurs
    BT->>BT: battle_pick_winner() : votes, puis VA initiale, puis id
    BT->>PL: run_action(BATTLE_RESULT, gagnante, perdante, votes, critère)
    PL->>BC: miner ACTION_BATTLE_RESULT
    PL->>ST: ajouter le bloc, apply_block()
    Note over ST: recommandations actives de la perdante<br/>transférées à la gagnante (doublons fusionnés),<br/>gagnante : victoires + 1, retour ACTIVE ou AUTHENTIFIEE,<br/>perdante : INACTIVE
    PL->>DB: transaction : INSERT blocks, UPDATE battles, UPDATE cards, UPDATE recommendations
    PL-->>CL: EVT|BATTLE_ENDED|battle_id|gagnante|perdante|votes_a|votes_b|critère
    PL-->>CL: EVT|CARD_UPDATED (les deux cartes)
    Note over BC: les blocs antérieurs de la carte perdante restent dans la chaîne
```

<!-- png: diagrammes-serveur/09-sequence-battle-result-serveur.png -->

### D.2.8 Légitimation (`diagrammes-serveur/10-sequence-legitimation-serveur.png`)

La revendication (`CLAIM_CARD`) suit le même enchaînement, avec ses propres validations (voir [cas-erreur.md](cas-erreur.md) §3.3) et un bloc `ACTION_CLAIM_CARD`.

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    CL->>CT: REQUEST_LEGITIMATION|3
    CT->>PL: run_action(LEGITIMATION, 3)
    PL->>ST: déjà légitimé ? au moins 3 recommandations actives ?
    alt refus
        PL-->>CL: ERR|REQUEST_LEGITIMATION|ALREADY_LEGITIMATE / NOT_ENOUGH_RECOMMENDATIONS
    else valide
        PL->>BC: miner ACTION_LEGITIMATION (user_id|card_id)
        PL->>ST: revalider, ajouter le bloc, is_legitimate = 1
        PL->>DB: transaction : INSERT blocks, UPDATE users
        PL-->>CL: OK|REQUEST_LEGITIMATION|block_id
    end
```

<!-- png: diagrammes-serveur/10-sequence-legitimation-serveur.png -->

### D.2.9 Authentification (`diagrammes-serveur/11-sequence-authentification-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant PL as pipeline
    participant ST as État
    participant BC as Blockchain
    participant DB as PostgreSQL
    CL->>CT: AUTHENTICATE_CARD|3
    CT->>PL: run_action(AUTHENTICATE_CARD, 3)
    PL->>ST: utilisateur légitimé ? propriétaire ? déjà authentifiée ?<br/>carte ni INACTIVE ni EN_BATTLE ?
    alt refus
        PL-->>CL: ERR|AUTHENTICATE_CARD|NOT_LEGITIMATE / NOT_CARD_OWNER / ALREADY_AUTHENTICATED / CARD_NOT_AVAILABLE
    else valide
        PL->>BC: miner ACTION_AUTHENTICATE_CARD (user_id|card_id)
        PL->>ST: revalider, ajouter le bloc, statut AUTHENTIFIEE, VA + 10, caractéristiques verrouillées
        PL->>DB: transaction : INSERT blocks, UPDATE cards
        PL-->>CL: OK|AUTHENTICATE_CARD|3|nouvelle_va|block_id
        PL-->>CL: EVT|CARD_UPDATED (tous les clients)
    end
```

<!-- png: diagrammes-serveur/11-sequence-authentification-serveur.png -->

### D.2.10 Déconnexion (`diagrammes-serveur/12-sequence-disconnect-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client
    participant CT as Thread client
    participant SL as clients (emplacements)
    participant O as Autres clients
    alt départ propre
        CL->>CT: DISCONNECT
        CT-->>CL: OK|DISCONNECT
    else coupure réseau
        CL--xCT: socket fermé (read renvoie 0 ou une erreur)
        CT->>CT: abandonne la requête en cours<br/>(aucun bloc ajouté)
    end
    CT->>SL: clients_mutex : libère l'emplacement
    CT->>CT: close(socket)
    CT-->>O: EVT|USER_DISCONNECTED
    CT->>CT: le thread se termine
    Note over CT: une battle en cours continue jusqu'à l'échéance
```

<!-- png: diagrammes-serveur/12-sequence-disconnect-serveur.png -->

### D.2.11 Minage dans le serveur (`diagrammes-serveur/13-sequence-minage-serveur.png`)

Cette séquence détaille la partie « minage optimiste » du pipeline : elle montre pourquoi le calcul ne bloque pas les autres clients.

```mermaid
sequenceDiagram
    participant T1 as Thread client 1
    participant T2 as Thread client 2
    participant BC as Blockchain (chain_mutex)
    T1->>BC: lire dernier bloc (id 7, hash H7)
    T2->>BC: lire dernier bloc (id 7, hash H7)
    par minage sans verrou
        T1->>T1: miner le bloc 8 avec prev = H7
    and
        T2->>T2: miner le bloc 8 avec prev = H7
    end
    T1->>BC: dernier bloc toujours H7 ?
    BC-->>T1: oui : bloc 8 ajouté (hash H8)
    T2->>BC: dernier bloc toujours H7 ?
    BC-->>T2: non (maintenant H8)
    T2->>T2: remine le bloc 9 avec prev = H8
    T2->>BC: dernier bloc toujours H8 ?
    BC-->>T2: oui : bloc 9 ajouté
```

<!-- png: diagrammes-serveur/13-sequence-minage-serveur.png -->

### D.2.12 Vérification dans le serveur (`diagrammes-serveur/14-sequence-verification-serveur.png`)

```mermaid
sequenceDiagram
    participant CL as Client ou console
    participant CT as Thread
    participant BC as Blockchain
    participant V as blockchain_verify
    CL->>CT: VERIFY_CHAIN (ou « verify » sur la console)
    CT->>BC: chain_mutex : fige la chaîne pendant la lecture
    CT->>V: blockchain_verify()
    loop pour chaque bloc
        V->>V: block_compute_hash() et comparaison
        V->>V: prev_hash égal au hash précédent ?
        V->>V: hash commence par 000 ?
    end
    alt anomalie
        V-->>CT: INVALID(block_id, BAD_HASH / BAD_PREV_HASH / BAD_POW)
        CT-->>CL: OK|VERIFY_CHAIN|INVALID|block_id|raison
    else chaîne intègre
        V-->>CT: VALID(nb_blocs)
        CT-->>CL: OK|VERIFY_CHAIN|VALID|nb_blocs
    end
```

<!-- png: diagrammes-serveur/14-sequence-verification-serveur.png -->

> Pendant la vérification, `chain_mutex` est tenu : les ajouts de blocs attendent. La vérification d'une chaîne de quelques milliers de blocs dure quelques millisecondes (un SHA-256 par bloc), donc cette pause reste négligeable.
