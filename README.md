# Homework #2: Feature Store для Movie Recommendation
---

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Dimaxdm/goit-rdb-hw-02/blob/main/hw02_movie_recommendation.ipynb)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)
![Dataset](https://img.shields.io/badge/dataset-MovieLens%20100K-orange)

Домашнє завдання №2 курсу **Neoversity: Relational Databases for Data Science**.

Проєктування реляційної схеми для ML-сценарію **Movie Recommendation**: бізнесові сутності MovieLens, offline feature store
з історією ознак (`valid_from` / `valid_to`), training events з `prediction_ts`, online feature store і **point-in-time коректне**
формування навчальних даних через AS-OF JOIN.

---

## Зміст

- [ML-задача](#ml-задача)
- [Дані](#дані)
- [ER-діаграма](#er-діаграма)
- [Схема БД](#схема-бд)
- [Point-in-Time correctness](#point-in-time-correctness)
- [Результати перевірок](#результати-перевірок)
- [Структура репозиторію](#структура-репозиторію)
- [Запуск](#запуск)
- [Технології](#технології)
- [Ліцензія та джерела](#ліцензія-та-джерела)

## ML-задача

У момент взаємодії користувача з фільмом (`prediction_ts`) передбачити, чи сподобається йому фільм:

```
is_liked = (rating >= 4)
```

Ознаки користувача беруться з offline feature store **станом на `prediction_ts`**, без жодної інформації з майбутнього.

## Дані

[MovieLens 100K](https://grouplens.org/datasets/movielens/100k/) використовується як **доменний референс**: схема повторює
його сутності (`u.user`, `u.item`, `u.genre`, `u.data`). Мета роботи — спроєктувати схему й показати коректне формування
навчальних даних, а не обробити великий обсяг, тому в БД вставляється невелика **детермінована вибірка реальних рядків**:

| Що | Обсяг |
|---|---|
| Користувачі | 3 |
| Оцінки (перша оцінка в кожному місяці активності користувача, оригінальні unix-мітки → `TIMESTAMPTZ` UTC) | 17 |
| Фільми (ті, що оцінені у вибірці) | 17 |
| Жанри (повний довідник) | 19 |

Ідентифікатори користувачів і фільмів у вибірці перенумеровано (`1..3`, `1..17`). Дані вбудовані в notebook як `INSERT`,
тому завантаження датасету з мережі не потрібне. `MovieLens 25M` не використовується.

## ER-діаграма

![ER-діаграма](er_diagram.gif)

Текстове джерело діаграми — [`er_diagram.dbml`](er_diagram.dbml) (відкривається на [dbdiagram.io](https://dbdiagram.io)).

| Батьківська таблиця | Дочірня таблиця | Кардинальність | Примітка |
|---|---|---|---|
| `users` | `ratings` | 1 : 0..N | FK, частина PK |
| `movies` | `ratings` | 1 : 0..N | FK, частина PK |
| `users` ↔ `movies` | через `ratings` | **N : M** | bridge-таблиця з атрибутами `rating`, `rated_at` |
| `movies` | `movie_genres` | 1 : 1..N | кожен фільм має ≥ 1 жанр |
| `genres` | `movie_genres` | 1 : 0..N | movies ↔ genres — другий природний N:M |
| `users` | `user_features_offline` | 1 : 0..N | історія версій (SCD-2) |
| `genres` | `user_features_offline` | 0..1 : 0..N | `favorite_genre_id` — необов'язковий FK |
| `users` / `movies` | `training_events` | 1 : 0..N | обидва FK `NOT NULL` |
| `training_events` | `user_features_offline` | N : 0..1 | логічний AS-OF JOIN за `prediction_ts` |
| `users` | `user_features_online` | 1 : 0..1 | PK = FK `user_id` |

## Схема БД

| Шар | Таблиця | Ключ | Призначення |
|---|---|---|---|
| Business (3НФ) | `users` | `user_id` | профіль: `age`, `gender`, `occupation`, `zip_code` |
| | `movies` | `movie_id` | `title`, `release_year` |
| | `genres` | `genre_id` | довідник жанрів, `genre_name` UNIQUE |
| | `ratings` | (`user_id`, `movie_id`) | bridge users N:M movies, `rating` 1..5, `rated_at` |
| | `movie_genres` | (`movie_id`, `genre_id`) | bridge movies N:M genres |
| Offline feature store | `user_features_offline` | (`user_id`, `valid_from`) | версії ознак на `[valid_from, valid_to)` |
| | `training_events` | `event_id` | `prediction_ts`, `target_rating`, target `is_liked` |
| Online feature store | `user_features_online` | `user_id` | денормалізований поточний рядок для lookup без `JOIN` |

Ознаки користувача: `ratings_cnt_total`, `ratings_cnt_30d`, `avg_rating`, `favorite_genre_id`.

Цілісність на рівні БД:

- `PRIMARY KEY` у кожній таблиці, `FOREIGN KEY` для всіх зв'язків, `NOT NULL` для обов'язкових колонок;
- доменні `CHECK`: рейтинг 1..5, вік 1..120, стать `M`/`F`, лічильники ≥ 0, `avg_rating` 1..5, `is_liked = (target_rating >= 4)`;
- temporal `CHECK (valid_to IS NULL OR valid_to > valid_from)`;
- тригер `trg_no_overlap_user_features` забороняє перекривні версії для одного `user_id`;
- індекси під access patterns: `(user_id, valid_from DESC)` для AS-OF lookup, `(user_id, rated_at)`, `(movie_id)`, `(user_id, prediction_ts)`.

## Point-in-Time correctness

Версія ознак чинна на напіввідкритому інтервалі `[valid_from, valid_to)`; `valid_to = NULL` означає поточну версію.
Training data формується AS-OF JOIN-ом:

```sql
SELECT
    e.*,
    f.ratings_cnt_total,
    f.ratings_cnt_30d,
    f.avg_rating, f.favorite_genre_id
FROM
    training_events AS e
LEFT JOIN
    user_features_offline AS f
        ON  f.user_id = e.user_id
            AND f.valid_from <= e.prediction_ts
            AND (f.valid_to > e.prediction_ts OR f.valid_to IS NULL);
```

У notebook є:

- 18 історичних версій ознак для 3 `entity_id` (7 / 8 / 3), явні UTC-літерали з `+00`;
- перерахунок кожної версії лише з оцінок `rated_at < valid_from` як доказ відсутності leakage під час побудови ознак;
- 4 AS-OF `SELECT`, зокрема запит рівно на межі двох версій;
- AS-OF JOIN для training events і `VIEW v_training_dataset`;
- перевірка перекривних інтервалів, зведені перевірки цілісності та негативні тести;
- порівняння з наївним `JOIN` лише за `user_id`;
- матеріалізація online-таблиці з offline через `INSERT … ON CONFLICT DO UPDATE` і перевірка узгодженості.

## Результати перевірок

| Перевірка | Результат |
|---|---|
| Розбіжності між збереженими та перерахованими версіями ознак | **0** з 18 |
| AS-OF JOIN: рядків / подій | **15 / 15**, по одному рядку на подію |
| Ознаки з майбутнього (`valid_from > prediction_ts`) | **0** |
| Cold-start події (ознак на `prediction_ts` ще немає → `NULL`) | 1 |
| Перекривні інтервали | **0 рядків** |
| Наївний `JOIN` лише за `user_id` | 100 рядків замість 15, з них **51** з майбутніми ознаками |
| Вставка `valid_to <= valid_from` | відхилена `CHECK` |
| Вставка перекривної версії | відхилена тригером |
| Розбіжності online ↔ offline | **0** |

## Структура репозиторію

```
goit-rdb-hw-02/
├── hw02_movie_recommendation.ipynb   # рішення: ER, DDL, вибірка, нормалізація, PiT, Reflection
├── er_diagram.dbml                   # ER-діаграма у форматі DBML (dbdiagram.io)
├── er-diagrame.gif                   # ER-діаграма як зображення
├── requirements.txt                  # Python-залежності
├── .gitignore
└── README.md
```

Розділи notebook відповідають завданням ДЗ:

| Завдання | Розділ notebook |
|---|---|
| 1 | ER-діаграма, зв'язки й кардинальності |
| 2 | DDL у PostgreSQL 16, перевірка обмежень та індексів через системний каталог |
| — | Детермінована вибірка MovieLens 100K |
| 3 | Нормалізація: 3НФ, функціональні залежності, денормалізація online-шару |
| 4 | Point-in-Time correctness |
| 5 | Reflection |

## Запуск

### Google Colab (рекомендовано)

1. Відкрийте notebook за бейджем **Open in Colab**.
2. `Runtime → Restart session and run all`.

Перша клітинка сама встановлює й запускає PostgreSQL 16 (`apt-get`, `service postgresql start`) та задає пароль `postgres`.
Усі SQL-клітинки працюють з однією базою `postgres` через `%%sql`. Повторний запуск відтворюваний завдяки
`DROP TABLE IF EXISTS … CASCADE` на початку DDL.

### Локально

Потрібні Python 3.10+ і PostgreSQL 16, доступний на `localhost:5432` з користувачем `postgres` і паролем `postgres`.
Найпростіше підняти базу в Docker:

```bash
docker run -d --name hw02-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  postgres:16
```

Далі:

```bash
git clone https://github.com/Dimaxdm/goit-rdb-hw-02.git
cd goit-rdb-hw-02
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter lab hw02_movie_recommendation.ipynb
```

Команди `!apt-get`, `!service` і `!sudo` у першій клітинці призначені для Colab; локально їх можна пропустити, якщо база вже запущена.

## Технології

- PostgreSQL 16
- Python: SQLAlchemy 2, psycopg2, pandas
- `ipython-sql` (`%%sql`) з PrettyTable для виводу результатів
- Google Colab / JupyterLab
- dbdiagram.io (DBML) для ER-діаграми

## Ліцензія та джерела

Навчальний проєкт. Дані: F. Maxwell Harper and Joseph A. Konstan. 2015.
*The MovieLens Datasets: History and Context.* ACM Transactions on Interactive Intelligent Systems 5(4).
Використання відповідає [умовам GroupLens](https://files.grouplens.org/datasets/movielens/ml-100k-README.txt) (некомерційне, з посиланням на джерело).
