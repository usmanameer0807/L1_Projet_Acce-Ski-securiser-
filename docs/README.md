# Documentation — Phase 1

Cette documentation permet à une équipe de développement de comprendre la structure de la plateforme SAÉ Recommender et les choix techniques retenus. Elle est livrée avec le dépôt, étiquetée `phase1`, et complétée par le document unique de [spécification technique](specification-technique.md).

## Par où commencer

| Je veux… | Je lis… |
| :--- | :--- |
| Comprendre le projet d'un coup d'œil | [specification-technique.md](specification-technique.md) |
| Connaître les règles de la plateforme | [regles-metier.md](regles-metier.md) |
| Savoir qui fait quoi | [repartition-roles.md](repartition-roles.md) |
| Écrire ou tester le client | [architecture-client.md](architecture-client.md), [donnees-client.md](donnees-client.md), puis [protocole-applicatif-commun.md](protocole-applicatif-commun.md) |
| Écrire ou tester le serveur | [architecture-serveur.md](architecture-serveur.md), [donnees-serveur.md](donnees-serveur.md), puis [protocole-applicatif-commun.md](protocole-applicatif-commun.md) |
| Travailler sur la blockchain ou la base | [structures-blockchain.md](structures-blockchain.md), [schema-bdd.md](schema-bdd.md) |

## Documents communs

| Fichier | Contenu |
| :--- | :--- |
| [specification-technique.md](specification-technique.md) | Document de synthèse rédigé, avec renvois aux annexes |
| [repartition-roles.md](repartition-roles.md) | Rôles, domaines, tests croisés |
| [regles-metier.md](regles-metier.md) | Règles de la plateforme, précisions de conception, points ouverts |
| [protocole-applicatif-commun.md](protocole-applicatif-commun.md) | Contrat client-serveur : commandes, réponses, événements |
| [cas-erreur.md](cas-erreur.md) | Codes d'erreur et comportements attendus |

## Documents par domaine

| Domaine | Responsables | Fichiers |
| :--- | :--- | :--- |
| Client Java / JavaFX | Usman, Omar | [architecture-client.md](architecture-client.md), [donnees-client.md](donnees-client.md) |
| Serveur C | Mai, Albine | [architecture-serveur.md](architecture-serveur.md), [donnees-serveur.md](donnees-serveur.md) |
| Blockchain et base de données | Iyore | [structures-blockchain.md](structures-blockchain.md), [schema-bdd.md](schema-bdd.md) |

## Annexes

| Annexe | Contenu |
| :--- | :--- |
| [A — Parcours utilisateur](annexe-a-parcours-utilisateur.md) | Navigation et parcours de l'utilisateur |
| [B — Cas d'utilisation](annexe-b-cas-utilisation.md) | Cas d'utilisation global, client, serveur |
| [C — Classes](annexe-c-classes.md) | Diagrammes de classes |
| [D — Séquences](annexe-d-sequences.md) | Diagrammes de séquence client et serveur |
| [E — Déploiement](annexe-e-deploiement.md) | Architecture globale et déploiement |

## Diagrammes

Les diagrammes sont écrits en **Mermaid** à l'intérieur des fichiers Markdown : GitLab les affiche directement. Chaque diagramme est précédé d'un commentaire `<!-- png: … -->` qui donne le nom du fichier image correspondant dans `src/`.

Pour produire les images PNG :

1. copier le code du bloc `mermaid` dans <https://mermaid.live> ;
2. exporter en PNG ;
3. enregistrer le fichier dans `src/` sous le nom indiqué.

Les dossiers `src/` sont organisés ainsi :

| Dossier | Contenu |
| :--- | :--- |
| `src/diagrammes-client/` | Architecture, cas d'utilisation, classes, séquences et maquettes du client |
| `src/diagrammes-serveur/` | Cas d'utilisation, architecture, classes et séquences du serveur |
| `src/diagrammes-blockchain-bdd/` | Structure de la blockchain, schéma de base, minage, vérification, cycle de vie |
| `src/diagrammes-generaux/` | Architecture globale, cas d'utilisation global, navigation, parcours, déploiement |

`src/` est le seul sous-dossier de `docs/`.

## Types de diagrammes

Cinq types de diagrammes UML sont utilisés, conformément au sujet : cas d'utilisation, parcours utilisateur, classes, séquence, déploiement. Les schémas d'architecture, de navigation, de cycle de vie, les maquettes et le schéma relationnel sont des schémas non UML.

## Conventions

* Les identifiants de protocole, d'erreurs et de fonctions sont en anglais (`RECOMMEND`, `CARD_NOT_FOUND`, `block_mine`). Les explications sont en français.
* Un changement de protocole, de format de bloc ou de schéma de table se fait dans le document de référence, avec l'accord des équipes concernées.
* La documentation et la spécification technique doivent rester cohérentes et être mises à jour aux phases 2 et 3.
