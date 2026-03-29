# Конспект по SQL

## 📚 Оглавление

1. [Основы SQL](#1-основы-sql)
   - [Что такое SQL?](#что-такое-sql)
   - [Типы команд SQL](#типы-команд-sql)
2. [Работа с таблицами (DDL)](#2-работа-с-таблицами-ddl)
3. [Базовые запросы (SELECT)](#3-базовые-запросы-select)
4. [Операторы WHERE (условия)](#4-операторы-where-условия)
5. [Вставка, обновление, удаление (DML)](#5-вставка-обновление-удаление-dml)
6. [Агрегатные функции](#6-агрегатные-функции)
7. [Группировка (GROUP BY) и фильтрация групп (HAVING)](#7-группировка-group-by-и-фильтрация-групп-having)
8. [Сортировка и уникальные значения](#8-сортировка-и-уникальные-значения)
9. [Объединение таблиц (JOIN)](#9-объединение-таблиц-join)
   - [Типы JOIN визуально](#типы-join-визуально)
10. [Подзапросы (Subqueries)](#10-подзапросы-subqueries)
11. [CTE (Common Table Expressions)](#11-cte-common-table-expressions)
12. [Индексы (для производительности)](#12-индексы-для-производительности)
13. [Транзакции (TCL)](#13-транзакции-tcl)
14. [Краткая шпаргалка](#14-краткая-шпаргалка)
15. [Полезные сокращения и стиль](#15-полезные-сокращения-и-стиль)
16. [Пример для практики](#16-пример-для-практики)

## 1. Основы SQL

### Что такое SQL?
SQL (Structured Query Language) — язык структурированных запросов для работы с реляционными базами данных.

### Типы команд SQL

| Тип | Расшифровка | Что делает | Примеры команд |
|-----|-------------|------------|----------------|
| DDL | Data Definition Language | Определяет структуру БД | `CREATE`, `ALTER`, `DROP` |
| DML | Data Manipulation Language | Работает с данными | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| DCL | Data Control Language | Управляет правами доступа | `GRANT`, `REVOKE` |
| TCL | Transaction Control Language | Управляет транзакциями | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

---

## 2. Работа с таблицами (DDL)

```sql
-- Создание таблицы
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    age INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Удаление таблицы
DROP TABLE users;

-- Изменение структуры таблицы
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users MODIFY COLUMN name VARCHAR(150);
```

---

## 3. Базовые запросы (SELECT)

```sql
-- Выбрать всё из таблицы
SELECT * FROM users;

-- Выбрать конкретные столбцы
SELECT name, email FROM users;

-- Выбрать с условием
SELECT * FROM users WHERE age > 18;

-- Выбрать с сортировкой
SELECT * FROM users ORDER BY name ASC;   -- ASC — по возрастанию
SELECT * FROM users ORDER BY created_at DESC; -- DESC — по убыванию

-- Ограничить количество записей
SELECT * FROM users LIMIT 10;

-- Пропустить первые N записей
SELECT * FROM users LIMIT 10 OFFSET 5;
```

---

## 4. Операторы WHERE (условия)

```sql
-- Равенство и неравенство
WHERE id = 5
WHERE name = 'Иван'
WHERE age != 18
WHERE age <> 18   -- то же самое

-- Сравнение
WHERE age > 18
WHERE age >= 18
WHERE age BETWEEN 18 AND 30

-- Несколько условий
WHERE age > 18 AND city = 'Москва'
WHERE age < 18 OR city = 'СПб'

-- Проверка на NULL
WHERE email IS NULL
WHERE phone IS NOT NULL

-- Поиск по шаблону (LIKE)
WHERE name LIKE 'А%'    -- начинается с А
WHERE name LIKE '%ов'   -- заканчивается на "ов"
WHERE name LIKE '%а%'   -- содержит "а"

-- Проверка на вхождение в список
WHERE city IN ('Москва', 'СПб', 'Казань')
```

---

## 5. Вставка, обновление, удаление (DML)

```sql
-- Вставка одной записи
INSERT INTO users (name, email, age) 
VALUES ('Иван Петров', 'ivan@example.com', 25);

-- Вставка нескольких записей
INSERT INTO users (name, email, age) VALUES 
('Анна Иванова', 'anna@example.com', 30),
('Петр Сидоров', 'petr@example.com', 28);

-- Обновление данных
UPDATE users 
SET age = 26 
WHERE name = 'Иван Петров';

-- Удаление данных
DELETE FROM users WHERE id = 5;

-- Удалить все записи (быстро)
TRUNCATE TABLE users;
```

---

## 6. Агрегатные функции

```sql
-- Подсчет количества записей
SELECT COUNT(*) FROM users;
SELECT COUNT(email) FROM users;  -- не считает NULL
SELECT COUNT(DISTINCT city) FROM users;

-- Сумма, среднее, минимум, максимум
SELECT SUM(age) FROM users;
SELECT AVG(age) FROM users;
SELECT MIN(age) FROM users;
SELECT MAX(age) FROM users;

-- Комбинация с WHERE
SELECT AVG(age) FROM users WHERE city = 'Москва';
```

---

## 7. Группировка (GROUP BY) и фильтрация групп (HAVING)

```sql
-- Сгруппировать по городу и посчитать количество
SELECT city, COUNT(*) as user_count 
FROM users 
GROUP BY city;

-- Сгруппировать с условием на группу
SELECT city, AVG(age) as avg_age 
FROM users 
GROUP BY city 
HAVING AVG(age) > 25;
```

**Разница WHERE и HAVING:**
- `WHERE` — фильтрует строки **ДО** группировки
- `HAVING` — фильтрует группы **ПОСЛЕ** группировки

---

## 8. Сортировка и уникальные значения

```sql
-- Уникальные значения
SELECT DISTINCT city FROM users;

-- Сортировка по нескольким полям
SELECT * FROM users 
ORDER BY city ASC, name DESC;
```

---

## 9. Объединение таблиц (JOIN)

```sql
-- Создадим вторую таблицу для примеров
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    product VARCHAR(100),
    amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- INNER JOIN — только совпадающие записи
SELECT users.name, orders.product, orders.amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- LEFT JOIN — все пользователи + их заказы (если есть)
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- RIGHT JOIN — все заказы + информация о пользователях
SELECT users.name, orders.product
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;

-- FULL OUTER JOIN — все записи из обеих таблиц
-- (не поддерживается в MySQL, но есть в PostgreSQL)
```

### Типы JOIN визуально

| Тип JOIN | Результат |
|----------|-----------|
| `INNER JOIN` | Только записи, совпадающие в обеих таблицах |
| `LEFT JOIN` | Все записи из левой таблицы + совпадения из правой |
| `RIGHT JOIN` | Все записи из правой таблицы + совпадения из левой |
| `FULL OUTER JOIN` | Все записи из обеих таблиц |
| `CROSS JOIN` | Декартово произведение (каждая с каждой) |

---

## 10. Подзапросы (Subqueries)

```sql
-- Подзапрос в WHERE
SELECT name, age 
FROM users 
WHERE age > (SELECT AVG(age) FROM users);

-- Подзапрос в FROM
SELECT AVG(age_by_city) 
FROM (SELECT city, AVG(age) as age_by_city 
      FROM users 
      GROUP BY city) as city_stats;

-- Подзапрос в SELECT
SELECT name, 
       (SELECT COUNT(*) FROM orders WHERE user_id = users.id) as orders_count
FROM users;
```

---

## 11. CTE (Common Table Expressions)

**CTE** (Common Table Expression) — это временный именованный результат, который существует только в рамках одного запроса. CTE делают сложные запросы более читаемыми и позволяют создавать рекурсивные запросы.

### Простой CTE

```sql
-- Базовый синтаксис
WITH cte_name AS (
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
SELECT * FROM cte_name;

-- Пример: найти отделы с зарплатой выше средней
WITH dept_avg AS (
    SELECT department, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.department, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_salary;

-- Несколько CTE в одном запросе
WITH 
high_earners AS (
    SELECT * FROM employees WHERE salary > 70000
),
dept_stats AS (
    SELECT department, COUNT(*) as emp_count
    FROM high_earners
    GROUP BY department
)
SELECT * FROM dept_stats
ORDER BY emp_count DESC;
```

---

## 12. Индексы (для производительности)

```sql
-- Создать индекс
CREATE INDEX idx_users_email ON users(email);

-- Создать уникальный индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Удалить индекс
DROP INDEX idx_users_email ON users;

-- Показать индексы таблицы
SHOW INDEX FROM users;
```

---

## 13. Транзакции (TCL)

```sql
-- Начать транзакцию
START TRANSACTION;

-- Выполнить операции
INSERT INTO users (name, email) VALUES ('Тест', 'test@example.com');
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- Если всё успешно
COMMIT;

-- Если ошибка
ROLLBACK;
```

---

## 14. Краткая шпаргалка

| Задача | Команда |
|--------|---------|
| Показать все базы данных | `SHOW DATABASES;` |
| Выбрать базу данных | `USE database_name;` |
| Показать все таблицы | `SHOW TABLES;` |
| Структура таблицы | `DESCRIBE table_name;` |

---

## 15. Полезные сокращения и стиль

```sql
-- Псевдонимы (AS)
SELECT name AS user_name FROM users;

-- Конкатенация строк (MySQL)
SELECT CONCAT(name, ' (', email, ')') AS user_info FROM users;

-- Текущая дата и время
SELECT NOW();
SELECT CURDATE();
SELECT CURTIME();

-- Приведение типов
SELECT CAST('123' AS UNSIGNED);
SELECT CONVERT('2024-01-01', DATE);
```

---

## 16. Пример для практики

```sql
-- Задача: найти топ-5 пользователей по сумме заказов
SELECT u.name, SUM(o.amount) as total_spent
FROM users u
INNER JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
ORDER BY total_spent DESC
LIMIT 5;
```