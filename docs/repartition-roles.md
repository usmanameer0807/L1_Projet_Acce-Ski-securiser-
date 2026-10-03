# Répartition des rôles

Équipe : Groupe 1-C, cinq étudiants. Ce document répartit les responsabilités par domaine technique et par fonction transversale.

## 1. Répartition par domaine

| Domaine | Membres | Livrables de phase 1 | Code de phase 2 |
| :--- | :--- | :--- | :--- |
| Client Java / JavaFX | Usman, Omar | `architecture-client.md`, `donnees-client.md`, `diagrammes-client/` | `phase2/client-java/` |
| Serveur C | Mai, Albine | `architecture-serveur.md`, `donnees-serveur.md`, `diagrammes-serveur/` | `phase2/server-c/` |
| Base de données et blockchain | Iyore | `schema-bdd.md`, `structures-blockchain.md`, `diagrammes-blockchain-bdd/` | modules `blockchain.c`, `db.c` avec l'équipe serveur |
| Documents communs | Toute l'équipe | règles métier, protocole, cas d'erreur, annexes, diagrammes généraux | — |

## 2. Fonctions transversales

Le cahier des charges demande de nommer un chef de projet, des concepteurs, des codeurs, des testeurs et un responsable qualité. Dans une équipe de cinq, chacun cumule plusieurs fonctions.

| Fonction | Mission | Titulaire |
| :--- | :--- | :--- |
| Chef de projet | Planning, suivi des échéances, arbitrage des désaccords, relations avec l'enseignant | À désigner par le groupe |
| Responsable qualité | Relecture des documents, cohérence entre `docs/` et la spécification, conventions de code, revue des fusions | À désigner par le groupe |
| Concepteurs | Rédaction et mise à jour des documents et diagrammes de leur domaine | Les membres de chaque domaine (section 1) |
| Codeurs (phase 2) | Implémentation du domaine | Les membres de chaque domaine (section 1) |
| Testeurs (phase 2) | Tests unitaires du domaine, tests croisés du domaine voisin | Chaque membre teste le module d'un autre domaine (voir section 3) |

> Les deux titulaires « À désigner » doivent être nommés avant la livraison du 10/10. La répartition proposée évite qu'une même personne soit à la fois chef de projet et seule responsable qualité.

## 3. Tests croisés

Un auteur ne teste pas seul son propre code : un second regard détecte plus d'erreurs.

| Code écrit par | Tests relus ou complétés par |
| :--- | :--- |
| Usman, Omar (client) | Mai, Albine (le protocole côté serveur sert de référence) |
| Mai, Albine (serveur) | Iyore (intégrité de la blockchain, état en base) |
| Iyore (blockchain, BDD) | Usman, Omar (cas d'usage de bout en bout) |

## 4. Interfaces entre domaines

Les trois domaines ne se coordonnent que par trois contrats écrits. Toute modification d'un contrat passe par une discussion de l'équipe et une mise à jour du document.

| Contrat | Entre | Document de référence |
| :--- | :--- | :--- |
| Protocole applicatif | Client ↔ Serveur | [protocole-applicatif-commun.md](protocole-applicatif-commun.md) |
| Structure des blocs et des actions | Serveur ↔ Blockchain | [structures-blockchain.md](structures-blockchain.md) |
| Schéma des tables | Serveur ↔ Base de données | [schema-bdd.md](schema-bdd.md) |

## 5. Outils et organisation

* Dépôt Git sur GitLab, une branche par fonctionnalité, fusion après relecture par un autre membre.
* Étiquettes de livraison `phase1`, `phase2`, `phase3`, `phase4`.
* Spécification technique : document unique partagé sur OneDrive avec le professeur.
* Point d'équipe hebdomadaire sur les créneaux de SAÉ. Le temps est réparti entre le projet principal (60 %) et les deux SAÉ de ressource R301 et R307 (20 % chacune).
