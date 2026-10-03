# Annexe B — Cas d'utilisation

Cette annexe présente les cas d'utilisation de la plateforme. Ils reprennent les 27 cas du cahier des charges, regroupés par acteur. Les diagrammes sont écrits en Mermaid (les acteurs sont en rectangles arrondis, les cas en ellipses).

## B.1 Acteurs

| Acteur | Description |
| :--- | :--- |
| Utilisateur | Humain connecté avec le client JavaFX |
| Bot | Client automatique, `kind = BOT` |
| Administrateur | Personne qui lance et arrête le serveur depuis sa console |
| PostgreSQL | Système externe qui sauvegarde la blockchain |

## B.2 Cas d'utilisation global (`diagrammes-generaux/01-cas-utilisation-global.png`)

```mermaid
flowchart LR
    U([Utilisateur])
    BOT([Bot])
    ADM([Administrateur])
    DB([PostgreSQL])

    subgraph Plateforme
        UC1([Se connecter])
        UC2([Créer une carte])
        UC3([Recommander une carte])
        UC4([Répudier une recommandation])
        UC5([Lancer une battle])
        UC6([Voter dans une battle])
        UC7([Demander la légitimation])
        UC8([Revendiquer une carte])
        UC9([Authentifier une carte])
        UC10([Consulter cartes, battles, utilisateurs])
        UC11([Vérifier la blockchain])
        UC12([Se déconnecter])
        UC13([Démarrer le serveur])
        UC14([Arrêter proprement le serveur])
        UC15([Restaurer le contexte])
        UC16([Appliquer le résultat d'une battle])
    end

    U --- UC1 & UC2 & UC3 & UC4 & UC5 & UC6 & UC7 & UC8 & UC9 & UC10 & UC11 & UC12
    BOT --- UC1 & UC2 & UC6 & UC12
    ADM --- UC13 & UC14 & UC11
    UC13 -.->|inclut| UC15
    UC6 -.->|déclenche| UC16
    UC15 --- DB
    UC2 --- DB
```

<!-- png: diagrammes-generaux/01-cas-utilisation-global.png -->

## B.3 Cas d'utilisation du client (`diagrammes-client/01-diagramme-cas-utilisation.png`)

```mermaid
flowchart LR
    U([Utilisateur])
    BOT([Bot])

    subgraph Client["Client Java"]
        C1([Configurer IP et port])
        C2([Se connecter])
        C3([Créer une carte])
        C4([Consulter mes cartes])
        C5([Recommander / répudier])
        C6([Lancer une battle])
        C7([Voter])
        C8([Demander la légitimation])
        C9([Revendiquer une carte])
        C10([Authentifier une carte])
        C11([Voir les utilisateurs connectés])
        C12([Voir les battles])
        C13([Vérifier la blockchain])
        C14([Se déconnecter])
    end

    U --- C1 & C2 & C3 & C4 & C5 & C6 & C7 & C8 & C9 & C10 & C11 & C12 & C13 & C14
    BOT --- C1 & C2 & C3 & C7 & C14
    C2 -.->|inclut| C1
```

<!-- png: diagrammes-client/01-diagramme-cas-utilisation.png -->

## B.4 Cas d'utilisation du serveur (`diagrammes-serveur/00-cas-utilisation-serveur.png`)

```mermaid
flowchart LR
    CL([Client])
    ADM([Administrateur])
    DB([PostgreSQL])

    subgraph Serveur
        S1([Démarrer la plateforme])
        S2([Initialiser ou charger la blockchain])
        S3([Communiquer par sockets])
        S4([Limiter les connexions à 20])
        S5([Reconstituer le contexte d'un client])
        S6([Traiter une demande d'action])
        S7([Miner et ajouter un bloc])
        S8([Sauvegarder en base])
        S9([Appliquer le résultat d'une battle])
        S10([Rendre une carte inactive])
        S11([Vérifier la cohérence])
        S12([Gérer une déconnexion])
        S13([Arrêter proprement])
    end

    ADM --- S1 & S11 & S13
    CL --- S3 & S6 & S11 & S12
    S1 -.->|inclut| S2
    S3 -.->|inclut| S4
    S3 -.->|inclut| S5
    S6 -.->|inclut| S7
    S7 -.->|inclut| S8
    S9 -.->|inclut| S10
    S2 --- DB
    S8 --- DB
```

