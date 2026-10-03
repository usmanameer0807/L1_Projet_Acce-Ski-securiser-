# Spécification technique — SAÉ Recommender

**Projet :** plateforme de recommandations de langages de programmation fondée sur une blockchain privée  
**Groupe :** 1-C — SAÉ BUT2 S3, 2026-2027  
**Version :** phase 1 (conception)  
**Lecteurs visés :** équipe de développement, enseignants et toute personne qui doit comprendre le projet sans lire le code

Ce document est la version Markdown de la spécification technique. Il présente de manière continue ce que les fichiers du dossier `docs/` détaillent par sujet. Les éléments les plus techniques (diagrammes, schémas, tables complètes) sont regroupés dans les annexes, référencées dans le texte. La version remise sur OneDrive reprend ce contenu ; les deux doivent rester identiques.

---

## 1. Introduction

### 1.1 Objet du projet

Le projet consiste à construire une plateforme où des utilisateurs recommandent des objets, appelés cartes, et les confrontent entre eux. Dans notre version, chaque carte représente un **langage de programmation**. La communauté peut ainsi dire quels langages elle juge pertinents, pour quel domaine d'application, et départager deux langages concurrents par un vote.

La plateforme suit une architecture client-serveur. Un **serveur** unique, écrit en C, centralise les règles et les données. Plusieurs **clients**, écrits en Java avec une interface JavaFX, se connectent à lui par sockets. Toute action acceptée par le serveur est inscrite dans une **blockchain privée simplifiée** : une suite de blocs liés par des empreintes SHA-256, qui rend toute modification a posteriori détectable. La blockchain est sauvegardée dans une base **PostgreSQL**, ce qui permet au serveur de retrouver tout son contexte après un redémarrage.

### 1.2 Pourquoi une blockchain ici

La blockchain n'est pas là pour décentraliser : il n'y a qu'un serveur. Elle sert à deux choses que le sujet demande explicitement. D'abord, **tracer** chaque action validée de façon immuable : une recommandation répudiée disparaît de la valeur d'une carte, mais son historique reste. Ensuite, **vérifier l'intégrité** : si quelqu'un modifie directement une donnée en base, la vérification de la chaîne le détecte. La preuve de travail, qui impose de chercher un nonce, rend la modification d'un bloc coûteuse à dissimuler.

### 1.3 Périmètre

Sont dans le périmètre : la création de cartes, la recommandation et sa répudiation, les battles avec vote, la légitimation, la revendication, l'authentification, la restauration du contexte, la vérification de la blockchain, l'arrêt propre du serveur, une interface graphique et un client automatique facultatif.

Sont hors périmètre : le partage ciblé de cartes (facultatif dans le sujet), le chiffrement du canal, la gestion de mots de passe, toute interface autre que les sockets (notamment une API REST).

### 1.4 Technologies

Le sujet impose Linux Debian, le C pour le serveur, Java et JavaFX pour les clients, SHA-256, PostgreSQL et Git. Côté C, nous utilisons uniquement les bibliothèques autorisées : OpenSSL pour le hachage, `libpq` pour PostgreSQL, et les interfaces POSIX du cours R305 (`pthread`, sockets, `time`). Côté Java, nous utilisons JavaFX et JUnit.

---

## 2. Organisation du projet

### 2.1 Équipe et rôles

Le groupe compte cinq étudiants répartis en trois domaines : le client (Usman et Omar), le serveur (Mai et Albine) et la base de données avec la blockchain (Iyore). Des fonctions transversales (chef de projet, responsable qualité, testeurs) complètent cette répartition. Elle est détaillée dans [repartition-roles.md](repartition-roles.md).

Ce découpage correspond aux trois contrats qui relient les parties : le protocole applicatif entre client et serveur, la structure des blocs entre serveur et blockchain, le schéma des tables entre serveur et base. Chaque équipe peut avancer en s'appuyant sur ces contrats écrits, sans attendre le code des autres.

### 2.2 Planning

Le projet suit quatre phases.

