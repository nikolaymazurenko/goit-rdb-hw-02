# goit-rdb-hw-02 - Customer Churn Prediction

Домашнє завдання до Теми 2 курсу "Relational Databases for Data Science": проектування реляційної БД для ML-сценарію **Customer Churn Prediction** з offline/online feature store і point-in-time (PiT) correct формуванням training dataset.

Публічний churn dataset використовується лише як доменний референс. Temporal history (версії features і training events) контрольовано змодельована, тому реальний churn timestamp із зовнішнього dataset не припускається відомим.

**Виконавець:** Мазуренко Микола

## Структура проєкту

| Файл | Опис |
|------|------|
| `hw_02.ipynb` | Основний notebook: ER-модель, DDL, дані, AS-OF запити, перевірки, аналіз нормалізації та Reflection |
| `hw_02_er_diagram.svg` | ER-діаграма схеми |

## Як запустити

1. Відкрити `hw_02.ipynb` у Google Colab.
2. Перевірити Runtime type: потрібна версія **2026.04** (в останній версії runtime `pgserver` недоступний).
3. Виконати всі клітинки по порядку (Runtime -> Run all).

Notebook встановлює `pgserver`, `sqlalchemy`, `psycopg2-binary`, `pandas` і піднімає локальний PostgreSQL 16 у `/tmp/pg_data`. Усі таблиці мають префікс `hw2_`, DDL перестворює їх з нуля, тому notebook можна перезапускати.

## Ключові архітектурні рішення

- **Business tables у 3НФ:** `customers`, `service_plans`, `subscriptions`, `payments`. Зв'язок N:M customers - plans у часі реалізований через асоціативну таблицю `subscriptions`.
- **Offline feature store** (`customer_features_offline`): історія версій features з інтервалом валідності `[valid_from, valid_to)`, PK `(customer_id, valid_from)`, `valid_to IS NULL` - поточна версія. Індекс `(customer_id, valid_from DESC)` під AS-OF запити.
- **Захист від overlap:** temporal CHECK `valid_to > valid_from` + trigger, що через `tstzrange(...) &&` забороняє перекривні інтервали для одного customer.
- **Online feature store** (`customer_features_online`): навмисно денормалізований snapshot - один рядок на customer для low-latency inference. Матеріалізується з offline source (остання версія через `DISTINCT ON`).
- **Training events** (`training_events`): `prediction_ts`, `label_horizon_end` (30-денний горизонт) і label `churn_in_30d`.
- **Дані:** 12 історичних feature-записів для 4 customers (по 3 версії на кожного).

## Що перевіряється в PiT correctness

Базова умова, яка не дозволяє training example бачити майбутню версію features:

```sql
f.valid_from <= e.prediction_ts
AND (f.valid_to > e.prediction_ts OR f.valid_to IS NULL)
```

У notebook є:

- **4 AS-OF запити:** customer 42 до і після зміни features (2024-12-15), customer 3, а також training-data JOIN за `prediction_ts`. Кожен запит перевіряється `assert` (рівно одна версія, очікуване значення `tenure_days`).
- **PiT training JOIN:** кожна training event отримує рівно одну версію features, актуальну на `prediction_ts`.
- **Overlap check:** self-join по `customer_id` повертає 0 rows - інтервали не перекриваються.
- **Демонстрація data leakage:** JOIN лише за `customer_id` дублює events і приєднує майбутні feature versions.
- **Автоматичний checklist:** кількість історичних записів, entities, версій на entity та результат overlap check.

Також notebook містить пояснення data leakage, нормалізації/денормалізації і Reflection.