<!-- png: diagrammes-serveur/00-cas-utilisation-serveur.png -->

## B.5 Correspondance avec le cahier des charges

| N° | Cas du cahier des charges | Où il est traité |
| :--- | :--- | :--- |
| 1 | Démarrer la plateforme | Serveur : démarrage, [architecture-serveur.md](architecture-serveur.md) §7 |
| 2 | Initialiser ou charger la blockchain | [architecture-serveur.md](architecture-serveur.md) §7, [schema-bdd.md](schema-bdd.md) §5 |
| 3 | Communiquer avec les clients | [protocole-applicatif-commun.md](protocole-applicatif-commun.md) |
| 4 | Limiter les connexions simultanées | `MAX_CLIENTS = 20`, [donnees-serveur.md](donnees-serveur.md) |
| 5 | Configurer la connexion d'un client | Écran de login, [architecture-client.md](architecture-client.md) |
| 6 | Types de clients | Humain et bot, [architecture-client.md](architecture-client.md) §9 |
| 7 | Reconstituer le contexte d'un utilisateur | `LOGIN` puis `LIST_*`, [protocole](protocole-applicatif-commun.md) §4.1 |
| 8 | Traiter une demande d'action | Pipeline, [architecture-serveur.md](architecture-serveur.md) §5 |
| 9 | Traitement concurrent | Un thread par client, minage sans verrou |
| 10 | Actions minimales | [protocole](protocole-applicatif-commun.md) §4 |
| 11 | Créer une carte | `CREATE_CARD`, [règles métier](regles-metier.md) §1 |
| 12 | Recommander une carte | `RECOMMEND`, [règles métier](regles-metier.md) §2 |
| 13 | Consulter recommandations | `LIST_CARDS\|RECOMMENDED`, `LIST_RECEIVED` |
| 14 | Répudier | `REPUDIATE` |
| 15 | Lancer une battle | `START_BATTLE`, [règles métier](regles-metier.md) §3 |
| 16 | Voter | `VOTE` |
| 17 | Appliquer le résultat | `ACTION_BATTLE_RESULT`, [règles métier](regles-metier.md) §3.5 |
| 18 | Demander une légitimation | `REQUEST_LEGITIMATION`, [règles métier](regles-metier.md) §4.1 |
| 19 | Revendiquer | `CLAIM_CARD`, [règles métier](regles-metier.md) §4.2 |
| 20 | Authentifier | `AUTHENTICATE_CARD`, [règles métier](regles-metier.md) §4.3 |
| 21 | Rendre une carte inactive | Perdante d'une battle, [règles métier](regles-metier.md) §3.5 |
| 22 | Restaurer le contexte | Rejeu des blocs, [structures-blockchain.md](structures-blockchain.md) §7 |
| 23 | Interface graphique | [architecture-client.md](architecture-client.md) §8 |
| 24 | Client automatique | Bot, [architecture-client.md](architecture-client.md) §9 |
| 25 | Déconnexion d'un client | [architecture-serveur.md](architecture-serveur.md) §9 |
| 26 | Vérifier la cohérence | `VERIFY_CHAIN`, [structures-blockchain.md](structures-blockchain.md) §5 |
| 27 | Arrêter proprement le serveur | [architecture-serveur.md](architecture-serveur.md) §8 |

Aucun cas d'utilisation supplémentaire n'est ajouté à ceux du cahier des charges. Le partage ciblé, facultatif, n'est pas retenu pour cette version.
