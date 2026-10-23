# Rapport - Guided Lab 1 : d'un script fragile à un pipeline testé

## Étape 0

**Q1. Pourquoi écrire les bibliothèques dans `requirements.txt` plutôt que taper `pip install pandas` ?**
Parce que le fichier rend l'environnement reproductible : une seule commande (`pip install -r requirements.txt`) reconstruit les mêmes dépendances sur une autre machine, avec un minimum de version commun. Le fichier est versionné avec le code, donc chaque commit garde la liste des dépendances qui allait avec.

## Étape 1

**Q2. Combien de lignes le rapport affiche-t-il pour 5 pays ? Pourquoi ?**
8 lignes au lieu de 5. Pandas traite `tunisia`, ` Tunisia` (avec un espace) et `MOROCCO` comme des pays différents de `Tunisia` et `Morocco`, à cause de la casse et des espaces.

**Q3. Problèmes dans le fichier**

| order_id | Problème | Dimension |
|---|---|---|
| 1004 | quantité négative (-2) | Validité |
| 1008 | customer_id manquant | Complétude |
| 1013 | date impossible (2026-13-45) | Validité |
| 1010 (2e ligne) | commande présente deux fois | Unicité |
| 1019 | prix de 4500 pour un clavier à 45 | Exactitude |
| 1006, 1016, 1023 | `tunisia`, `MOROCCO`, ` Tunisia` | Cohérence |

**Q4. Le total est-il trop haut ou trop bas ?**
Trop haut (12771.10 contre 7655.90 corrigé). On peut le soupçonner sans tout recalculer : le prix de 4500 et le doublon gonflent le total, alors que la quantité négative le réduit légèrement. Sans recalcul, on ne peut cependant pas être sûr du sens global ni de l'ampleur de l'écart.

## Étape 2

**Q5. Deux avantages du découpage en fonctions**
1. Chaque étape peut être testée seule sur une petite table (tests unitaires).
2. On peut modifier ou remplacer une étape sans toucher aux autres, et une erreur est localisée plus vite dans l'étape concernée.

## Étape 3

**Q6. Total correct et erreur de l'ancien script**
Le total correct est 7655.90. L'ancien script affichait 12771.10, soit 5115.20 de trop : environ 66,8 % d'erreur par rapport au total correct.

**Q7. Quelle commande a causé l'essentiel de l'erreur ?**
La commande 1019 (clavier à 4500.0, Italie) : elle pèse 4500 sur les 5115.20 d'écart. Elle a été rattrapée par la règle 4 (`invalid_unit_price`).

**Q8. Pourquoi sauvegarder les lignes rejetées ?**
Pour garder une trace : on peut comprendre la cause, faire corriger la source et retraiter les lignes, au lieu de les perdre sans explication.

## Étape 4

**Q9. Que s'est-il passé en cassant la règle 3 (`> 0` remplacé par `> -5`) ?**
La quantité -2 n'est plus rejetée, car -2 > -5. Le test `test_negative_quantity_is_rejected` a échoué, ce qui montre que les tests détectent une régression de la logique.

**Q10. Test unitaire et contrôle de données**
Un test unitaire vérifie que mon code est correct sur une petite table dont je connais la réponse. Un contrôle de données vérifie que les données réelles du jour sont acceptables.

## Étape 6

**Q11. Comparaison**

| | Ancien script | Mon pipeline |
|---|---|---|
| Jour 2 : message d'erreur ? | Aucun | `PIPELINE STOPPED: Reject rate 100% ...` |
| Jour 2 : total affiché | 270200.0 | Aucun (rien publié) |
| Jour 3 : message d'erreur ? | `KeyError: 'unit_price'` | `PIPELINE STOPPED: Missing columns: ['unit_price']` |

**Q12. Changements dans le système source**
Jour 2 : les prix sont multipliés par 100 (probablement exprimés en centimes). Jour 3 : la colonne `unit_price` a été renommée `price`.

**Q13. Quel jour est le plus dangereux sans pipeline ?**
Le jour 2. Le script s'exécute sans erreur et publie un total faux (270200) que personne ne remet en question, alors que le jour 3 plante de façon visible.

**Q14. Qui peut utiliser le code de sortie 1 ?**
Un planificateur (cron), un orchestrateur, un pipeline CI/CD ou un outil de supervision : ils lisent le code pour bloquer la suite, relancer ou envoyer une alerte.


