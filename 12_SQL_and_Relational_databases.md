---

# SQL & Relational Databases — Senior Developer Edition

## Overview

Practical, production-ready SQL patterns with small, composable CTEs. Examples aim to be ANSI-SQL friendly; PostgreSQL notes appear where helpful.

---

## Example Schema

```sql
-- Classes and students
CREATE TABLE classes   (id BIGINT PRIMARY KEY, name TEXT);
CREATE TABLE students  (id BIGINT PRIMARY KEY, name TEXT, class_id BIGINT, grade NUMERIC,
                        FOREIGN KEY (class_id) REFERENCES classes(id));

-- Countries, cities, and people
CREATE TABLE countries (id BIGINT PRIMARY KEY, name TEXT);
CREATE TABLE cities    (id BIGINT PRIMARY KEY, name TEXT, country_id BIGINT,
                        FOREIGN KEY (country_id) REFERENCES countries(id));
CREATE TABLE people    (id BIGINT PRIMARY KEY, name TEXT, city_id BIGINT, age INT,
                        FOREIGN KEY (city_id) REFERENCES cities(id));
```

> Tip: Favor **surrogate keys** for joins, FKs for integrity, and **NOT NULL** where appropriate. Normalize until it hurts; denormalize where it helps reads.

---

## 1) Top grade per class (handles ties)

### a) Window functions (preferred for clarity)

```sql
SELECT s.*
FROM (
  SELECT s.*,
         RANK() OVER (PARTITION BY s.class_id ORDER BY s.grade DESC) AS rnk
  FROM students s
) s
WHERE s.rnk = 1;
```

### b) Join with MAX

```sql
SELECT s.*
FROM students s
JOIN (
  SELECT class_id, MAX(grade) AS max_grade
  FROM students
  GROUP BY class_id
) mx ON mx.class_id = s.class_id AND mx.max_grade = s.grade;
```

---

## 2) People living in the city (per country) with the **lowest average age**

Steps: (1) average age per city, (2) country-level minimum, (3) people in those cities.

```sql
WITH city_avg AS (
  SELECT c.id AS city_id, c.country_id, AVG(p.age)::NUMERIC AS avg_age
  FROM cities c
  JOIN people p ON p.city_id = c.id
  GROUP BY c.id, c.country_id
),
country_min AS (
  SELECT country_id, MIN(avg_age) AS min_avg_age
  FROM city_avg
  GROUP BY country_id
),
target_cities AS (
  SELECT ca.city_id, ca.country_id
  FROM city_avg ca
  JOIN country_min cm
    ON cm.country_id = ca.country_id
   AND cm.min_avg_age = ca.avg_age
)
SELECT co.name AS country_name,
       ci.name AS city_name,
       pe.id, pe.name, pe.age
FROM target_cities tc
JOIN cities    ci ON ci.id = tc.city_id
JOIN countries co ON co.id = tc.country_id
JOIN people   pe ON pe.city_id = tc.city_id
ORDER BY co.name, ci.name, pe.name;
```

*Note:* Using `JOIN people` excludes cities with no residents; swap to `LEFT JOIN` to include them with NULL averages.

---

## 3) Students above their class average

```sql
WITH class_avg AS (
  SELECT class_id, AVG(grade) AS avg_grade
  FROM students
  GROUP BY class_id
)
SELECT s.*, ca.avg_grade
FROM students s
JOIN class_avg ca ON ca.class_id = s.class_id
WHERE s.grade > ca.avg_grade
ORDER BY s.class_id, s.grade DESC;
```

---

## 4) For each country, the city with the **lowest average age**

(Return only city and its average.)

```sql
WITH city_avg AS (
  SELECT c.country_id, c.id AS city_id, c.name AS city_name, AVG(p.age) AS avg_age
  FROM cities c
  JOIN people p ON p.city_id = c.id
  GROUP BY c.country_id, c.id, c.name
),
picked AS (
  SELECT *,
         RANK() OVER (PARTITION BY country_id ORDER BY avg_age ASC) AS rnk
  FROM city_avg
)
SELECT co.name AS country_name, city_name, avg_age
FROM picked pk
JOIN countries co ON co.id = pk.country_id
WHERE pk.rnk = 1
ORDER BY co.name, city_name;
```

