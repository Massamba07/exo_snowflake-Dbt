# exo_snowflake-Dbt
# TP Guidé - dbt natif avec Snowflake

## Objectif pédagogique

Dans ce projet, tu vas apprendre à :

* Structurer un projet data avec dbt
* Transformer des données dans Snowflake
* Construire un pipeline analytique simple
* Tester la qualité des données

---

## Contexte métier

Tu es analytics Engineer dans une entreprise e-commerce.

On te demande :

> Calculer le chiffre d’affaires journalier basé uniquement sur les commandes complétées.

---

## Architecture cible

```
RAW → STAGING → MARTS
```

* RAW : données brutes
* STAGING : données nettoyées
* MARTS : données prêtes pour le business

---

## Étape 1 — Setup des données

### 1.1 Créer la base

Exécute ce script dans Snowflake :

```sql
CREATE OR REPLACE DATABASE DEMO_DBT_NATIVE;
CREATE OR REPLACE SCHEMA DEMO_DBT_NATIVE.RAW;

USE DATABASE DEMO_DBT_NATIVE;
USE SCHEMA RAW;

CREATE OR REPLACE TABLE ORDERS (
    ORDER_ID INTEGER,
    CUSTOMER_ID INTEGER,
    ORDER_DATE DATE,
    AMOUNT NUMBER(10,2),
    STATUS STRING
);
```

---

### 1.2 Insérer les données

```sql
INSERT INTO ORDERS VALUES
(1, 101, '2026-04-20', 120.50, 'completed'),
(2, 102, '2026-04-20', 80.00, 'completed'),
(3, 101, '2026-04-21', 45.90, 'cancelled'),
(4, 103, '2026-04-21', 210.00, 'completed'),
(5, 104, '2026-04-22', 99.99, 'completed'),
(6, 105, '2026-04-22', 150.00, 'pending'),
(7, 101, '2026-04-23', 300.00, 'completed'),
(8, 106, '2026-04-23', 50.00, 'cancelled');
```

---

## Étape 2 — Initialiser le projet dbt

Créer la structure suivante :

```
dbt_native_demo/
│
├── dbt_project.yml
├── models/
│   ├── sources.yml
│   ├── staging/
│   │   └── stg_orders.sql
│   └── marts/
│       ├── fct_daily_sales.sql
│       └── schema.yml
```

---

## Étape 3 — Configuration dbt

### Fichier `dbt_project.yml`

```yaml
name: 'dbt_native_demo'
version: '1.0'
config-version: 2

model-paths: ["models"]

models:
  dbt_native_demo:
    staging:
      +schema: STAGING
      +materialized: view
    marts:
      +schema: MARTS
      +materialized: table
```

---

## Étape 4 — Déclarer la source

### Fichier `models/sources.yml`

```yaml
version: 2

sources:
  - name: raw
    database: DEMO_DBT_NATIVE
    schema: RAW
    tables:
      - name: ORDERS
```

---

## Étape 5 — Modèle STAGING

### Fichier `models/staging/stg_orders.sql`

```sql
select
    order_id,
    customer_id,
    order_date,
    amount,
    lower(status) as status
from {{ source('raw', 'ORDERS') }}
```

Objectif : nettoyer les données sans appliquer de logique métier.

---

## Étape 6 — Modèle MART

### Fichier `models/marts/fct_daily_sales.sql`

```sql
select
    order_date,
    count(*) as total_orders,
    sum(amount) as total_amount
from {{ ref('stg_orders') }}
where status = 'completed'
group by order_date
```

Objectif : produire une table directement exploitable pour le reporting.

---

## Étape 7 — Tests de qualité

### Fichier `models/marts/schema.yml`

```yaml
version: 2

models:
  - name: fct_daily_sales
    columns:
      - name: order_date
        tests:
          - not_null
          - unique
```

Objectif : garantir la qualité des données produites.

---

## Étape 8 — Exécuter dbt

### Option 1 — CLI ou Workspace

```bash
dbt run
dbt test
```

---

### Option 2 — Snowflake natif

```sql
EXECUTE DBT PROJECT dbt_native_demo
ARGS='run';
```

---

## Étape 9 — Vérifier les résultats

```sql
SELECT *
FROM DEMO_DBT_NATIVE.MARTS.FCT_DAILY_SALES
ORDER BY ORDER_DATE;
```

---

## Résultat attendu

| order_date | total_orders | total_amount |
| ---------- | ------------ | ------------ |
| 2026-04-20 | 2            | 200.50       |
| 2026-04-21 | 1            | 210.00       |
| 2026-04-22 | 1            | 99.99        |

---

## Points clés à retenir

* `source()` permet de lire les données brutes
* `ref()` permet de chaîner les modèles
* dbt structure les transformations
* Snowflake exécute le pipeline

---

## Exercices

### Exercice 1

Ajouter une colonne :

```
avg(amount) as avg_order_value
```

---

### Exercice 2

Créer un modèle :

```
fct_sales_by_customer
```

---

### Exercice 3

Ajouter un test :

```
not_null sur total_amount
```

---

## Bonus

* Ajouter un modèle incremental
* Générer la documentation (`dbt docs`)
* Ajouter une table `CUSTOMERS`

---

## Conclusion

Ce projet montre comment passer de données brutes à des données prêtes pour l’analyse avec dbt et Snowflake.