| Phase | Période | Contenu | Livrable |
| :--- | :--- | :--- | :--- |
| 1 | début septembre au 10 octobre 2026 | Conception et spécification | Dépôt étiqueté `phase1`, spécification sur OneDrive |
| 2 | 12 octobre au 18 décembre 2026 | Codage et tests | Dépôt étiqueté `phase2` |
| 3 | 4 au 15 janvier 2027 | Qualité et consolidation | Dépôt étiqueté `phase3`, diagramme de Gantt |
| 4 | 18 au 21 janvier 2027 | Présentation et démonstration | Document de présentation, étiqueté `phase4` |

Le temps de travail est partagé avec deux autres SAÉ liées aux ressources R301 et R307. Le projet principal représente 60 % du temps. Un diagramme de Gantt réel sera ajouté en phase 3, avec les écarts par rapport à ce planning initial.

### 2.3 Gestion du code

Le code est géré avec Git sur GitLab. Une branche par fonctionnalité, une relecture par un autre membre avant fusion, et une étiquette par phase. Le dépôt est organisé par phase (voir le [README racine](../../README.md)).

---

## 3. Concepts fonctionnels

### 3.1 La carte

Une carte décrit un langage de programmation : son nom, son créateur historique, son année de création, un domaine d'application et une description. Elle a un identifiant unique donné par le serveur, un propriétaire courant, un créateur, une valeur et un statut. Par défaut, celui qui crée une carte en est le propriétaire.

Le **domaine** (`Web`, `GameDev`, `DataScience`, `Systems`, `Mobile`) joue un rôle central : c'est le critère qui décide si deux cartes peuvent s'affronter. Nous l'avons choisi parce qu'il est objectif, vérifiable par le serveur sans jugement, et qu'il rend les confrontations sensées : comparer Python et R est plus parlant que comparer Rust et PHP.

### 3.2 Valeur d'une carte

La valeur d'appréciation (VA) d'une carte est le nombre de ses recommandations actives, plus 10 si elle est authentifiée. Ce choix est volontairement simple : chaque utilisateur peut recommander une carte une seule fois, la répudiation retire exactement un point, et l'authentification apporte un bonus fixe qui récompense une carte reconnue par son représentant légitime.

### 3.3 Recommandation et répudiation

Recommander une carte, c'est lui apporter un soutien, comme un « like ». Il n'existe pas de vote négatif : un utilisateur peut seulement retirer son soutien en répudiant sa recommandation. Cette asymétrie évite le « dénigrement » entre utilisateurs. La répudiation retire la contribution à la valeur, mais le bloc de la recommandation d'origine reste dans la blockchain.

### 3.4 Battles

Une battle confronte deux cartes du même domaine pendant 60 secondes. Pour la lancer, il faut posséder au moins l'une des deux cartes, et chacune doit valoir au moins 5. Le seuil évite de confronter des cartes sans soutien. Les cartes engagées passent au statut `EN_BATTLE` et ne peuvent plus participer à une autre battle.

Tous les utilisateurs connectés peuvent voter, sauf les propriétaires des deux cartes, avec un vote par personne. En cas d'égalité, la carte qui avait la plus grande VA au lancement l'emporte, puis la plus ancienne. Un vainqueur est donc toujours désigné. La carte gagnante récupère les recommandations actives de la perdante, qui devient `INACTIVE`. Aucun bloc n'est supprimé : l'histoire de la carte perdante reste dans la chaîne.

### 3.5 Légitimation, revendication, authentification

Ces trois mécanismes forment une chaîne de confiance simulée, sans vérification d'identité réelle. Un utilisateur qui a donné au moins trois recommandations actives peut demander sa **légitimation**. Une fois légitimé, il peut **revendiquer** une carte créée par un tiers non légitimé : il en devient propriétaire, le créateur d'origine restant mémorisé. Enfin, le propriétaire légitime d'une carte peut l'**authentifier** : la carte gagne un badge, dix points de VA, et ses caractéristiques sont verrouillées.

### 3.6 Limites

La plateforme accepte 20 connexions simultanées et chaque utilisateur peut posséder 10 cartes actives. Ces plafonds protègent le serveur et rendent les démonstrations prévisibles.

Les règles complètes sont dans [regles-metier.md](regles-metier.md), qui contient aussi neuf précisions de conception et trois points laissés ouverts.

---

## 4. Architecture générale

