# Homework 7 — PostgreSQL: Time-Series ML Features, JSONB and Indexing

## 1. Project Overview

Цей проєкт демонструє використання PostgreSQL для підготовки даних до Machine Learning, роботи з часовими рядами, JSONB-документами та оптимізації SQL-запитів.

У межах роботи виконано п'ять завдань:

1. Налаштування PostgreSQL та завантаження NYC Taxi Dataset.
2. Створення Time-Series ML Features за допомогою Window Functions.
3. Генерація та аналіз JSONB event-log.
4. Оптимізація JSONB-запитів за допомогою GIN та B-tree індексів.
5. Порівняння JSONB і нормалізованої моделі зберігання даних.

## 2. Data Source

**Dataset:** NYC TLC Yellow Taxi Trip Records — March 2024.

**Офіційне джерело:**

https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

**Parquet-файл:**

https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-03.parquet

Датасет містить інформацію про поїздки таксі в Нью-Йорку: час посадки та висадки, зони, відстань, вартість і кількість пасажирів.

Для роботи використано випадкову вибірку до 500 000 записів з оригінального Parquet-файлу.

Перед завантаженням у PostgreSQL виконано очищення даних:

- `fare_amount > 0` та `fare_amount < 500`
- `trip_distance > 0` та `trip_distance < 100`
- Видалення записів із порожнім `PULocationID`

Основна таблиця: `nyc_taxi_raw`.

## 3. Time-Series ML Features

На основі NYC Taxi Dataset створено таблицю `t7_hw_taxi_daily`, яка містить денні агреговані показники для кожної pickup-zone.

Для генерації ML-ознак використано PostgreSQL Window Functions з `PARTITION BY pickup_zone_id`.

**Створені features:**

| Feature | Description |
|---|---|
| `trips_lag_1d` | Кількість поїздок за попередній запис |
| `trips_lag_7d` | Кількість поїздок за 7 попередніх записів |
| `trips_lag_30d` | Кількість поїздок за 30 попередніх записів |
| `rolling_avg_fare_7d` | Середнє значення денного тарифу за останні 7 календарних днів |
| `rolling_std_fare_30d` | Стандартне відхилення денного тарифу за 30 календарних днів |
| `cumulative_trips` | Накопичена кількість поїздок |
| `expanding_avg_trips` | Середня кількість поїздок від початку спостережень |

**Фінальний VIEW:** `v_t7_hw_taxi_features`.

Примітка: функція `LAG()` працює з попередніми рядками. Якщо для зони відсутні календарні дні, відстань між записами може перевищувати один день.

Для формування повного набору ознак записи без `trips_lag_30d` відфільтровано.

## 4. JSONB Event Log

Створено синтетичну таблицю `t7_hw_events`, яка містить 50 000 подій п'яти типів:

- `page_view`
- `click`
- `add_to_cart`
- `checkout`
- `recommendation_shown`

Кожен тип події має власну структуру JSONB-поля `payload`.

Для аналізу подій реалізовано:

- JSONB containment operator `@>`
- Key existence operators `?` та `?|`
- JSONPath functions `jsonb_path_query()` і `jsonb_path_exists()`

## 5. Indexing and Query Optimization

Для оптимізації JSONB-запитів створено два індекси:

- GIN index із `jsonb_path_ops` для containment-запитів.
- Expression B-tree index для порівняння `payload ->> 'device'`.

Продуктивність виміряно за допомогою `EXPLAIN (ANALYZE, BUFFERS)`.

**Результати експериментів:**

| Query | Before | After |
|---|---:|---:|
| GIN containment | 31.778 ms | 8.494 ms |
| B-tree device equality | 12.104 ms | 3.293 ms |

Після створення індексів PostgreSQL використовував `Bitmap Index Scan` та `Bitmap Heap Scan` замість `Seq Scan`.

Результати демонструють вплив індексів на продуктивність вибіркових запитів.

## 6. JSONB vs Normalized Data

Для порівняння моделей зберігання створено нормалізовану таблицю `t7_hw_checkouts_norm`.

Виконано однакову агрегацію середньої вартості замовлень за методами оплати.

**Результати:**

| Storage Model | Execution Time |
|---|---:|
| JSONB | 14.622 ms |
| Normalized | 4.644 ms |

У проведеному експерименті нормалізована версія була приблизно у 3.15 раза швидшою.

Водночас порівнювалися таблиці різного розміру, тому результат не означає, що нормалізована модель завжди швидша за JSONB.

JSONB підходить для різнорідних event-log, тоді як типізовані SQL-таблиці зручні для стабільних ML-ознак та аналітичних запитів.

## 7. Technologies

У проєкті використано:

- Python 3
- PostgreSQL
- pgserver
- SQLAlchemy
- psycopg2-binary
- pandas
- PyArrow
- Jupyter Notebook

## 8. How to Run

**1. Відкрити Jupyter Notebook.**

Для роботи потрібне середовище Python із підтримкою Jupyter Notebook.

**2. Встановити залежності:**

`%pip install -q pgserver psycopg2-binary sqlalchemy pandas pyarrow requests`

**3. Запустити PostgreSQL.**

Виконати першу комірку notebook для ініціалізації `pgserver` та створення підключення `engine`.

**4. Завантажити NYC Taxi Dataset.**

Notebook автоматично завантажує Parquet-файл з офіційного джерела NYC TLC та зберігає його в папці `data/`.

**5. Виконати всі комірки notebook послідовно:**

- Task 1 — Database setup and data loading.
- Task 2 — Time-series feature engineering.
- Task 3 — JSONB event-log.
- Task 4 — GIN indexes and EXPLAIN ANALYZE.
- Task 5 — JSONB vs normalized comparison.

**6. Перевірити результати.**

Після виконання повинні існувати таблиці `nyc_taxi_raw`, `t7_hw_taxi_daily`, `t7_hw_events`, `t7_hw_checkouts_norm` та VIEW `v_t7_hw_taxi_features`.

Результати `EXPLAIN (ANALYZE, BUFFERS)` мають бути збережені у notebook.

## 9. Conclusion

У роботі продемонстровано практичне використання PostgreSQL для ML Data Engineering.

Реалізовано генерацію часових ознак, роботу з JSONB, створення індексів і порівняння різних моделей зберігання даних.

Експерименти показали, що правильний вибір індексів та структури таблиць може суттєво впливати на продуктивність SQL-запитів.