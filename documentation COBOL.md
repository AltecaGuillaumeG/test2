# Documentation technique COBOL — `AltecaGuillaumeG/test2`

## 1. Cartographie des liens entre programmes

### Vue d’ensemble

Le dépôt regroupe deux programmes COBOL autonomes, reliés fonctionnellement par la table PostgreSQL `EMP`. `INSERTTBL` prépare cette table et y insère des données ; `FETCHTBL` interroge ensuite la même table, compte ses lignes et les affiche dans l’ordre des numéros d’employé.

Aucun appel COBOL direct (`CALL`) ni partage de fichier de données entre les programmes n’est visible. Le lien est une dépendance commune à la base de données et au schéma `EMP`. L’ordre d’exploitation logique est donc la préparation/initialisation, puis la consultation.

```text
                    Base PostgreSQL : testdb
                    Table EMP
                    (EMP_NO, EMP_NAME, EMP_SALARY)
                       ▲                 │
                       │ création,       │ comptage,
                       │ chargement       │ lecture triée
                       │                 ▼
                  INSERTTBL ────────► FETCHTBL
                  initialisation       consultation
```

La flèche représente un enchaînement fonctionnel via la base, et non une invocation de programme.

### Niveaux et rôles

- **Programme de préparation des données :** `INSERTTBL.cbl`. Il supprime la table `EMP` si elle existe, la recrée, puis insère des lignes littérales et un lot de dix lignes défini dans le programme. C’est le point d’entrée logique pour établir le jeu de données attendu par l’exemple.
- **Programme de consultation :** `FETCHTBL.cbl`. Il compte les lignes, ouvre un curseur sur `EMP`, récupère les lignes dans l’ordre `EMP_NO` et les affiche.
- **Programme intermédiaire ou sous-programme partagé :** aucun n’est présent dans les fichiers observés. Les deux programmes ont chacun leur `PROGRAM-ID`, leur `MAIN-RTN`, leur gestion d’erreur et leur séquence de connexion/déconnexion.

### Contrat de données partagé

| Colonne | Définition dans `INSERTTBL` | Utilisation dans `FETCHTBL` |
|---|---|---|
| `EMP_NO` | `NUMERIC(4,0) NOT NULL`, clé primaire | Hôte COBOL `EMP-NO`, affiché dans `D-EMP-NO` |
| `EMP_NAME` | `CHAR(20)` | Hôte COBOL `EMP-NAME`, affiché dans `D-EMP-NAME` |
| `EMP_SALARY` | `NUMERIC(4,0)` | Hôte COBOL `EMP-SALARY`, affiché dans `D-EMP-SALARY` |

Les champs hôtes COBOL correspondants sont définis comme `S9(04)`, `X(20)` et `S9(04)`. La compatibilité de représentation et de conversion est confiée à l’environnement Open COBOL ESQL (Ocesql) et au pilote/base utilisés ; le dépôt ne documente pas davantage ces règles.

## 2. Analyse détaillée par fichier

### `INSERTTBL.cbl`

- **Nom du programme :** `INSERTTBL`.
- **Objectif fonctionnel :** initialiser un jeu d’exemple dans la base `testdb` en reconstruisant la table `EMP` puis en y insérant des enregistrements.
- **Type de logique :** accès aux données, définition de schéma, chargement de données, boucle de traitement et contrôle conditionnel des erreurs.
- **Variables et structures clés :**
  - `TEST-DATA` contient les valeurs d’exemple ; `TEST-DATA-R REDEFINES TEST-DATA` les interprète sous forme d’une table de dix occurrences (`TEST-TBL`). Chaque occurrence comprend `TEST-NO`, `TEST-NAME` et `TEST-SALARY`.
  - `IDX` pilote le parcours des dix occurrences.
  - `DBNAME`, `USERNAME` et `PASSWD` sont les variables de connexion SQL ; le programme leur affecte respectivement `testdb`, `postgres` et un mot de passe vide.
  - `EMP-REC-VARS` regroupe les variables hôtes `EMP-NO`, `EMP-NAME` et `EMP-SALARY` utilisées par l’`INSERT` paramétré.
  - `SQLCA` est inclus pour l’état des opérations SQL ; `SQLCODE`, `SQLSTATE` et `SQLERRMC` alimentent la gestion des erreurs.
- **Mécanismes COBOL et SQL utilisés :**
  - divisions `IDENTIFICATION`, `DATA` et `PROCEDURE`, stockage `WORKING-STORAGE` et paragraphes `MAIN-RTN` / `ERROR-RTN` ;
  - `REDEFINES` et `OCCURS 10` pour présenter des données initialisées comme une table ;
  - instructions Ocesql `CONNECT`, `DROP TABLE IF EXISTS`, `CREATE TABLE`, `INSERT`, `COMMIT WORK` et `DISCONNECT ALL` ;
  - deux insertions avec valeurs SQL littérales, puis insertion par variables hôtes dans `PERFORM VARYING IDX ... UNTIL IDX > 10` ;
  - tests de `SQLCODE`, `EVALUATE` pour classifier certains codes et `EXIT PERFORM` en cas d’erreur dans la boucle.
- **Déroulement et singularités :**
  1. Connexion à `testdb` avec le compte `postgres` et un mot de passe espace.
  2. Suppression de `EMP` si elle existe, puis création avec `EMP_NO` comme clé primaire.
  3. Insertion de deux lignes codées directement dans les instructions SQL (numéros 46 et 47).
  4. Parcours des dix lignes de `TEST-TBL` et insertion par variables hôtes.
  5. Validation par `COMMIT WORK`, déconnexion et affichage du message de fin.

  La suppression préalable rend cette opération réinitialisatrice et potentiellement destructive pour les données antérieures de `EMP`. Les valeurs actives du lot d’exemple sont des noms japonais et des salaires numériques ; certaines valeurs source se terminent par la lettre `p` (par exemple `025p`). Leur conversion en `PIC S9(04)` dépend du comportement de l’environnement et n’est pas expliquée dans le dépôt ; elles méritent une vérification avant exécution avec une chaîne d’outils donnée. Des variantes de données en alphabet latin sont présentes uniquement sous forme de lignes commentées.