TP2 : 
Q1 : non, main n’avait pas la règle 6 pendant le travail sur la branche. Elle y est apparue au git merge --no-ff (commit « Merge the country rule »). C’est utile car main reste stable : le rapport de nuit ne tourne jamais sur du code à moitié fini ou non testé.
Q2 : une pull request apporte une relecture par un collègue, une discussion, et l’exécution automatique du CI avant la fusion. Elle laisse aussi une trace de la décision.
Q3 : recopiez la ligne md5 de data/raw.dvc. Elle identifie le contenu exact du dossier (empreinte). Si un seul caractère change dans un CSV, l’empreinte change entièrement.
Q4 : c’est data/.gitignore, qui contient /raw.
Q5 : les fichiers sont nommés par leur empreinte md5 (dossier de 2 caractères, puis le reste). Deux versions du même fichier ont des empreintes différentes, donc ne s’écrasent jamais, et un contenu identique n’est stocké qu’une fois.
Q6 : le pointeur ne contient que l’empreinte, la taille et le nombre de fichiers : un changement de données change ces lignes, donc Git voit un diff minuscule. Il faut commiter car sans commit, aucune version de Git (aucun tag) n’est liée à ces nouvelles données.
Q7 : non, le code n’a pas changé entre data-v1 et data-v2. Totaux : 7655.90 puis 8835.40.
Q8 : git checkout data-v1 -- data/raw.dvc a remis le pointeur de l’ancienne version ; dvc checkout a remplacé les fichiers de data/raw pour qu’ils correspondent à ce pointeur.
Q9 : sans dvc checkout, le pointeur et les données ne correspondent plus. Le rapport peut être produit avec les mauvaises données tout en laissant croire à la version voulue, ce qui est une erreur silencieuse.
Q10 : git clone a donné le code, l’historique, les tags et les pointeurs .dvc (dont data/raw.dvc), mais pas les CSV. dvc pull a téléchargé depuis le remote les données correspondant au pointeur.
Q11 : sans dvc push à l’étape 4, le remote ne contiendrait que data-v1. Le dvc pull du collègue échouerait (fichiers manquants dans le remote) ou, selon l’état, ne récupérerait pas la v2 ; il n’obtiendrait jamais 8835.40, et au mieux 7655.90 avec l’ancienne version.
Q12 : le TP a réduit surtout « ça marche sur ma machine » (étape 6, git clone + dvc pull redonnent le même total) et les erreurs silencieuses (étapes 4 et 5, chaque rapport est relié à ses données par une empreinte et un tag). Un peu de livraison lente aussi, car le retour à une version passée prend quelques commandes (étape 5).

## TP2

**Q1. Pendant le travail sur la branche, `main` contenait-elle la règle 6 ?**
Non. La règle 6 est apparue sur `main` au `git merge --no-ff feature/country-rule` (commit « Merge the country rule »). C'est utile car `main` reste stable : le rapport de nuit ne tourne jamais sur du code à moitié fini ou non testé.

**Q2. Qu'apporte une pull request par rapport à un `git merge` local ?**
Une relecture par un collègue, une discussion, l'exécution automatique du CI avant la fusion, et une trace écrite de la décision.

**Q3. Ligne md5 de `data/raw.dvc`**
`- md5: 6ba3aa1b65216de95bf9d5faaddad416.dir`
Elle identifie le contenu exact du dossier `data/raw`. Si un seul caractère change dans un CSV, l'empreinte change entièrement.

**Q4. Quel fichier fait que Git ignore les CSV ?**
`data/.gitignore`, qui contient `/raw`.

**Q5. Comment sont nommés les fichiers de `dvc-storage` ?**
Par leur empreinte md5 : un dossier de 2 caractères, puis le reste de l'empreinte. Deux versions d'un même fichier ont des empreintes différentes, donc ne s'écrasent jamais, et un contenu identique n'est stocké qu'une seule fois.

**Q6. Pourquoi deux lignes modifiées suffisent-elles, et pourquoi commiter ?**
Le pointeur ne contient que l'empreinte, la taille et le nombre de fichiers : un changement de données modifie ces lignes, donc le diff est minuscule. Il faut commiter, sinon aucun commit ni aucun tag ne relie la nouvelle version des données au code.

**Q7. Le code a-t-il changé entre `data-v1` et `data-v2` ?**
Non. Seules les données ont changé. Total avec `data-v1` : 7655.90. Total avec `data-v2` : 8835.40.

**Q8. Rôle de `git checkout data-v1 -- data/raw.dvc` et de `dvc checkout`**
La commande Git a remis le pointeur de l'ancienne version. `dvc checkout` a remplacé les fichiers de `data/raw` pour qu'ils correspondent à ce pointeur.

**Q9. Que se passe-t-il si on oublie `dvc checkout` ?**
Le pointeur annonce `data-v1` mais les fichiers restent ceux de `data-v2` : `dvc status` signale `modified: data\raw` et le pipeline affiche toujours 8835.40. Le rapport est produit avec de mauvaises données en laissant croire à la bonne version : c'est une erreur silencieuse.

**Q10. Que donnent `git clone` et `dvc pull` ?**
`git clone` donne le code, l'historique, les tags et les pointeurs `.dvc`, mais pas les CSV. `dvc pull` télécharge depuis le remote les données correspondant au pointeur.

**Q11. Et si `dvc push` avait été oublié à l'étape 4 ?**
Le remote ne contiendrait que `data-v1`. Le `dvc pull` du collègue échouerait (fichiers manquants) : il n'obtiendrait jamais 8835.40, au mieux l'ancienne version (7655.90).

**Q12. Quelles douleurs de la séance 1 ont été réduites ?**
« Ça marche sur ma machine » : `git clone` + `dvc pull` redonnent le même code, les mêmes données et le même total (étape 6). Erreurs silencieuses : chaque rapport est relié à ses données par une empreinte et un tag (étapes 4 et 5). Livraison lente : réduite en partie, car revenir à une version passée ne prend que quelques commandes (étape 5). Les étapes manuelles restent à traiter (CI/CD, séance suivante).

### Historique Git

```text
* 22edade (HEAD -> main, tag: data-v2) Data v2: late orders for 2026-10-05
* 98c97e3 Add the DVC remote
* 705eded (tag: data-v1) Track raw data with DVC
*   f1b4d83 Merge the country rule
|\  
| * 42e97d7 Reject orders from countries the shop does not sell to
|/  
* b383e0e Finish TP1, add DVC to requirements
* 9c6d957 (origin/main) Add CI workflow
* 4ff7854 Add REPORT.md with answers
* b348a92 Add pipeline, data checks and unit tests
* fc72483 Empty project with requirements
```
