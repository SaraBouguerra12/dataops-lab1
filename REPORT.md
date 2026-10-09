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
