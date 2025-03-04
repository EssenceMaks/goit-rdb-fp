# **Тема Фінальний проект **
# **(SQL імпорт, нормалізація та додаткові функції)**

## **p1**

### Створення схеми та імпорт даних
- Створення схеми та таблиці: Створює схему `pandemic` та таблицю `infectious_cases` з необхідними полями.
- Імпорт через wizard провалився, тому імпорт був виконаний за допомогою команди.
- Видаляє таблицю `infectious_cases`, якщо вона існує.
- Створює нову таблицю `infectious_cases` з необхідними полями.
- Перевірка шляху для імпорту: Показує дозволену директорію для імпорту файлів.
- Імпорт даних: Завантажує дані з CSV-файлу з обробкою пустих значень для полів `polio_cases` та `cases_guinea_worm`.

![p1](/p1.png)

```sql
CREATE SCHEMA pandemic;
USE pandemic;

DROP TABLE IF EXISTS pandemic.infectious_cases;

CREATE TABLE pandemic.infectious_cases (
    id INT AUTO_INCREMENT PRIMARY KEY,
    Entity TEXT,
    Code TEXT,
    Year INT,
    Number_yaws TEXT,
    polio_cases INT,
    cases_guinea_worm INT,
    Number_rabies TEXT,
    Number_malaria TEXT,
    Number_hiv TEXT,
    Number_tuberculosis TEXT,
    Number_smallpox TEXT,
    Number_cholera_cases TEXT
);

SHOW VARIABLES LIKE 'secure_file_priv';

LOAD DATA INFILE 'C:\\ProgramData\\MySQL\\MySQL Server 8.4\\Uploads\\infectious_cases.csv'
INTO TABLE pandemic.infectious_cases
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(Entity, Code, Year, Number_yaws, @polio_cases, @cases_guinea_worm, Number_rabies, Number_malaria, Number_hiv, Number_tuberculosis, Number_smallpox, Number_cholera_cases)
SET
    polio_cases = CAST(NULLIF(TRIM(@polio_cases), '') AS SIGNED),
    cases_guinea_worm = CAST(NULLIF(TRIM(@cases_guinea_worm), '') AS SIGNED);
```

![p1](/p1_1.png)

---

## **p2**

### Нормалізація таблиці
- Створення таблиці країн: Створює окрему таблицю `countries` для зберігання унікальних комбінацій `Entity` та `Code`.
- Заповнення таблиці країн: Додає унікальні значення з основної таблиці.
- Додавання зв'язку: Створює поле `country_id` в основній таблиці.
- Оновлення зв'язків: Встановлює відповідні значення `country_id` для кожного запису.
- Видалення надлишкових даних: Видаляє поля `Entity` та `Code` з основної таблиці.
- Створення зовнішнього ключа: Додає обмеження для забезпечення цілісності даних.

![p2](/p2.png)

```sql
INSERT INTO countries (entity, code)
SELECT DISTINCT entity, code
FROM infectious_cases;

CREATE TABLE pandemic.countries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    Entity TEXT,
    Code TEXT
);

INSERT INTO pandemic.countries (Entity, Code)
SELECT DISTINCT Entity, Code FROM pandemic.infectious_cases;

ALTER TABLE pandemic.infectious_cases
ADD COLUMN country_id INT;

UPDATE pandemic.infectious_cases ic
JOIN pandemic.countries c ON ic.Entity = c.Entity AND ic.Code = c.Code
SET ic.country_id = c.id;

SET SQL_SAFE_UPDATES = 0;

UPDATE pandemic.infectious_cases ic
JOIN pandemic.countries c ON ic.Entity = c.Entity AND ic.Code = c.Code
SET ic.country_id = c.id;

SET SQL_SAFE_UPDATES = 1;

ALTER TABLE pandemic.infectious_cases
DROP COLUMN Entity,
DROP COLUMN Code;

ALTER TABLE pandemic.infectious_cases
ADD CONSTRAINT fk_country
FOREIGN KEY (country_id) REFERENCES pandemic.countries(id);
```

![p2](/p2_1.png)

---

## **p3**

### Аналіз даних про сказ
Запит обчислює середнє, мінімальне, максимальне значення та суму для атрибута `Number_rabies` для кожної країни, фільтрує пусті значення, сортує за середнім значенням у порядку спадання та обмежує виведення 10 рядками.

![p3](/p3.png)

```sql
SELECT 
    c.id AS country_id,
    c.Entity,
    c.Code,
    AVG(CAST(ic.Number_rabies AS DECIMAL(10,2))) AS avg_rabies,
    MIN(CAST(ic.Number_rabies AS DECIMAL(10,2))) AS min_rabies,
    MAX(CAST(ic.Number_rabies AS DECIMAL(10,2))) AS max_rabies,
    SUM(CAST(ic.Number_rabies AS DECIMAL(10,2))) AS sum_rabies
FROM 
    pandemic.infectious_cases ic
JOIN 
    pandemic.countries c ON ic.country_id = c.id
WHERE 
    ic.Number_rabies IS NOT NULL 
    AND ic.Number_rabies != ''
GROUP BY 
    c.id, c.Entity, c.Code
ORDER BY 
    avg_rabies DESC
LIMIT 10;
```

---

## **p4**

### Обчислення різниці в роках
Запит створює дату першого січня відповідного року, отримує поточну дату та обчислює різницю в роках між ними для кожного запису.

![p4](/p4.png)

```sql
SELECT
    Year,
    DATE_FORMAT(STR_TO_DATE(CONCAT(Year, '-01-01'), '%Y-%m-%d'), '%Y-%m-%d') AS first_day_of_year,
    CURDATE() AS `current_date`,
    TIMESTAMPDIFF(YEAR, 
                 STR_TO_DATE(CONCAT(Year, '-01-01'), '%Y-%m-%d'),
                 CURDATE()) AS years_difference
FROM
    infectious_cases;
```

---

## **p5**

### Створення та використання функції
Створює функцію `calculate_year_diff`, яка приймає рік і повертає різницю в роках між першим січня цього року та поточною датою, та використовує цю функцію для обчислення різниці для кожного запису.

![p5](/p5.png)

```sql
DELIMITER //
CREATE FUNCTION calculate_year_diff(input_year INT)
RETURNS INT
DETERMINISTIC 
NO SQL
BEGIN
    DECLARE result INT;
    SET result = TIMESTAMPDIFF(YEAR, MAKEDATE(input_year, 1), CURDATE());
    RETURN result;
END //
DELIMITER ;

SELECT
    Year,
    MAKEDATE(Year, 1) AS new_date,
    CURDATE() AS today,
    calculate_year_diff(Year) AS difference
FROM infectious_cases;