---

## 5) Top 3 students in each class

```sql
SELECT *
FROM (
  SELECT s.*,
         DENSE_RANK() OVER (PARTITION BY s.class_id ORDER BY s.grade DESC) AS rnk
  FROM students s
) t
WHERE t.rnk <= 3
ORDER BY t.class_id, t.rnk, t.grade DESC;
```

---

## 6) Median age per city (PostgreSQL)

```sql
SELECT c.name AS city_name,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY p.age) AS median_age
FROM cities c
JOIN people p ON p.city_id = c.id
GROUP BY c.name
ORDER BY city_name;
```

---

## 7) People in cities with **average age < 25**

```sql
WITH city_avg AS (
  SELECT c.id AS city_id, AVG(p.age) AS avg_age
  FROM cities c
  JOIN people p ON p.city_id = c.id
  GROUP BY c.id
)
SELECT ci.name AS city, pe.*
FROM city_avg ca
JOIN cities ci ON ci.id = ca.city_id
JOIN people pe ON pe.city_id = ca.city_id
WHERE ca.avg_age < 25
ORDER BY city, pe.name;
```

---

## 8) Grade distribution per class (percentiles, PostgreSQL)

```sql
SELECT class_id,
       PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY grade) AS p90,
       PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY grade) AS p75,
       PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY grade) AS median,
       PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY grade) AS p25
FROM students
GROUP BY class_id
ORDER BY class_id;
```

---

## 9) Top 2 most populous cities per country

```sql
WITH city_pop AS (
  SELECT c.id AS city_id, c.country_id, COUNT(p.id) AS population
  FROM cities c
  LEFT JOIN people p ON p.city_id = c.id
  GROUP BY c.id, c.country_id
),
ranked AS (
  SELECT cp.*,
         DENSE_RANK() OVER (PARTITION BY cp.country_id ORDER BY cp.population DESC) AS rnk
  FROM city_pop cp
)
SELECT co.name AS country, ci.name AS city, population
FROM ranked r
JOIN countries co ON co.id = r.country_id
JOIN cities    ci ON ci.id = r.city_id
WHERE r.rnk <= 2
ORDER BY country, r.rnk, population DESC, city;
```

---

## 10) People whose age is **≥ 20% above** their city’s average

```sql
WITH city_avg AS (
  SELECT c.id AS city_id, AVG(p.age) AS avg_age
  FROM cities c
  JOIN people p ON p.city_id = c.id
  GROUP BY c.id
)
SELECT pe.*, ci.name AS city, ca.avg_age
FROM people pe
JOIN cities ci ON ci.id = pe.city_id
JOIN city_avg ca ON ca.city_id = pe.city_id
WHERE pe.age >= ca.avg_age * 1.20
ORDER BY city, pe.age DESC;
```

---

## Performance Notes

* Useful indexes:

```sql
CREATE INDEX idx_students_class ON students(class_id, grade DESC);
CREATE INDEX idx_people_city    ON people(city_id, age);
CREATE INDEX idx_cities_country ON cities(country_id);
```

* Consider **materialized views** for frequently reused aggregates (e.g., city averages).
* Prefer **keyset pagination** for large result sets; align `ORDER BY` with index order.
* Always verify execution plans (`EXPLAIN` / `EXPLAIN ANALYZE`).

---

## Quick Interview Nuggets

* **Window vs GROUP BY**: Windows keep row detail + aggregate; GROUP BY collapses rows.
* **RANK vs DENSE_RANK vs ROW_NUMBER**: Ties → gaps with RANK, no gaps with DENSE_RANK, none with ROW_NUMBER.
* **NULLs in aggregates**: `AVG()` ignores NULL; use `COALESCE` for explicit defaults.
* **JOIN vs EXISTS**: `EXISTS` can short-circuit; often faster for semi-joins.
* **Cardinality & stats** drive plans — keep **ANALYZE** / auto-stats on.

---

If you want, I can try saving this as a downloadable file again, or split it into smaller topic files for your repo structure.