Le système comprend trois ensembles : des clients, un serveur, une base de données. Les clients ne parlent qu'au serveur, par sockets TCP. Seul le serveur accède à PostgreSQL. Le schéma de l'architecture globale et le diagramme de déploiement sont dans l'[annexe E](annexe-e-deploiement.md).

La séparation est stricte pour une raison simple : toutes les règles métier vivent à un seul endroit, le serveur. Un client, qu'il soit humain ou automatique, ne peut pas tricher, puisque le serveur revalide chaque demande. Le client contrôle les champs avant l'envoi uniquement pour le confort de l'utilisateur.

---

## 5. Le serveur

### 5.1 Multitâche

Nous avons choisi les **threads POSIX**, un thread par client. Les threads partagent la mémoire du processus, ce qui convient à un état commun (cartes, recommandations, blockchain). Le choix de plusieurs processus aurait imposé de la mémoire partagée et des échanges entre processus, sans bénéfice ici. Le nombre de threads est borné par la limite de 20 clients.

### 5.2 Le traitement d'une action

Toutes les actions qui modifient la plateforme suivent le même enchaînement, implémenté une seule fois. Le serveur contrôle les champs, valide l'action selon les règles métier, puis construit un bloc et le mine. Après le minage, il vérifie que personne n'a ajouté de bloc entre-temps, revalide l'action, ajoute le bloc à la chaîne, applique l'action à l'état en mémoire et sauvegarde le tout en base dans une transaction. Enfin il répond au client et prévient les autres par des événements.

Le point délicat est le minage. C'est un calcul long qui ne doit pas bloquer les autres clients, comme le demande le sujet. Nous avons donc choisi un **minage optimiste** : le thread mine sans tenir aucun verrou, en partant du dernier bloc connu. Si un autre thread a ajouté un bloc pendant ce temps, le hash précédent n'est plus bon et le thread recommence avec le nouveau dernier bloc. Cette situation est rare avec 20 clients et une difficulté de 3, et coûte quelques millisecondes. La revalidation avant ajout est nécessaire parce que l'état a pu changer pendant le minage : par exemple, deux utilisateurs qui recommandent la même carte au même instant restent soumis à la règle d'unicité.

