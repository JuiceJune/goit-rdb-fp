# goit-rdb-fp: фінальний проєкт «Реляційні бази даних»

Фінальний проєкт курсу «Реляційні бази даних» (GoIT). Датасет `infectious_cases` — статистика інфекційних захворювань за країнами/регіонами та роками. Робота виконана в **PostgreSQL 16**, який запускається локально з Python через `pgserver`; SQL виконується з notebook через SQLAlchemy.

Уся робота — у notebook [`FP.ipynb`](FP.ipynb) (з output-ами всіх клітинок, описом кожного кроку та висновками до кожного завдання).

## Дані

- Файл: [`infectious_cases.csv`](infectious_cases.csv) (додано до репозиторію, щоб notebook відтворювався без зовнішніх завантажень).
- 10 521 рядок, 12 колонок: `Entity`, `Code`, `Year` і 9 показників (`Number_rabies`, `Number_malaria`, `Number_hiv`, `Number_tuberculosis`, `polio_cases`, `Number_cholera_cases` тощо).
- 245 країн/регіонів, роки 1920–2023. Серед `Entity` є агрегати (`World`, `Africa`, `G20`, регіони ВООЗ і Світового банку); у більшості з них `Code` порожній.
- Порожні значення показників імпортовано як `NULL`; `Number_rabies` заповнено в 6 780 рядках (дані про сказ є за 1990–2019 роки).

## Виконання

### Завдання 1. Завантаження даних

- `CREATE SCHEMA IF NOT EXISTS pandemic;` і `SET search_path TO pandemic;` — схема `pandemic` як схема за замовчуванням.
- Таблиця `infectious_cases` (назви колонок малими літерами; `year INTEGER`, показники `DOUBLE PRECISION`, бо частина з них дробові).
- Імпорт CSV через `COPY ... FROM STDIN WITH (FORMAT csv, HEADER true)` (`cursor.copy_expert` psycopg2) — завантажено 10 521 рядок.
- Огляд: перші рядки, кількість рядків і NULL, унікальні значення, рядки без коду країни.

### Завдання 2. Нормалізація до 3НФ

У вихідній таблиці `code` залежить лише від `entity` (частини ключа `(entity, year)`), тому пара `entity, code` дублюється в кожному рядку. Після нормалізації:

| Таблиця | Вміст | Ключі |
|---|---|---|
| `entities` | 245 унікальних комбінацій `entity, code` | `id SERIAL PRIMARY KEY`, `UNIQUE NULLS NOT DISTINCT (entity, code)` |
| `disease_cases` | 10 521 рядок показників з `entity_id` замість `entity` і `code` | `id SERIAL PRIMARY KEY`, `entity_id REFERENCES entities(id)`, `UNIQUE (entity_id, year)` |

`disease_cases` заповнюється через `JOIN` з `entities` по `entity` і `code IS NOT DISTINCT FROM` — інакше рядки регіонів із `code = NULL` (1 128 шт.) було б втрачено. Кількість рядків і заповнених значень `number_rabies` збігається з оригіналом.

### Завдання 3. Аналіз `number_rabies`

`AVG`, `MIN`, `MAX`, `SUM` для кожної комбінації `entity, code` (`GROUP BY e.id, e.entity, e.code`), лише для рядків `number_rabies IS NOT NULL`, `ORDER BY avg_rabies DESC LIMIT 10`.

Фільтр важливий: 19 країн/регіонів не мають жодного значення `number_rabies`, їхнє `AVG` дорівнює NULL, а PostgreSQL під час сортування `DESC` ставить NULL першими. Лідери — `World` (середнє ≈ 20 192), регіональні агрегати й India (6-те місце, ≈ 8 599).

### Завдання 4. Колонка різниці в роках

Для `year` побудовано вбудованими функціями:

- `MAKE_DATE(year, 1, 1)` — 1 січня відповідного року;
- `CURRENT_DATE` — поточна дата;
- `EXTRACT(YEAR FROM AGE(CURRENT_DATE, MAKE_DATE(year, 1, 1)))::INTEGER` — різниця в повних роках.

### Завдання 5. Власні функції

| Функція | Що робить | Волатильність |
|---|---|---|
| `fn_years_since(p_year INTEGER) RETURNS INTEGER` | різниця в роках між поточною датою та 1 січня року (1996 → 30 на 2026-11-07) | `STABLE` (залежить від `CURRENT_DATE`) |
| `fn_avg_cases_per_period(p_cases_per_year DOUBLE PRECISION, p_divider DOUBLE PRECISION) RETURNS DOUBLE PRECISION` | середня кількість захворювань за період: `p_cases_per_year / NULLIF(p_divider, 0)` | `IMMUTABLE` |

Обидві функції використано в запитах: `fn_years_since(year)` порівняно з колонкою із завдання 4 (значення збігаються); `fn_avg_cases_per_period(number_rabies, 12 / 4 / 2)` — середнє на місяць, квартал і півріччя лише для рядків з числовим `number_rabies`. Ділення на 0 повертає NULL замість помилки.

> Результати завдань 4 і 5, що залежать від `CURRENT_DATE`, у notebook пораховано на **2026-11-07**; при запуску в інший день вони відповідно зміняться.

## Скриншоти

Запити та їхні результати з notebook (папка [`screenshots/`](screenshots/)):

| Завдання | Скриншот | Що показано |
|---|---|---|
| 1 | [01_schema.png](screenshots/01_schema.png) | `CREATE SCHEMA pandemic`, `SET search_path`, `current_schema()` |
| 1 | [02_create_table.png](screenshots/02_create_table.png) | `CREATE TABLE infectious_cases` |
| 1 | [03_import.png](screenshots/03_import.png) | імпорт CSV через `COPY` — 10 521 рядок |
| 1 | [04_overview.png](screenshots/04_overview.png) | огляд даних |
| 2 | [05_entities.png](screenshots/05_entities.png) | таблиця `entities` (унікальні `entity, code`) |
| 2 | [06_disease_cases.png](screenshots/06_disease_cases.png) | таблиця `disease_cases` з `entity_id` |
| 2 | [07_normalization_check.png](screenshots/07_normalization_check.png) | перевірка нормалізації без втрат |
| 3 | [08_rabies_top10.png](screenshots/08_rabies_top10.png) | `AVG` / `MIN` / `MAX` / `SUM` для `number_rabies`, топ-10 |
| 4 | [09_years_diff.png](screenshots/09_years_diff.png) | дата 1 січня, поточна дата, різниця в роках |
| 5 | [10_fn_years_since_1.png](screenshots/10_fn_years_since_1.png) | створення `fn_years_since`, приклад 1996 → 30 |
| 5 | [10_fn_years_since_2.png](screenshots/10_fn_years_since_2.png) | використання `fn_years_since` і порівняння із завданням 4 |
| 5 | [11_fn_avg_cases_1.png](screenshots/11_fn_avg_cases_1.png) | створення `fn_avg_cases_per_period` і перевірка (12, 0, NULL) |
| 5 | [11_fn_avg_cases_2.png](screenshots/11_fn_avg_cases_2.png) | використання з дільниками 12 / 4 / 2 |

## Як запустити

Потрібен Python 3.9–3.12 (`pgserver` 0.1.4 має wheels лише для цих версій).

```bash
uv venv --python 3.12
uv pip install pgserver psycopg2-binary sqlalchemy pandas ipykernel
```

Далі відкрити `FP.ipynb`, вибрати kernel `.venv` і виконати *Restart & Run All*. База PostgreSQL створюється в новому тимчасовому каталозі й видаляється після зупинки kernel, тому весь результат відтворюється кодом notebook.