- **Gestion d’erreur :** après la connexion, les opérations de suppression et les insertions contrôlent `SQLCODE` et invoquent `ERROR-RTN`. La création de table et la connexion arrêtent le programme (`STOP RUN`) après erreur ; dans la boucle d’insertion, l’erreur provoque la sortie de boucle, mais le flot atteint ensuite le `COMMIT` et la déconnexion. Le paragraphe d’erreur affiche le code, et pour certains cas l’état SQL et le message du moteur ; un `ROLLBACK` est prévu pour le cas `SQLCODE -30`. Le dépôt ne montre pas de stratégie d’arrêt uniforme pour chaque erreur.
- **Relation avec les autres programmes :** établit le schéma et les données attendus par `FETCHTBL`. Il n’appelle pas ce dernier ; le transfert est assuré par la persistance dans `EMP`.

### `FETCHTBL.cbl`

- **Nom du programme :** `FETCHTBL`.
- **Objectif fonctionnel :** consulter la table `EMP`, afficher son nombre de lignes, puis présenter chaque enregistrement trié par `EMP_NO`.
- **Type de logique :** lecture SQL, parcours séquentiel par curseur, formatage de sortie et gestion conditionnelle d’erreurs.
- **Variables et structures clés :**
  - `EMP-REC-VARS` déclare les hôtes `EMP-NO`, `EMP-NAME` et `EMP-SALARY` utilisés par le curseur ; `EMP-CNT` reçoit le résultat du `COUNT(*)`.
  - `D-EMP-REC` constitue la zone de sortie assemblée : numéro, séparateurs, nom et salaire avec une présentation numérique éditée (`PIC --,--9`).
  - `DBNAME`, `USERNAME` et `PASSWD` portent les mêmes paramètres de connexion que dans `INSERTTBL`.
  - `SQLCA` fournit les variables d’état SQL exploitées par le programme.
- **Mécanismes COBOL et SQL utilisés :**
  - `CONNECT`, `SELECT COUNT(*) INTO`, déclaration et ouverture d’un curseur, `FETCH`, `CLOSE`, `COMMIT WORK` et `DISCONNECT ALL` ;
  - curseur `C1` sur `EMP`, avec `ORDER BY EMP_NO` ;
  - `PERFORM UNTIL SQLCODE NOT = ZERO` pour consommer les lignes jusqu’à la fin ou une erreur SQL ;
  - transferts des variables hôtes vers la zone d’affichage par `MOVE`, puis sortie avec `DISPLAY` ;
  - `EVALUATE SQLCODE` dans `ERROR-RTN`.
- **Déroulement et singularités :** le programme affiche un compteur, imprime un en-tête de colonnes, effectue un premier `FETCH`, puis affiche les lignes et enchaîne les `FETCH` jusqu’à un `SQLCODE` non nul. Il ferme ensuite le curseur, valide la transaction, se déconnecte et termine. Le curseur donne un ordre d’affichage explicite par numéro, mais aucune pagination ou saisie utilisateur n’est visible.
- **Gestion d’erreur :** une erreur de connexion appelle `ERROR-RTN` puis arrête le programme. Les erreurs des opérations SQL ultérieures ne sont pas systématiquement suivies d’un test explicite ; la boucle de lecture s’arrête dès que `SQLCODE` devient non nul. Le paragraphe d’erreur distingue quelques codes, affiche `SQLSTATE`/`SQLERRMC` pour certains cas et effectue un `ROLLBACK` dans le cas PostgreSQL identifié comme `-30`. Le commentaire indique que `WHENEVER SQLERROR` n’est pas pris en charge par l’outil visé.
- **Relation avec les autres programmes :** consomme le schéma et les lignes préparés par `INSERTTBL`. Son résultat dépend donc de l’existence de la table `EMP` et de la réussite du chargement, mais aucun contrôle préalable du schéma n’est visible.

## 3. Fichiers principaux à retenir

| Fichier | Rôle dans l’ensemble | Importance architecturale |
|---|---|---|
| `INSERTTBL.cbl` | Création/réinitialisation de `EMP` et chargement des données d’exemple | Programme producteur et préparation du contrat de données commun |
| `FETCHTBL.cbl` | Comptage et lecture triée des enregistrements de `EMP` | Programme consommateur et démonstration de lecture SQL par curseur |

## Synthèse architecturale

Le dépôt forme un petit parcours applicatif COBOL centré sur une base relationnelle : un programme prépare une table et ses données, l’autre démontre leur consultation. La cohérence repose sur un schéma partagé (`EMP`) et sur les mêmes paramètres de connexion, non sur un module COBOL commun. Les commentaires et les séquences SQL indiquent un exemple Open COBOL ESQL/Ocesql orienté démonstration des opérations usuelles de transaction et de curseur.

Les fichiers visibles ne fournissent pas de script de compilation, de configuration de base, de tests ni de documentation de déploiement. La configuration exacte du runtime et la procédure d’exécution doivent donc être déterminées hors du code fourni. Avant un usage réel, il convient notamment de vérifier les identifiants de connexion codés en dur, l’effet destructif de `DROP TABLE`, l’encodage des noms et les valeurs de salaire atypiques du jeu d’essai.
