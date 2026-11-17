# Lab 2 — Kubeflow Pipelines vs Airflow : rapport comparatif

**Auteur :** Selim Marzouka · **Date :** 17 novembre 2026 · **Cas d'usage :** credit scoring (German Credit)

## 1. Contexte

Le pipeline DVC du Lab 1 (download → prepare → train → evaluate) a été porté sur deux
orchestrateurs, avec six étapes : ingest → validate (gate) → prepare → train → evaluate →
register. Les deux variantes utilisent la même image `localhost:5001/scoring-pipeline:1.0.0`
et le même code (`src/`, piloté par `dataset.yaml`) : seul le transport des données change.
Dataset : 1 000 demandes, 20 variables, 30 % de mauvais payeurs. Machine : portable de
8 Go de RAM (Ubuntu) avec 12 Go de swap sur SSD, cluster kind à 3 nœuds.

## 2. Résultats quantitatifs

| Dimension | Kubeflow Pipelines 2.17 | Airflow 3.3 (KubernetesPodOperator) |
|---|---|---|
| Installation | ~10 min (`make kfp`, 14 pods) | ~5 min (`make airflow`, 7 conteneurs) |
| RAM mesurée en plus | ≈ 1 Go | ≈ 0,9 Go |
| Lignes de code propres | 282 (components.py 193 + scoring_pipeline.py 89) | 70 (scoring_dag.py) |
| Durée d'un run | ~3 à 4 min | 53 s |
| Relance identique | ingest, validate, prepare en cache | tout est ré-exécuté |
| Résultat | roc_auc 0,8094, coût 108, @champion v7 (`launched_by=kfp`) | mêmes métriques, @champion v8 (`launched_by=airflow`) |

Les deux orchestrateurs produisent exactement les mêmes métriques que `dvc repro`
(roc_auc 0,8094, coût métier 108, matrice 67/73/7/53) : le code est bien le même.

## 3. Analyse qualitative

**Developer experience.** KFP demande plus de code (composants typés, `Input`/`Output`,
imports dans les fonctions) mais offre en retour des artefacts typés et des métriques
affichées dans l'UI, dont la matrice de confusion. Airflow s'écrit en moins de lignes,
mais il faut gérer soi-même le passage des fichiers (préfixe S3 par run) ; seules de
petites valeurs passent par XCom. Piège rencontré : un nouveau DAG n'est lu qu'après
environ 5 minutes (`airflow dags reserialize` pour forcer).

**Scaling et performance.** Sur ce petit dataset, Airflow est plus rapide : KFP lance
pour chaque étape un pod « driver » puis un pod « impl », ce qui ajoute du temps fixe.
En revanche, le cache de KFP (fondé sur les entrées, l'image et la commande) évite de
refaire les étapes données : seul l'entraînement est relancé, ce qui devient un gain net
dès que les données sont volumineuses. Les ressources se déclarent dans les deux outils
(`set_memory_limit` d'un côté, `container_resources` de l'autre).

**Debugging et lineage.** KFP affiche le DAG avec les artefacts et leurs métadonnées
(md5 du fichier brut identique à `dvc.lock`) : remonter d'un modèle à ses données prend un
clic, et le tag `kfp_run_id` relie MLflow au run KFP. Avec Airflow, les logs sont tout
aussi accessibles, mais le lineage doit être reconstitué à partir des logs et du préfixe S3.

## 4. Recommandation

Pour un pipeline d'entraînement ML (artefacts, métriques, itérations sur les
hyper-paramètres), je recommande **Kubeflow Pipelines** : typage, cache et lineage sont
natifs. **Airflow** est préférable quand le ML n'est qu'une partie d'un flux de données plus
large (ETL, planification, nombreuses intégrations) ou quand l'équipe le maîtrise déjà ;
il faut alors prévoir une convention de stockage des artefacts et un suivi via MLflow.

## 5. Limites de la comparaison

Les mesures viennent d'une seule machine de 8 Go, avec du swap : les durées sont
indicatives. KFP est en mode standalone (sans multi-utilisateur, sans Katib) et n'a pas
été testé sur un cluster managé (GKE, AKS). Airflow tourne en Docker Compose avec le
CeleryExecutor, pas avec le KubernetesExecutor. Le dataset (1 000 lignes) est trop petit
pour mesurer l'intérêt réel du cache et du parallélisme.
