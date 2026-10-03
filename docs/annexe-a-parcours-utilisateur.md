# Annexe A — Parcours utilisateur

Cette annexe montre comment un utilisateur traverse l'application et comment l'interface (GUI) échange avec le modèle et le serveur. Elle sert de lien entre les [maquettes](architecture-client.md) et le [protocole](protocole-applicatif-commun.md).

## A.1 Navigation générale (`diagrammes-generaux/02-navigation-javafx.png`)

```mermaid
flowchart TD
    START([Lancement du client]) --> L[Écran Login<br/>IP, port, nom]
    L -->|LOGIN OK| M[Écran principal]
    L -->|ERR SERVER_FULL / USERNAME_TAKEN| L
    M --> C[Cartes]
    M --> CC[Création de carte]
    M --> B[Battles]
    M --> P[Profil]
    C -->|Lancer une battle| B
    CC -->|carte créée| C
    C --> M
    B --> M
    P --> M
    M -->|DISCONNECT| L
    M -->|EVT SERVER_SHUTDOWN| L
```

<!-- png: diagrammes-generaux/02-navigation-javafx.png -->

## A.2 Parcours principal (`diagrammes-generaux/03-parcours-utilisateur.png`)

Ce schéma suit un utilisateur de la connexion à la déconnexion. Les losanges sont des décisions de l'utilisateur ou du serveur ; les actions du serveur sont en gras.

```mermaid
flowchart TD
    A([Démarrer le client]) --> B[Saisir IP, port et nom]
    B --> C[Envoyer LOGIN]
    C --> D{Serveur<br/>accepte ?}
    D -- non --> B
    D -- oui --> E[Le serveur reconstitue le contexte<br/>depuis la blockchain]
    E --> F[Le client charge cartes,<br/>utilisateurs, battles]
    F --> G{Que faire ?}

    G -- Créer --> H[Remplir le formulaire]
    H --> H2[CREATE_CARD]
    H2 --> H3{Valide ?}
    H3 -- oui --> G
    H3 -- non --> H4[Message d'erreur] --> G

    G -- Recommander --> I[Choisir une carte]
    I --> I2[RECOMMEND]
    I2 --> G

    G -- Répudier --> J[Choisir une carte recommandée]
    J --> J2[REPUDIATE]
    J2 --> G

    G -- Battle --> K[Choisir deux cartes<br/>du même domaine]
    K --> K2[START_BATTLE]
    K2 --> K3{Acceptée ?}
    K3 -- non --> K4[Message d'erreur] --> G
    K3 -- oui --> K5[60 s de vote<br/>VOTE par les autres utilisateurs]
    K5 --> K6[Le serveur applique le résultat]
    K6 --> G

    G -- Voter --> V[Choisir une carte<br/>d'une battle en cours]
    V --> V2[VOTE]
    V2 --> G

    G -- Légitimation --> L1[REQUEST_LEGITIMATION]
    L1 --> L2{3 recommandations<br/>actives ?}
    L2 -- oui --> L3[Utilisateur légitimé] --> G
    L2 -- non --> L4[Message d'erreur] --> G

    G -- Revendiquer --> R1[CLAIM_CARD sur une carte<br/>d'un propriétaire non légitimé]
    R1 --> G

    G -- Authentifier --> U1[AUTHENTICATE_CARD<br/>sur une de mes cartes]
    U1 --> G

    G -- Quitter --> Z[DISCONNECT]
    Z --> Y([Fin])
```

<!-- png: diagrammes-generaux/03-parcours-utilisateur.png -->

## A.3 Échange entre GUI, modèle et serveur

Pour toute action, l'interface suit le même schéma :

1. L'utilisateur clique sur un bouton. Le contrôleur contrôle les champs.
2. Le contrôleur demande à la couche réseau d'envoyer la requête et affiche un indicateur d'attente.
3. Le serveur valide, mine, enregistre, puis répond.
4. À la réponse, le contrôleur met à jour le modèle ; les vues liées au modèle se rafraîchissent.
5. Les autres clients reçoivent l'événement `EVT` correspondant et mettent à jour leur modèle.

Le détail seconde par seconde de chaque action est dans les [diagrammes de séquence](annexe-d-sequences.md).

## A.4 Parcours de deux utilisateurs pendant une battle

| Temps | Alice (propriétaire de Python) | Bob (propriétaire de R) | Carol (tiers) | Serveur |
| :--- | :--- | :--- | :--- | :--- |
| 0 s | `START_BATTLE\|1\|2` | — | — | Valide (même domaine, VA ≥ 5), mine, `EVT BATTLE_STARTED` |
| 5 s | Voit le compte à rebours | Voit le compte à rebours | Voit les boutons de vote | — |
| 10 s | Boutons de vote désactivés | Boutons de vote désactivés | `VOTE\|1\|1` | Mine le bloc de vote, `EVT VOTE_CAST` |
| 60 s | — | — | — | Calcule le vainqueur, transfère les recommandations, `EVT BATTLE_ENDED` |
| 61 s | Python : gagnante | R : `INACTIVE` | Voit le résultat | — |