Un principe garantit la cohérence : **l'état de la plateforme ne change qu'à un seul endroit**, la fonction qui applique un bloc. Le fonctionnement normal et la restauration au démarrage utilisent la même fonction, donc ils ne peuvent pas diverger. La synchronisation (quatre verrous et leur ordre d'acquisition) est décrite dans [architecture-serveur.md](architecture-serveur.md) §4.

### 5.3 Battles et minuteries

Chaque battle possède un thread qui attend 60 secondes. À l'échéance, il ferme les votes, calcule le vainqueur et passe par le même enchaînement qu'une action utilisateur pour inscrire le résultat. Les votes sont inscrits dans la blockchain un par un : ils restent traçables et une battle interrompue par un redémarrage peut être clôturée correctement.

### 5.4 Démarrage, arrêt, déconnexions

Au démarrage, le serveur charge la blockchain depuis la base, la vérifie et rejoue ses blocs. Si la base est indisponible, il considère qu'aucune sauvegarde n'existe et crée un bloc genesis : il fonctionne alors sans sauvegarde et le signale. L'arrêt est déclenché par un signal ou par la console ; il prévient les clients, attend les threads et libère toutes les ressources. Une déconnexion, volontaire ou brutale, libère l'emplacement du client sans modifier la blockchain : une action non confirmée est simplement abandonnée.

Le détail est dans [architecture-serveur.md](architecture-serveur.md) et [donnees-serveur.md](donnees-serveur.md).

---

## 6. Les clients

Le client suit le patron MVC. Un **modèle** observable regroupe les cartes, utilisateurs et battles connus. Des **vues** FXML s'y lient et se mettent à jour d'elles-mêmes. Des **contrôleurs** traitent les actions de l'utilisateur. Une couche réseau isole le socket : un thread de lecture dédié reçoit les messages du serveur, de sorte que le thread de l'interface n'attend jamais le réseau.

Chaque requête est envoyée sans bloquer, et la réponse arrive dans un `CompletableFuture`. Les événements du serveur (une carte créée, un vote, la fin d'une battle) mettent à jour le modèle directement, ce qui maintient tous les clients synchronisés. L'interface comprend six écrans : connexion, écran principal, cartes, création de carte, battles et profil. Les maquettes et la navigation sont dans [architecture-client.md](architecture-client.md), les parcours dans l'[annexe A](annexe-a-parcours-utilisateur.md).

Un **client automatique** (bot) réutilise la couche réseau sans interface. Il se déclare `BOT` à la connexion, ce qui le rend identifiable par le serveur et par les autres utilisateurs. Pour ne pas fausser la valeur des cartes, il ne recommande jamais : il crée des cartes de test et vote dans les battles.

---

## 7. La blockchain

La blockchain est une liste chaînée de blocs. Chaque bloc contient un identifiant, un horodatage, le type et les données de l'action, le hash du bloc précédent, un nonce et son propre hash. Le hash est le SHA-256 de ces éléments (le pointeur de chaînage en mémoire n'y entre pas). Le premier bloc, le genesis, a un hash précédent nul.

Le **minage** consiste à faire varier le nonce jusqu'à obtenir un hash qui commence par trois zéros hexadécimaux, soit environ 4 000 calculs en moyenne. La **vérification** parcourt la chaîne et contrôle pour chaque bloc que son hash est correct, que son hash précédent correspond au bloc d'avant et que la preuve de travail est respectée. Modifier une donnée d'un bloc rend son hash invalide : l'altération est détectée sur le bloc modifié.

Chaque type d'action possède un format de données fixe, ce qui permet de reconstruire tout l'état à partir de la chaîne. Les formats, la structure du bloc et les diagrammes sont dans [structures-blockchain.md](structures-blockchain.md).

---

## 8. La base de données et la cohérence

La table `blocks` est la sauvegarde de la blockchain, qui reste la référence. Pour éviter de parcourir toute la chaîne à chaque consultation, nous conservons aussi des tables d'**état courant** (utilisateurs, cartes, recommandations, battles, votes). Elles sont dérivées de la blockchain.

Pour que les deux restent cohérentes, un bloc et la mise à jour de l'état qu'il provoque sont écrits dans **une seule transaction**. Si l'écriture échoue, rien n'est écrit et le bloc reste en attente de sauvegarde, rejoué dans l'ordre à la prochaine occasion. Si une incohérence survient malgré tout, les tables d'état sont reconstruites entièrement depuis `blocks`, avec le même code d'application des blocs qu'en fonctionnement normal. Le schéma relationnel, les contraintes (comme l'unicité d'une recommandation active) et la procédure de reconstruction sont dans [schema-bdd.md](schema-bdd.md).

---

## 9. Le protocole applicatif

La communication utilise TCP et un protocole **texte**, une ligne par message, champs séparés par `|`. Ce choix permet de tester le serveur avec `nc` et évite d'écrire un analyseur JSON en C. Le client envoie une requête, attend la réponse (`OK` ou `ERR`) puis peut en envoyer une autre. Le serveur peut en plus envoyer à tout moment des événements `EVT`, qui informent tous les clients des changements.

Les commandes couvrent la session (`LOGIN`, `DISCONNECT`), les huit actions enregistrées dans la blockchain et les consultations (`LIST_*`, `GET_PROFILE`, `VERIFY_CHAIN`). Chaque refus renvoie un code d'erreur précis. La spécification complète, avec un exemple de dialogue et les alternatives écartées, est dans [protocole-applicatif-commun.md](protocole-applicatif-commun.md).

---

## 10. Gestion des erreurs

Nous traitons les erreurs les plus représentatives : requêtes mal formées, serveur plein, nom déjà connecté, violation de chaque règle métier, panne de la base de données, coupure d'un client en pleine action, conflit pendant le minage et altération de la blockchain. Pour chacune, [cas-erreur.md](cas-erreur.md) donne le code, la situation et le comportement attendu. Le principe commun : une action refusée ou interrompue n'est jamais inscrite dans la blockchain et ne laisse aucune trace partielle.

---

## 11. Stratégie de test

Les tests seront écrits en phase 2.

* **Serveur (C)** : fonctions appelées directement avec `assert`, sans framework. Cas couverts : chaque action en succès et en refus, le minage, la vérification, les accès concurrents (plusieurs threads qui recommandent la même carte), la restauration du contexte.
* **Test d'altération** : une donnée d'un bloc est modifiée, puis la vérification doit désigner ce bloc avec la raison `BAD_HASH`.
* **Client (Java)** : JUnit sur l'analyseur de protocole, sur le modèle et sur un faux serveur local.
* **Bout en bout** : un scénario avec plusieurs clients et un bot, qui reprend les étapes de la démonstration finale.

---

## 12. Choix laissés aux étudiants

Le sujet laisse dix-sept choix. Voici ceux du groupe et où ils sont justifiés.

| Choix | Décision du groupe | Détail |
| :--- | :--- | :--- |
| Nature des cartes | Langages de programmation | §3.1 |
| Informations d'une carte | Nom, créateur historique, année, domaine, description | [regles-metier.md](regles-metier.md) §1 |
| Proximité entre cartes | Même domaine | §3.4 |
| Règles de vote | 60 s, un vote par utilisateur, propriétaires exclus | §3.4 |
| Organisation des battles | Deux cartes, statut `EN_BATTLE`, VA minimale de 5 | §3.4 |
| Valeur d'une carte | Recommandations actives + 10 si authentifiée | §3.2 |
| Valeur d'un utilisateur | Somme des VA de ses cartes (affichée dans le profil) | [architecture-client.md](architecture-client.md) §8.6 |
| Légitimation | Au moins 3 recommandations actives, processus simulé | §3.5 |
| Revendication | Transfert direct par le serveur, légitimé contre non légitimé | §3.5 |
| Authentification | Propriétaire légitime, badge, +10 VA, caractéristiques verrouillées | §3.5 |
| Transaction | Sans acceptation : validée directement par le serveur | §3.5 |
| Client automatique | Oui, sans interface, ne recommande jamais | §6 |
| Comportement du bot | Crée des cartes de test, vote au hasard | §6 |
| Partage ciblé | Non retenu | §1.3 |
| Protocole applicatif | Texte, lignes, séparateur `\|` | §9 |
| Interface graphique | Six écrans JavaFX | §6 |
| Cas d'erreur traités | Voir la liste | §10 |

---

## 13. Risques et points ouverts

| Risque ou point ouvert | Mesure |
| :--- | :--- |
| Interblocage entre verrous | Ordre d'acquisition fixé et documenté ([architecture-serveur.md](architecture-serveur.md) §4.2) |
| Minage répété à cause de conflits | Rare avec 20 clients et difficulté 3, mesuré en phase 2 |
| Base indisponible pendant la démonstration | Mode dégradé, démarrage avec genesis |
| Écart entre documents et code | Mise à jour des diagrammes en phases 2 et 3, responsable qualité |
| Trois points de règles non tranchés (recommander sa propre carte, doublons de noms, réactivation) | À décider avant la phase 2 ([regles-metier.md](regles-metier.md) §6) |
| Deux fonctions à attribuer (chef de projet, responsable qualité) | Avant la livraison ([repartition-roles.md](repartition-roles.md) §2) |

---

## Annexes

Les annexes regroupent les éléments techniques détaillés. Chacune est expliquée dans le texte ci-dessus.

| Annexe | Contenu | Référencée en |
| :--- | :--- | :--- |
| [A — Parcours utilisateur](annexe-a-parcours-utilisateur.md) | Navigation, parcours principal, échanges GUI-modèle-serveur | §6 |
| [B — Cas d'utilisation](annexe-b-cas-utilisation.md) | Diagrammes global, client, serveur ; correspondance avec les 27 cas du sujet | §1.3 |
| [C — Classes](annexe-c-classes.md) | Diagrammes de classes du client et du serveur | §5, §6 |
| [D — Séquences](annexe-d-sequences.md) | Séquences client et serveur de chaque action | §5.2, §6 |
| [E — Déploiement](annexe-e-deploiement.md) | Architecture globale et déploiement | §4 |

Les cinq types de diagrammes UML utilisés sont : cas d'utilisation, parcours utilisateur (diagramme d'activité), classes, séquence et déploiement. Les schémas d'architecture, de navigation, de cycle de vie des blocs, les maquettes et le schéma relationnel ne sont pas des diagrammes UML.
