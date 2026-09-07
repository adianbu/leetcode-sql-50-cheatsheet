# LeetCode SQL 50 — Complete Cheatsheet

---

## Part 1: SQL Query Patterns Reference

---

### Pattern 1 — Pivot (Rows → Columns)
**Identify when:** Data is stored in long/tall format (one row per category) and you need to reshape it wide (one column per category).  
**Approach:** MySQL has no native PIVOT — use conditional aggregation: `MAX(CASE WHEN category = 'X' THEN value END) AS X`.  
**Watch out for:** Unknown categories at query time — you'd need dynamic SQL. For known/fixed categories, the CASE WHEN approach is clean.

```sql
-- Score table: student_id, subject, score
-- Pivot to: student_id | Math | Science | English
SELECT student_id,
  MAX(CASE WHEN subject = 'Math'    THEN score END) AS Math,
  MAX(CASE WHEN subject = 'Science' THEN score END) AS Science,
  MAX(CASE WHEN subject = 'English' THEN score END) AS English
FROM Scores
GROUP BY student_id;
```

> **Unpivot** (columns → rows) uses UNION ALL:
> ```sql
> SELECT student_id, 'Math'    AS subject, math_score    AS score FROM Wide
> UNION ALL
> SELECT student_id, 'Science' AS subject, science_score AS score FROM Wide;
> ```


**Example:**

Input `Scores`:
| student_id | subject | score |
|---|---|---|
| 1 | Math | 90 |
| 1 | Science | 85 |
| 1 | English | 78 |

Output:
| student_id | Math | Science | English |
|---|---|---|---|
| 1 | 90 | 85 | 78 |

---

### Pattern 2 — Gaps and Islands
**Identify when:** You need to find consecutive sequences (islands) and breaks in them (gaps) — e.g., consecutive login days, consecutive IDs, uninterrupted streaks.

**Core idea:** Subtract `ROW_NUMBER()` from the ordered value. Within a consecutive run, this difference is constant → group by it to identify islands. A gap is where the sequence breaks.

```sql
-- Find islands of consecutive dates per user
WITH numbered AS (
  SELECT user_id, login_date,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
  FROM Logins
),
islands AS (
  SELECT user_id, login_date,
    DATE_SUB(login_date, INTERVAL rn DAY) AS island_key  -- constant within a run
  FROM numbered
)
SELECT user_id,
  MIN(login_date) AS streak_start,
  MAX(login_date) AS streak_end,
  COUNT(*) AS streak_length
FROM islands
GROUP BY user_id, island_key
ORDER BY user_id, streak_start;
```

**Finding gaps** (missing values between min and max):
```sql
-- Find missing IDs between 1 and MAX
WITH RECURSIVE seq AS (
  SELECT 1 AS n
  UNION ALL SELECT n + 1 FROM seq WHERE n < (SELECT MAX(id) FROM T)
)
SELECT n AS missing_id
FROM seq
WHERE n NOT IN (SELECT id FROM T);
```

> **Key insight:** If values are consecutive, `value - ROW_NUMBER()` is the same for every row in that island. Any change in that difference signals a new island (i.e., a gap occurred).


**Example:**

Input `Logins`:
| user_id | login_date |
|---|---|
| 1 | 2024-01-01 |
| 1 | 2024-01-02 |
| 1 | 2024-01-03 |
| 1 | 2024-01-05 |
| 1 | 2024-01-06 |

Output:
| user_id | streak_start | streak_end | streak_length |
|---|---|---|---|
| 1 | 2024-01-01 | 2024-01-03 | 3 |
| 1 | 2024-01-05 | 2024-01-06 | 2 |

---

### Pattern 3 — Recursive CTE (Hierarchical / Sequence Generation)
**Identify when:** Data has a parent-child hierarchy (org charts, category trees, bill of materials), or you need to generate a sequence of numbers/dates.  
**Structure:** Anchor member (base case) + UNION ALL + Recursive member (references the CTE itself).

```sql
-- Traverse org hierarchy: find all reports under a manager
WITH RECURSIVE org AS (
  -- Anchor: the root manager
  SELECT employee_id, name, manager_id, 0 AS depth
  FROM Employees WHERE employee_id = 1
  UNION ALL
  -- Recursive: join each employee to their manager already in the CTE
  SELECT e.employee_id, e.name, e.manager_id, org.depth + 1
  FROM Employees e
  JOIN org ON e.manager_id = org.employee_id
)
SELECT * FROM org;
```

**Generate a date spine (all dates in a range):**
```sql
WITH RECURSIVE dates AS (
  SELECT '2024-01-01' AS dt
  UNION ALL
  SELECT DATE_ADD(dt, INTERVAL 1 DAY) FROM dates WHERE dt < '2024-01-31'
)
SELECT dt FROM dates;
```

> **Watch out for:** Infinite loops — always have a termination condition in the WHERE clause. MySQL defaults to 1000 recursion depth (`SET SESSION cte_max_recursion_depth = n` to raise it).


**Example:**

Input `Employees`:
| employee_id | name | manager_id |
|---|---|---|
| 1 | Alice | NULL |
| 2 | Bob | 1 |
| 3 | Carol | 1 |
| 4 | Dan | 2 |

Output (`org`, root = 1):
| employee_id | name | manager_id | depth |
|---|---|---|---|
| 1 | Alice | NULL | 0 |
| 2 | Bob | 1 | 1 |
| 3 | Carol | 1 | 1 |
| 4 | Dan | 2 | 2 |

---

### Pattern 4 — EXISTS / NOT EXISTS (Semi-join / Anti-semi-join)
**Identify when:** You want rows from table A where a matching row in table B does or does not exist — especially when the match involves multiple columns or NULLs could be a problem.  
**Why prefer over IN/NOT IN:** `NOT IN` fails silently when the subquery contains any NULL (returns empty result). `NOT EXISTS` handles NULLs correctly and is often faster with proper indexes.

```sql
-- Semi-join: customers who placed at least one order
SELECT c.customer_id, c.name
FROM Customers c
WHERE EXISTS (
  SELECT 1 FROM Orders o WHERE o.customer_id = c.customer_id
);

-- Anti-semi-join: customers who placed NO orders (NULL-safe)
SELECT c.customer_id, c.name
FROM Customers c
WHERE NOT EXISTS (
  SELECT 1 FROM Orders o WHERE o.customer_id = c.customer_id
);
```

> `SELECT 1` inside EXISTS is convention — the optimizer ignores what you select; it only checks if any row is returned.


**Example:**

Input `Customers` / `Orders`:
| customer_id | name |   | customer_id (Orders) |
|---|---|---|---|
| 1 | Joe |  | 1 |
| 2 | Amy |  | 3 |
| 3 | Sam |  |  |

Output — semi-join (placed an order): `1, 3`
Output — anti-join (no orders, NULL-safe): `2`

---

### Pattern 5 — FIRST_VALUE / LAST_VALUE / NTH_VALUE
**Identify when:** You want to carry a boundary value (first or last in a window) alongside every row — e.g., first purchase date repeated on each row for comparison.

```sql
-- Show each sale alongside the first and last sale amount for that product
SELECT product_id, sale_date, amount,
  FIRST_VALUE(amount) OVER (PARTITION BY product_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS first_sale,
  LAST_VALUE(amount)  OVER (PARTITION BY product_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_sale
FROM Sales;
```

> **Gotcha for LAST_VALUE:** The default frame is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, so LAST_VALUE only sees up to the current row — not what you want. Always specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` explicitly.


**Example:**

Input `Sales` (product 1):
| product_id | sale_date | amount |
|---|---|---|
| 1 | 2024-01-01 | 100 |
| 1 | 2024-01-05 | 150 |
| 1 | 2024-01-10 | 120 |

Output:
| product_id | sale_date | amount | first_sale | last_sale |
|---|---|---|---|---|
| 1 | 2024-01-01 | 100 | 100 | 120 |
| 1 | 2024-01-05 | 150 | 100 | 120 |
| 1 | 2024-01-10 | 120 | 100 | 120 |

---

### Pattern 6 — NTILE / PERCENT_RANK / CUME_DIST (Distribution Functions)
**Identify when:** You need to bucket rows into equal groups (quartiles, deciles) or compute relative rank as a percentage.

```sql
-- Divide customers into 4 quartiles by spend
SELECT customer_id, total_spend,
  NTILE(4) OVER (ORDER BY total_spend DESC) AS quartile,       -- 1=top 25%
  ROUND(PERCENT_RANK() OVER (ORDER BY total_spend) * 100, 1) AS pct_rank,  -- 0-100
  ROUND(CUME_DIST()    OVER (ORDER BY total_spend) * 100, 1) AS cume_pct   -- % at or below
FROM CustomerSpend;
```

> `NTILE(n)` splits rows into n equal buckets. If rows don't divide evenly, earlier buckets get one extra row.  
> `PERCENT_RANK()` = (rank - 1) / (total_rows - 1), ranges 0 to 1.  
> `CUME_DIST()` = rows ≤ current value / total rows, ranges 1/n to 1.


**Example:**

Input `CustomerSpend`:
| customer_id | total_spend |
|---|---|
| 1 | 400 |
| 2 | 300 |
| 3 | 200 |
| 4 | 100 |

Output:
| customer_id | total_spend | quartile | pct_rank | cume_pct |
|---|---|---|---|---|
| 1 | 400 | 1 | 100.0 | 25.0 |
| 2 | 300 | 2 | 66.7 | 50.0 |
| 3 | 200 | 3 | 33.3 | 75.0 |
| 4 | 100 | 4 | 0.0 | 100.0 |

---

### Pattern 7 — Running Totals with Reset (Conditional Window)
**Identify when:** You need a running total that resets based on some condition — e.g., cumulative balance that resets each month, or a streak counter that resets on a loss.

```sql
-- Cumulative sum of sales, reset every month
SELECT sale_date, amount,
  SUM(amount) OVER (
    PARTITION BY DATE_FORMAT(sale_date, '%Y-%m')
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS monthly_running_total
FROM Sales;
```

**Streak counter (resets on condition change):**
```sql
-- Count consecutive wins (result = 'W'), reset on loss
WITH islands AS (
  SELECT player_id, game_date, result,
    ROW_NUMBER() OVER (PARTITION BY player_id ORDER BY game_date) -
    ROW_NUMBER() OVER (PARTITION BY player_id, result ORDER BY game_date) AS island_key
  FROM Games
)
SELECT player_id, result, COUNT(*) AS streak_length, MIN(game_date) AS start, MAX(game_date) AS end
FROM islands
WHERE result = 'W'
GROUP BY player_id, result, island_key;
```


**Example (monthly running total):**

Input `Sales`:
| sale_date | amount |
|---|---|
| 2024-01-05 | 50 |
| 2024-01-20 | 30 |
| 2024-02-02 | 40 |

Output:
| sale_date | amount | monthly_running_total |
|---|---|---|
| 2024-01-05 | 50 | 50 |
| 2024-01-20 | 30 | 80 |
| 2024-02-02 | 40 | 40 |

**Example (streak counter):**

Input `Games` (player 1): W, W, L, W → results on 4 consecutive dates.
Output: two streaks for player 1 — `W` length 2 (days 1-2), `W` length 1 (day 4).

---

### Pattern 8 — Simple WHERE Filter
**Identify when:** You just need rows matching some condition(s).  
**Approach:** SELECT columns FROM table WHERE condition(s).  
**Watch out for:** NULL values — `col != 2` won't match NULLs; always add `OR col IS NULL` if NULLs are valid matches.

```sql
-- Find products that are both low-fat AND recyclable
SELECT product_id FROM Products WHERE low_fats = 'Y' AND recyclable = 'Y';
```


**Example:**

Input `Products`:
| product_id | low_fats | recyclable |
|---|---|---|
| 1 | Y | N |
| 2 | Y | Y |
| 3 | N | Y |

Output: `product_id = 2`

---

### Pattern 9 — NULL Handling
**Identify when:** A column can be NULL and NULL rows should be included or excluded.  
**Approach:** Use `IS NULL` / `IS NOT NULL`. Remember `col != value` silently excludes NULLs.

```sql
-- Include rows where referee_id is not 2 OR has no referee at all
SELECT name FROM Customer WHERE referee_id != 2 OR referee_id IS NULL;
```


**Example:**

Input `Customer`:
| id | name | referee_id |
|---|---|---|
| 1 | Will | NULL |
| 2 | Jane | 1 |
| 3 | Alex | 2 |

Output (`referee_id != 2 OR referee_id IS NULL`): `Will, Jane`

---

### Pattern 10 — INNER JOIN
**Identify when:** You need matching rows from two tables (no unmatched rows needed).  
**Approach:** `JOIN t2 ON t1.key = t2.key` or implicit comma join with WHERE.

```sql
-- Get product name, year, price from Sales + Product
SELECT product_name, year, price
FROM Sales s JOIN Product p ON s.product_id = p.product_id;
```


**Example:**

Input `Sales` / `Product`:
| product_id | year | price |   | product_id | product_name |
|---|---|---|---|---|---|
| 100 | 2019 | 5000 |  | 100 | Nokia |

Output:
| product_name | year | price |
|---|---|---|
| Nokia | 2019 | 5000 |

---

### Pattern 11 — LEFT JOIN (and Anti-Join)
**Identify when:** You need ALL rows from the left table even if no match exists in the right.  
**Anti-join variant:** LEFT JOIN + `WHERE right_table.id IS NULL` to find rows with NO match.

```sql
-- Customers who visited but made NO transaction (anti-join)
SELECT customer_id, COUNT(*) count_no_trans
FROM Visits v LEFT JOIN Transactions t ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY customer_id;
```


**Example:**

Input `Visits` / `Transactions`:
| visit_id | customer_id |   | visit_id (Transactions) |
|---|---|---|---|
| 10 | 1 |  |  |
| 11 | 2 |  | 11 |

Output:
| customer_id | count_no_trans |
|---|---|
| 1 | 1 |

---

### Pattern 12 — CROSS JOIN + LEFT JOIN (All-Pairs Enumeration)
**Identify when:** You need every combination of two sets (e.g., every student × every subject), then optionally count matches.  
**Approach:** `CROSS JOIN` generates all pairs; then `LEFT JOIN` the data table on both keys.

```sql
-- Every student-subject pair, counting exams attended
SELECT s.student_id, s.student_name, sub.subject_name, COUNT(e.student_id) attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e ON s.student_id = e.student_id AND e.subject_name = sub.subject_name
GROUP BY s.student_id, s.student_name, sub.subject_name;
```


**Example:**

Input: 2 students (S1, S2) × 2 subjects (Math, Physics); `Examinations` has one row: S1/Math.

Output:
| student_id | student_name | subject_name | attended_exams |
|---|---|---|---|
| S1 | ... | Math | 1 |
| S1 | ... | Physics | 0 |
| S2 | ... | Math | 0 |
| S2 | ... | Physics | 0 |

---

### Pattern 13 — Self-JOIN
**Identify when:** A table references itself (manager/employee, comparing rows within same table).  
**Approach:** Alias the same table twice and join on the self-referencing key.

```sql
-- Managers with 5+ direct reports
SELECT e.name
FROM Employee e JOIN Employee m ON e.id = m.managerId
GROUP BY m.managerId HAVING COUNT(*) >= 5;
```


**Example** *(threshold lowered to ≥2 here for a compact illustration):*

Input `Employee`:
| id | name | managerId |
|---|---|---|
| 101 | John | NULL |
| 102 | Dan | 101 |
| 103 | James | 101 |

Output: `John` (2 direct reports)

---

### Pattern 14 — GROUP BY + HAVING
**Identify when:** You need to aggregate data per group, then filter on aggregate result.  
**Rule:** WHERE filters rows before grouping; HAVING filters groups after aggregation.

```sql
-- Classes with at least 5 students
SELECT class FROM Courses GROUP BY class HAVING COUNT(student) >= 5;
```


**Example** *(threshold lowered to ≥2 for illustration):*

Input `Courses`:
| student | class |
|---|---|
| A | Math |
| B | Math |
| C | English |

Output: `class = Math`

---

### Pattern 15 — Conditional Aggregation (CASE WHEN inside SUM/COUNT)
**Identify when:** You need to count or sum rows that meet a specific condition within a GROUP BY.  
**Approach:** `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` or `SUM(IF(condition, 1, 0))`.

```sql
-- Count approved vs total transactions per month
SELECT DATE_FORMAT(trans_date, '%Y-%m') month, country,
  COUNT(*) trans_count,
  SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END) approved_count
FROM Transactions GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```


**Example:**

Input `Transactions`:
| trans_date | country | state |
|---|---|---|
| 2018-12-18 | US | approved |
| 2018-12-19 | US | declined |

Output:
| month | country | trans_count | approved_count |
|---|---|---|---|
| 2018-12 | US | 2 | 1 |

---

### Pattern 16 — Subqueries (Scalar / IN / NOT IN)
**Identify when:** You need a value computed from another query, or need to filter based on a set of values from another query.  
**Scalar subquery:** Returns one value; used in SELECT or WHERE.  
**IN/NOT IN:** Filters based on membership in a result set.

```sql
-- Customers who bought ALL products
SELECT customer_id FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (SELECT COUNT(*) FROM Product);
```


**Example:**

Input `Customer` (customer_id, product_key) / `Product` (2 total products: 5, 6):
| customer_id | product_key |
|---|---|
| 1 | 5 |
| 1 | 6 |
| 2 | 5 |

Output: `customer_id = 1` (bought both products)

---

### Pattern 17 — CTE (WITH clause)
**Identify when:** You have a multi-step query that would be hard to read as nested subqueries, or you need to reference the same subquery multiple times.  
**Approach:** Define named temporary result sets before the main query.

```sql
WITH first_login AS (
  SELECT player_id, MIN(event_date) AS login_date FROM Activity GROUP BY player_id
)
SELECT ROUND(COUNT(a.player_id) / (SELECT COUNT(*) FROM first_login), 2) AS fraction
FROM first_login f
JOIN Activity a ON f.player_id = a.player_id
  AND a.event_date = DATE_ADD(f.login_date, INTERVAL 1 DAY);
```


**Example:**

Input `Activity`:
| player_id | event_date |
|---|---|
| 1 | 2016-03-01 |
| 1 | 2016-03-02 |
| 2 | 2017-06-25 |

Output: `fraction = 0.50` (1 of 2 players returned the next day)

---

### Pattern 18 — Window Functions: ROW_NUMBER / RANK / DENSE_RANK
**Identify when:** You need to rank rows within a group (top-N per group, first record per partition, etc.).  
- `ROW_NUMBER()` — unique sequential rank, no ties
- `RANK()` — ties get same rank, next rank skips (1,1,3)
- `DENSE_RANK()` — ties get same rank, no skipping (1,1,2)

```sql
-- Top 3 salaries per department (ties allowed → DENSE_RANK)
WITH ranked AS (
  SELECT name, salary, departmentId,
    DENSE_RANK() OVER (PARTITION BY departmentId ORDER BY salary DESC) AS rnk
  FROM Employee
)
SELECT d.name Department, r.name Employee, r.salary Salary
FROM Department d JOIN ranked r ON r.departmentId = d.id WHERE r.rnk <= 3;
```


**Example:**

Input `Employee`:
| name | salary | departmentId |
|---|---|---|
| Joe | 85000 | 1 |
| Henry | 80000 | 2 |
| Sam | 60000 | 2 |

Output (top 3 per dept, DENSE_RANK): all three rows qualify (each dept has ≤3 employees).

---

### Pattern 19 — Window Functions: LAG / LEAD
**Identify when:** You need to compare a row with its previous or next row (e.g., temperature yesterday vs today, consecutive values).  
- `LAG(col, n)` — value from n rows before
- `LEAD(col, n)` — value from n rows after

```sql
-- Days where temperature is higher than the previous day
SELECT id FROM (
  SELECT id, temperature,
    LAG(temperature) OVER (ORDER BY recordDate) AS prev_temp,
    LAG(recordDate) OVER (ORDER BY recordDate) AS prev_date
  FROM Weather
) t WHERE temperature > prev_temp AND DATEDIFF(recordDate, prev_date) = 1;
```


**Example:**

Input `Weather`:
| id | recordDate | temperature |
|---|---|---|
| 1 | 2015-01-01 | 10 |
| 2 | 2015-01-02 | 25 |
| 3 | 2015-01-03 | 20 |

Output: `id = 2` (25 > 10, consecutive day)

---

### Pattern 20 — Window Functions: SUM/AVG OVER (Rolling Aggregation)
**Identify when:** You need a running total or rolling average over a window of rows.  
**Approach:** `SUM(col) OVER (ORDER BY ... ROWS BETWEEN n PRECEDING AND CURRENT ROW)`.

```sql
-- 7-day rolling average of revenue
SELECT visited_on,
  SUM(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS amount,
  ROUND(AVG(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS avg_amount
FROM daily_table;
```


**Example:**

Input `daily_table`:
| visited_on | amount |
|---|---|
| 2019-01-01 | 100 |
| 2019-01-02 | 110 |

Output:
| visited_on | amount (rolling) | avg_amount |
|---|---|---|
| 2019-01-01 | 100 | 100.00 |
| 2019-01-02 | 210 | 105.00 |

---

### Pattern 21 — UNION / UNION ALL
**Identify when:** You need to combine results from multiple queries into one result set.  
- `UNION` — removes duplicates (slower)
- `UNION ALL` — keeps duplicates (faster, use when duplicates are fine or impossible)

```sql
-- Combine all friend relationships (bidirectional)
SELECT requester_id AS id FROM RequestAccepted
UNION ALL
SELECT accepter_id AS id FROM RequestAccepted;
```


**Example:**

Input `RequestAccepted`:
| requester_id | accepter_id |
|---|---|
| 1 | 2 |
| 3 | 4 |

Output (all ids, both directions): `1, 3, 2, 4`

---

### Pattern 22 — Date Functions
**Identify when:** You need to filter, format, or compute date differences.  
**MySQL key functions:**
- `DATE_FORMAT(date, '%Y-%m')` — format to year-month string
- `DATE_ADD(date, INTERVAL n DAY/MONTH/YEAR)` — add to a date
- `DATE_SUB(date, INTERVAL n DAY)` — subtract from a date
- `DATEDIFF(date1, date2)` — days between (date1 - date2)
- `YEAR(date)`, `MONTH(date)`, `DAY(date)` — extract parts

```sql
-- Active users in the last 30 days ending 2019-07-27
SELECT activity_date AS day, COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN DATE_SUB('2019-07-27', INTERVAL 29 DAY) AND '2019-07-27'
GROUP BY activity_date;
```


**Example:**

Input `Activity`:
| user_id | activity_date |
|---|---|
| 1 | 2019-07-20 |
| 2 | 2019-07-20 |

Output:
| day | active_users |
|---|---|
| 2019-07-20 | 2 |

---

### Pattern 23 — String Functions
**Identify when:** You need to manipulate text values.  
**MySQL key functions:**
- `CONCAT(a, b)` — join strings
- `UPPER(s)`, `LOWER(s)` — change case
- `LEFT(s, n)`, `RIGHT(s, n)` — take characters from start/end
- `LENGTH(s)` / `CHAR_LENGTH(s)` — byte length vs character length
- `LIKE '%pattern%'` — wildcard matching
- `REGEXP` / `RLIKE` — regex matching (MySQL), `~` (PostgreSQL)
- `GROUP_CONCAT(col ORDER BY ... SEPARATOR ',')` — aggregate strings

```sql
-- Proper-case a name: capitalize first letter, lowercase the rest
SELECT CONCAT(UPPER(LEFT(name, 1)), LOWER(SUBSTRING(name, 2))) AS name FROM Users;
```


**Example:**

Input `Users`: `name = 'FIRSTNAME lastname'`

Output: `name = 'Firstname lastname'`

---

### Pattern 24 — Tuple / Multi-Column IN (MySQL-specific)
**Identify when:** You need to filter on a combination of columns.  
**Approach:** `(col1, col2) IN (SELECT col1, col2 FROM ...)` — MySQL supports tuple comparison directly.

```sql
-- Policies with a unique (lat, lon) pair
WHERE (lat, lon) IN (SELECT lat, lon FROM Insurance GROUP BY lat, lon HAVING COUNT(*) = 1)
```


**Example:**

Input `Insurance`:
| pid | lat | lon |
|---|---|---|
| 1 | 5 | 10 |
| 2 | 5 | 10 |
| 3 | 6 | 20 |

Output: `pid = 3` (only lat/lon pair that appears once)

---

### Pattern 25 — DELETE with Subquery
**Identify when:** You need to delete rows based on a condition computed from the same table.  
**MySQL quirk:** You cannot reference the same table in a subquery directly in DELETE — wrap in another subquery.

```sql
-- Delete duplicate emails, keep lowest id
DELETE FROM Person
WHERE id NOT IN (SELECT * FROM (SELECT MIN(id) FROM Person GROUP BY email) tmp);
```


**Example:**

Input `Person`:
| id | email |
|---|---|
| 1 | a@b.com |
| 2 | a@b.com |
| 3 | c@d.com |

Output (after DELETE): rows `id = 1, 3` remain (id 2 removed as the higher duplicate id).

---

### Pattern 26 — COALESCE / IFNULL for Default Values
**Identify when:** A LEFT JOIN might return NULL for an unmatched row, and you want a default value instead.  
- `COALESCE(expr, default)` — returns first non-NULL value
- `IFNULL(expr, default)` — MySQL shorthand for COALESCE with one fallback

```sql
-- Products with no price change default to 10
SELECT ap.product_id, COALESCE(r.new_price, 10) AS price
FROM all_products ap LEFT JOIN ranked r ON ap.product_id = r.product_id AND r.rnk = 1;
```


**Example:**

Input `all_products` / `ranked` (rnk = 1 rows only):
| product_id |   | product_id | new_price | rnk |
|---|---|---|---|---|
| 1 |  | 1 | 20 | 1 |
| 2 |  |  |  |  |

Output:
| product_id | price |
|---|---|
| 1 | 20 |
| 2 | 10 |

---

### Pattern 27 — Odd/Even and Modulo Logic
**Identify when:** You need to swap adjacent rows, filter odd/even IDs, or categorize by remainder.  
**Approach:** `id % 2 = 1` for odd, `id % 2 = 0` for even. Use CASE WHEN to swap.

```sql
-- Swap every two adjacent students
SELECT
  CASE WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM Seat) THEN id
       WHEN id % 2 = 1 THEN id + 1
       ELSE id - 1 END AS id, student
FROM Seat ORDER BY id;
```


**Example:**

Input `Seat`:
| id | student |
|---|---|
| 1 | Abbot |
| 2 | Doris |
| 3 | Emerson |

Output:
| id | student |
|---|---|
| 1 | Doris |
| 2 | Abbot |
| 3 | Emerson |

---

### Pattern 28 — Deduplication with ROW_NUMBER
**Identify when:** A table has duplicate rows (same key, multiple entries) and you need to keep only one — usually the latest, earliest, or highest-priority record.  
**Approach:** `ROW_NUMBER() OVER (PARTITION BY key ORDER BY tiebreaker)` then filter `WHERE rn = 1`.

```sql
-- Keep only the most recent record per customer
WITH deduped AS (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) AS rn
  FROM Customers
)
SELECT * FROM deduped WHERE rn = 1;
```

**In a DELETE context (remove dupes, keep latest):**
```sql
DELETE FROM Customers
WHERE id NOT IN (
  SELECT id FROM (
    SELECT id,
      ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) AS rn
    FROM Customers
  ) t WHERE rn = 1
);
```


**Example:**

Input `Customers`:
| id | customer_id | updated_at |
|---|---|---|
| 1 | 100 | 2024-01-01 |
| 2 | 100 | 2024-02-01 |

Output (deduped, `rn = 1`): row `id = 2` (most recent per customer_id).

---

### Pattern 29 — Sessionization (Event Grouping by Time Gap)
**Identify when:** You have a stream of user events and want to group them into "sessions" — a session ends when the gap between consecutive events exceeds a threshold (e.g., 30 minutes).  
**Approach:** Use LAG to detect session boundaries, then cumulative SUM to assign session IDs.

```sql
WITH gaps AS (
  SELECT user_id, event_time,
    CASE
      WHEN TIMESTAMPDIFF(MINUTE,
             LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time),
             event_time) > 30
        OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
      THEN 1 ELSE 0
    END AS is_new_session
  FROM Events
),
sessions AS (
  SELECT user_id, event_time,
    SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
  FROM gaps
)
SELECT user_id, session_id,
  MIN(event_time) AS session_start,
  MAX(event_time) AS session_end,
  COUNT(*) AS event_count
FROM sessions
GROUP BY user_id, session_id;
```

> **Key insight:** Mark rows where the gap > threshold as a new session start (1), then a running SUM creates an incrementing session counter — each new session bumps the counter by 1.


**Example** *(30-min gap threshold):*

Input `Events` (user 1): 09:00, 09:15, 10:00, 10:05

Output:
| user_id | session_id | session_start | session_end | event_count |
|---|---|---|---|---|
| 1 | 1 | 09:00 | 09:15 | 2 |
| 1 | 2 | 10:00 | 10:05 | 2 |

---

### Pattern 30 — Temporal Join (Join on Date Ranges)
**Identify when:** You need to join on a date falling within a range, not an equality — e.g., get the price valid on a specific purchase date, or find the employee's department at a given point in time.

```sql
-- What was the salary of each employee on 2023-06-15?
SELECT e.employee_id, e.name, s.salary
FROM Employees e
JOIN SalaryHistory s
  ON e.employee_id = s.employee_id
  AND '2023-06-15' BETWEEN s.effective_date AND COALESCE(s.end_date, '9999-12-31');
```

**Latest-record-before-date join (SCD Type 2 style):**
```sql
-- Get the most recent salary record as of target date
WITH ranked AS (
  SELECT employee_id, salary, effective_date,
    ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY effective_date DESC) AS rn
  FROM SalaryHistory
  WHERE effective_date <= '2023-06-15'
)
SELECT * FROM ranked WHERE rn = 1;
```

> **COALESCE(end_date, '9999-12-31')** is the standard SCD Type 2 trick for "currently active" records that have no end date.


**Example:**

Input `SalaryHistory`:
| employee_id | salary | effective_date | end_date |
|---|---|---|---|
| 1 | 60000 | 2022-01-01 | 2023-01-01 |
| 1 | 65000 | 2023-01-01 | NULL |

Output (as of 2023-06-15): `employee_id = 1, salary = 65000`

---

### Pattern 31 — Funnel Analysis
**Identify when:** You need to measure how many users completed each step of a multi-step process (signup → verify → purchase → repeat).  
**Approach:** Use conditional COUNT DISTINCT per step, often with a single GROUP BY on user cohort.

```sql
-- Conversion funnel: how many users reached each step?
SELECT
  COUNT(DISTINCT user_id) AS total_users,
  COUNT(DISTINCT CASE WHEN step >= 1 THEN user_id END) AS step1_viewed,
  COUNT(DISTINCT CASE WHEN step >= 2 THEN user_id END) AS step2_added_to_cart,
  COUNT(DISTINCT CASE WHEN step >= 3 THEN user_id END) AS step3_purchased,
  ROUND(COUNT(DISTINCT CASE WHEN step >= 3 THEN user_id END) * 100.0
        / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS overall_conversion_pct
FROM FunnelEvents;
```

**With PIVOT-style output (one row per step):**
```sql
SELECT step_name, COUNT(DISTINCT user_id) AS users,
  ROUND(COUNT(DISTINCT user_id) * 100.0 /
    MAX(COUNT(DISTINCT user_id)) OVER (), 1) AS pct_of_top
FROM FunnelEvents
GROUP BY step_name
ORDER BY MIN(step_number);
```


**Example:**

Input `FunnelEvents`:
| user_id | step |
|---|---|
| 1 | 1 |
| 1 | 2 |
| 1 | 3 |
| 2 | 1 |

Output:
| total_users | step1_viewed | step2_added_to_cart | step3_purchased | overall_conversion_pct |
|---|---|---|---|---|
| 2 | 2 | 1 | 1 | 50.0 |

---

### Pattern 32 — Cohort Analysis (Retention)
**Identify when:** You want to track how a group of users acquired in a given period behaves over time — Week 0 (acquisition), Week 1 (retention), Week 2, etc.

```sql
WITH cohorts AS (
  SELECT user_id, DATE_FORMAT(MIN(event_date), '%Y-%m') AS cohort_month
  FROM Events GROUP BY user_id
),
activity AS (
  SELECT e.user_id, c.cohort_month,
    TIMESTAMPDIFF(MONTH, STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'), e.event_date) AS month_number
  FROM Events e JOIN cohorts c ON e.user_id = c.user_id
)
SELECT cohort_month, month_number,
  COUNT(DISTINCT user_id) AS active_users
FROM activity
GROUP BY cohort_month, month_number
ORDER BY cohort_month, month_number;
```

> Month 0 = first month (acquisition). Month 1 = users who came back the following month. Divide each month's count by the cohort's Month 0 count to get **retention rate**.


**Example:**

Input `Events`:
| user_id | event_date |
|---|---|
| 1 | 2024-01-05 |
| 1 | 2024-02-10 |
| 2 | 2024-01-20 |

Output:
| cohort_month | month_number | active_users |
|---|---|---|
| 2024-01 | 0 | 2 |
| 2024-01 | 1 | 1 |

---

### Pattern 33 — Median and Percentile Without Built-ins
**Identify when:** You need the median (or arbitrary percentile) but your DB lacks `PERCENTILE_CONT`. MySQL doesn't have it; PostgreSQL and SQL Server do.

**MySQL median approach:**
```sql
-- Median salary (works for both odd and even row counts)
SELECT AVG(salary) AS median
FROM (
  SELECT salary,
    ROW_NUMBER() OVER (ORDER BY salary) AS rn,
    COUNT(*) OVER () AS total
  FROM Employee
) t
WHERE rn IN (FLOOR((total + 1) / 2), CEIL((total + 1) / 2));
```

**PostgreSQL / SQL Server (native):**
```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median,
       PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS p25,
       PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS p75
FROM Employee;
```

> The MySQL approach picks the middle row(s): for odd total, `FLOOR = CEIL` → one row; for even total, they differ by one → AVG of the two middle values.


**Example:**

Input `Employee.salary`: `[3000, 4000, 5000, 6000]`

Output: `median = 4500` (avg of the two middle values, 4000 and 5000)

---

### Pattern 34 — Forward Fill (Last Observation Carried Forward)
**Identify when:** A time series has NULLs where data wasn't recorded, and you want to fill each NULL with the most recent non-NULL value above it.  
**Approach:** Use `MAX(col) IGNORE NULLS OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` — supported in some DBs. In MySQL, use a subquery workaround.

```sql
-- PostgreSQL / SQL Server (IGNORE NULLS supported)
SELECT date, price,
  LAST_VALUE(price IGNORE NULLS) OVER (ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS filled_price
FROM StockPrices;

-- MySQL workaround (correlated subquery)
SELECT date, price,
  (SELECT price FROM StockPrices s2
   WHERE s2.date <= s1.date AND s2.price IS NOT NULL
   ORDER BY s2.date DESC LIMIT 1) AS filled_price
FROM StockPrices s1;
```


**Example:**

Input `StockPrices`:
| date | price |
|---|---|
| 2024-01-01 | 100 |
| 2024-01-02 | NULL |
| 2024-01-03 | NULL |
| 2024-01-04 | 110 |

Output:
| date | filled_price |
|---|---|
| 2024-01-01 | 100 |
| 2024-01-02 | 100 |
| 2024-01-03 | 100 |
| 2024-01-04 | 110 |

---

### Pattern 35 — Set Operations: INTERSECT / EXCEPT
**Identify when:** You need rows common to two result sets (INTERSECT) or rows in one set but not the other (EXCEPT / MINUS).  
**MySQL note:** MySQL (before 8.0.31) didn't support INTERSECT/EXCEPT natively — simulate with JOIN or NOT IN.

```sql
-- Standard SQL (PostgreSQL, SQL Server, MySQL 8.0.31+)
-- Customers who placed orders in both Jan AND Feb
SELECT customer_id FROM Orders WHERE MONTH(order_date) = 1
INTERSECT
SELECT customer_id FROM Orders WHERE MONTH(order_date) = 2;

-- Customers who ordered in Jan but NOT Feb
SELECT customer_id FROM Orders WHERE MONTH(order_date) = 1
EXCEPT
SELECT customer_id FROM Orders WHERE MONTH(order_date) = 2;

-- MySQL pre-8.0.31 equivalents:
-- INTERSECT → INNER JOIN on keys
-- EXCEPT   → LEFT JOIN anti-join (WHERE right.key IS NULL)
```


**Example:**

Input `Orders`:
| customer_id | order_date |
|---|---|
| 1 | 2024-01-05 |
| 1 | 2024-02-05 |
| 2 | 2024-01-10 |

Output — INTERSECT (ordered in both Jan and Feb): `customer_id = 1`
Output — EXCEPT (Jan but not Feb): `customer_id = 2`

---

### Pattern 36 — Dynamic Top-N per Group with Ties
**Identify when:** You want the top N rows per partition, with the tie-handling choice affecting correctness.

| Function | Behavior on tie | Use case |
|---|---|---|
| `ROW_NUMBER()` | Breaks tie arbitrarily (one row) | Strict top-N, no duplicates |
| `RANK()` | Ties share rank, next rank skips | "Top 3 ranks" — may return >3 rows |
| `DENSE_RANK()` | Ties share rank, no skipping | "Top 3 salary levels" — may return >3 rows |

```sql
-- Top 2 earners per department (ties included → DENSE_RANK)
SELECT department, employee, salary
FROM (
  SELECT department, employee, salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dr
  FROM Employee
) t
WHERE dr <= 2;
```


**Example:**

Input `Employee`:
| department | employee | salary |
|---|---|---|
| Sales | A | 9000 |
| Sales | B | 9000 |
| Sales | C | 8000 |

Output (top 2 salary levels, DENSE_RANK): `A, B` (tied at rank 1), `C` (rank 2) — 3 rows returned for "top 2".

---

### Pattern 37 — Running Balance / Ledger Pattern
**Identify when:** You have a transactions table (credits and debits) and need a running account balance after each transaction.

```sql
SELECT txn_id, txn_date, amount, txn_type,
  SUM(CASE WHEN txn_type = 'credit' THEN amount ELSE -amount END)
    OVER (PARTITION BY account_id ORDER BY txn_date, txn_id
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
FROM Transactions
ORDER BY account_id, txn_date, txn_id;
```

> The `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame is key — it's a cumulative sum from the very first row in the partition up to and including the current one. Without this frame specification, the default may include future rows.


**Example:**

Input `Transactions` (account 1):
| txn_id | txn_date | amount | txn_type |
|---|---|---|---|
| 1 | 2024-01-01 | 100 | credit |
| 2 | 2024-01-02 | 30 | debit |

Output:
| txn_id | running_balance |
|---|---|
| 1 | 100 |
| 2 | 70 |

---

### Pattern 38 — LATERAL JOIN / CROSS APPLY
**Identify when:** You need to join a derived table that references columns from the outer query — like calling a function per row, or taking the top-N rows per outer row.  
- **MySQL 8.0+:** `LATERAL` keyword in subquery
- **SQL Server:** `CROSS APPLY` / `OUTER APPLY`
- **PostgreSQL:** Supports `LATERAL` natively

```sql
-- MySQL 8.0+: for each department, get the top 2 earners
SELECT d.name AS department, top_emp.name, top_emp.salary
FROM Department d
JOIN LATERAL (
  SELECT name, salary
  FROM Employee e
  WHERE e.departmentId = d.id
  ORDER BY salary DESC
  LIMIT 2
) top_emp ON TRUE;

-- SQL Server equivalent with CROSS APPLY
SELECT d.name, top_emp.name, top_emp.salary
FROM Department d
CROSS APPLY (
  SELECT TOP 2 name, salary
  FROM Employee e WHERE e.departmentId = d.id
  ORDER BY salary DESC
) top_emp;
```

> Without LATERAL, correlated subqueries can only return a scalar value. LATERAL lets the subquery see and use the outer row, effectively running once per outer row — powerful but potentially slow on large datasets without good indexes.


**Example:**

Input: `Department` (Sales), `Employee` (Sales: A/9000, B/8000, C/7000)

Output:
| department | name | salary |
|---|---|---|
| Sales | A | 9000 |
| Sales | B | 8000 |

---

### Pattern 39 — Slowly Changing Dimensions (SCD) Type 2
**Identify when:** You need to track historical changes to dimension data — what was the customer's address last year? What was the price in Q3?  
**Structure:** Each change creates a new row with `effective_date`, `expiry_date` (NULL for current), and often an `is_current` flag.

```sql
-- Find the current record for each entity
SELECT * FROM DimCustomer WHERE is_current = 1;

-- Find the record valid at a specific point in time
SELECT * FROM DimCustomer
WHERE customer_id = 42
  AND effective_date <= '2023-06-15'
  AND (expiry_date > '2023-06-15' OR expiry_date IS NULL);

-- Insert a new version when data changes (application logic):
-- 1. UPDATE old row: SET expiry_date = TODAY, is_current = 0
-- 2. INSERT new row: effective_date = TODAY, expiry_date = NULL, is_current = 1
```


**Example:**

Input `DimCustomer`:
| customer_id | address | effective_date | expiry_date | is_current |
|---|---|---|---|---|
| 42 | Old St | 2022-01-01 | 2023-01-01 | 0 |
| 42 | New Ave | 2023-01-01 | NULL | 1 |

Output (current record): `customer_id = 42, address = New Ave`

---

## Part 2: LeetCode SQL 50 — All Problems

---

### Q1 — Recyclable and Low Fat Products
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** Find products that are both low fat (`low_fats = 'Y'`) and recyclable (`recyclable = 'Y'`).

**Pattern:** Simple WHERE filter with AND.

**Solution:**
```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

**Complexity:** O(n) time — single table scan. O(k) space for k matching rows.

**Follow-ups:**
- *What if you want products that are EITHER low fat OR recyclable?* → Use `OR` instead of `AND`.
- *What if the columns stored 'Yes'/'No' instead of 'Y'/'N'?* → `WHERE low_fats = 'Yes' AND recyclable = 'Yes'` or use `LOWER(low_fats) = 'y'` for case-insensitive matching.


**Example:**

Input `Products`:
| product_id | low_fats | recyclable |
|---|---|---|
| 1 | Y | N |
| 2 | Y | Y |
| 3 | N | Y |

Output: `product_id = 2`

---

### Q2 — Find Customer Referee
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** Find customers NOT referred by customer with id = 2. Customers with no referee (`NULL`) should also be included.

**Pattern:** WHERE with NULL handling.

**Solution:**
```sql
SELECT name
FROM Customer
WHERE referee_id != 2 OR referee_id IS NULL;
```

**Complexity:** O(n) time. O(k) space.

**Follow-ups:**
- *Why doesn't `WHERE referee_id != 2` work alone?* → In SQL, any comparison with NULL returns UNKNOWN (not TRUE or FALSE), so rows where `referee_id IS NULL` are silently excluded by `!=`. You must explicitly check for NULLs.
- *Alternative using COALESCE?* → `WHERE COALESCE(referee_id, 0) != 2` — replaces NULL with 0 before comparison.


**Example:**

Input `Customer`:
| id | name | referee_id |
|---|---|---|
| 1 | Will | NULL |
| 2 | Jane | 1 |
| 3 | Alex | 2 |

Output: `Will, Jane`

---

### Q3 — Big Countries
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** A country is "big" if it has area ≥ 3,000,000 OR population ≥ 25,000,000.

**Pattern:** Simple WHERE with OR.

**Solution:**
```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000;
```

**Complexity:** O(n) time. O(k) space.

**Follow-ups:**
- *How would you sort results by population descending?* → Add `ORDER BY population DESC`.
- *What if the question asked for countries that are big by BOTH criteria?* → Replace `OR` with `AND`.


**Example:**

Input `World`:
| name | population | area |
|---|---|---|
| Algeria | 37100000 | 2381741 |
| Andorra | 78115 | 468 |

Output: `Algeria` (population ≥ 25,000,000)

---

### Q4 — Article Views I
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** Find all authors who viewed at least one of their own articles. An author views their own article when `author_id = viewer_id`.

**Pattern:** WHERE self-match + DISTINCT + ORDER BY.

**Solution:**
```sql
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY id;
```

**Complexity:** O(n log n) time (sorting). O(k) space.

**Follow-ups:**
- *Why DISTINCT?* → The same author might have viewed multiple of their own articles; we want each author listed once.
- *How would you find authors who have NEVER viewed their own articles?* → `SELECT DISTINCT author_id FROM Views WHERE author_id NOT IN (SELECT DISTINCT author_id FROM Views WHERE author_id = viewer_id)`.


**Example:**

Input `Views`:
| article_id | author_id | viewer_id |
|---|---|---|
| 1 | 3 | 5 |
| 2 | 7 | 7 |

Output: `id = 7`

---

### Q5 — Invalid Tweets
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** Find tweets where the content has more than 15 characters.

**Pattern:** String length filter.

**Solution (MS SQL Server — uses LEN):**
```sql
SELECT tweet_id
FROM Tweets
WHERE LEN(content) > 15;
```

**MySQL equivalent:**
```sql
SELECT tweet_id FROM Tweets WHERE CHAR_LENGTH(content) > 15;
```

> **Note:** `LEN()` is MS SQL Server; `CHAR_LENGTH()` counts characters (Unicode-safe) in MySQL; `LENGTH()` in MySQL counts bytes (differs for multi-byte chars).

**Complexity:** O(n) time. O(k) space.

**Follow-ups:**
- *What's the difference between LENGTH and CHAR_LENGTH in MySQL?* → `LENGTH()` returns byte count; `CHAR_LENGTH()` returns character count. They differ for multi-byte characters (e.g., UTF-8 emojis take 4 bytes but count as 1 character).
- *How would you filter tweets longer than 15 characters and containing a specific word?* → `WHERE CHAR_LENGTH(content) > 15 AND content LIKE '%word%'`.


**Example:**

Input `Tweets`:
| tweet_id | content |
|---|---|
| 1 | "This is a short tweet" |
| 2 | "This tweet content is way over fifteen characters long" |

Output: `tweet_id = 2`

---

### Q6 — Replace Employee ID With The Unique Identifier
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Show each employee's unique ID from EmployeeUNI, or NULL if they don't have one.

**Pattern:** LEFT JOIN — keep all employees, optionally attach unique ID.

**Solution:**
```sql
SELECT u.unique_id, e.name
FROM Employees e
LEFT JOIN EmployeeUNI u ON e.id = u.id;
```

**Complexity:** O(n + m) time where n = Employees rows, m = EmployeeUNI rows. O(n) space.

**Follow-ups:**
- *What if you wanted ONLY employees who have a unique ID?* → Use INNER JOIN instead.
- *What if an employee could have multiple unique IDs?* → The LEFT JOIN would produce duplicate rows. You'd need `GROUP BY e.id` or a subquery to pick one.


**Example:**

Input `Employees` / `EmployeeUNI`:
| id | name |   | id | unique_id |
|---|---|---|---|---|
| 1 | Alice |  | 1 | 101 |
| 2 | Bob |  |  |  |

Output:
| unique_id | name |
|---|---|
| 101 | Alice |
| NULL | Bob |

---

### Q7 — Product Sales Analysis I
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Get the product name, year, and price for every sale.

**Pattern:** INNER JOIN (implicit comma join with WHERE).

**Solution:**
```sql
SELECT product_name, year, price
FROM Sales s, Product p
WHERE s.product_id = p.product_id;
```

**Explicit JOIN equivalent:**
```sql
SELECT product_name, year, price
FROM Sales s JOIN Product p ON s.product_id = p.product_id;
```

**Complexity:** O(n * m) naive, O(n + m) with index on product_id. O(n) space.

**Follow-ups:**
- *What's the difference between implicit comma join and explicit JOIN syntax?* → Functionally equivalent for INNER JOIN, but explicit JOIN is preferred for readability and to avoid accidentally writing a Cartesian product.
- *How would you get only distinct product_name + year combinations?* → Add `SELECT DISTINCT` or `GROUP BY product_name, year`.


**Example:**

Input `Sales` / `Product`:
| product_id | year | price |   | product_id | product_name |
|---|---|---|---|---|---|
| 100 | 2019 | 5000 |  | 100 | Nokia |

Output: `product_name = Nokia, year = 2019, price = 5000`

---

### Q8 — Customer Who Visited but Did Not Make Any Transactions
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Find customers who visited but had no transactions, and count how many such visits each had.

**Pattern:** LEFT JOIN anti-join — find unmatched rows.

**Solution:**
```sql
SELECT customer_id, COUNT(*) AS count_no_trans
FROM Visits v
LEFT JOIN Transactions t ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY customer_id;
```

**Complexity:** O(n + m) time. O(k) space where k = distinct customers with no transactions.

**Follow-ups:**
- *Why not use NOT IN with a subquery?* → `NOT IN` with a subquery works but can be slow on large datasets and has NULL-trap issues. LEFT JOIN anti-join is typically faster.
- *What if a single visit had some transactions but you only want visits with zero transactions?* → The current solution is correct — if any transaction exists for a visit_id, the LEFT JOIN finds a match and WHERE filters it out.


**Example:**

Input `Visits` / `Transactions`:
| visit_id | customer_id |   | visit_id (Transactions) |
|---|---|---|---|
| 10 | 1 |  |  |
| 11 | 2 |  | 11 |

Output: `customer_id = 1, count_no_trans = 1`

---

### Q9 — Rising Temperature
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Find all dates where temperature was higher than the previous day.

**Pattern:** Self-join on date difference OR window function LAG.

**Your Solution (beats 18.91%):**
```sql
SELECT w.id AS id
FROM Weather w, Weather w2
WHERE DATEDIFF(w.recordDate, w2.recordDate) = 1
  AND w.temperature > w2.temperature;
```

**Claude's Solution (Solution 2) — using LAG (cleaner, avoids Cartesian product):**
```sql
SELECT id
FROM (
  SELECT id, temperature,
    LAG(temperature) OVER (ORDER BY recordDate) AS prev_temp,
    LAG(recordDate) OVER (ORDER BY recordDate) AS prev_date
  FROM Weather
) t
WHERE temperature > prev_temp
  AND DATEDIFF(recordDate, prev_date) = 1;
```

> The self-join approach creates an O(n²) Cartesian product before filtering. The LAG approach is O(n log n) — the sort for the window function. The DATEDIFF check in both solutions ensures non-consecutive dates are handled correctly (gaps in data).

**Complexity:**
- Your solution: O(n²) time, O(1) extra space
- Claude's solution: O(n log n) time, O(n) space for the window computation

**Follow-ups:**
- *What if there are gaps in dates (missing days)?* → Both solutions handle this correctly via the `DATEDIFF = 1` check.
- *What if multiple records exist for the same date?* → The problem guarantees unique dates, but if not, you'd need to aggregate per date first.


**Example:**

Input `Weather`:
| id | recordDate | temperature |
|---|---|---|
| 1 | 2015-01-01 | 10 |
| 2 | 2015-01-02 | 25 |
| 3 | 2015-01-03 | 20 |

Output: `id = 2`

---

### Q10 — Average Time of Process per Machine
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Each (machine_id, process_id) pair has a start and end timestamp. Find average processing time per machine.

**Pattern:** Conditional aggregation (pivot start/end within group), then average.

**Solution (beats 94.52%):**
```sql
SELECT machine_id, ROUND(AVG(end_time - start_time), 3) AS processing_time
FROM (
  SELECT machine_id, process_id,
    MAX(CASE WHEN activity_type = 'start' THEN timestamp END) AS start_time,
    MAX(CASE WHEN activity_type = 'end' THEN timestamp END) AS end_time
  FROM Activity
  GROUP BY machine_id, process_id
) AS subq
GROUP BY machine_id;
```

**Complexity:** O(n) time (two passes: one for the subquery GROUP BY, one for the outer GROUP BY). O(m) space where m = distinct (machine_id, process_id) pairs.

**Follow-ups:**
- *Could you solve this with a self-join instead?* → Yes: join `Activity a1` (start) with `Activity a2` (end) on `machine_id = machine_id AND process_id = process_id AND a1.type = 'start' AND a2.type = 'end'`, then `AVG(a2.timestamp - a1.timestamp)`.
- *What if a process had no end event?* → The CASE WHEN would return NULL for end_time, and the subtraction would be NULL, which AVG automatically ignores.


**Example:**

Input `Activity`:
| machine_id | process_id | activity_type | timestamp |
|---|---|---|---|
| 0 | 0 | start | 0.712 |
| 0 | 0 | end | 1.520 |

Output:
| machine_id | processing_time |
|---|---|
| 0 | 0.808 |

---

### Q11 — Employee Bonus
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** Report employees with bonus less than 1000, including those with no bonus record.

**Pattern:** LEFT JOIN + WHERE allowing NULLs.

**Solution (beats 52.10%):**
```sql
SELECT e.name, b.bonus
FROM Employee e
LEFT JOIN Bonus b ON e.empId = b.empId
WHERE b.bonus < 1000 OR b.empId IS NULL;
```

**Complexity:** O(n + m) time. O(n) space.

**Follow-ups:**
- *Why use `b.empId IS NULL` instead of `b.bonus IS NULL`?* → Both work here, but checking the join key (empId) is semantically clearer — it confirms the employee has no bonus record at all, not just that the bonus value is stored as NULL.
- *How would you find employees whose bonus is exactly NULL (i.e., have no record)?* → `WHERE b.empId IS NULL`.


**Example:**

Input `Employee` / `Bonus`:
| empId | name |   | empId | bonus |
|---|---|---|---|---|
| 3 | Brad |  | 2 | 500 |
| 2 | Ryan |  |  |  |

Output: `Brad, NULL` and `Ryan, 500` (Ryan's 500 < 1000)

---

### Q12 — Students and Examinations
**Difficulty:** Easy | **Category:** Basic Joins  
**Problem:** For every student–subject combination, count how many exams the student attended for that subject (0 if none).

**Pattern:** CROSS JOIN all student-subject pairs + LEFT JOIN actual exam data.

**Solution (beats 63.38%):**
```sql
SELECT s.student_id, s.student_name, sub.subject_name,
  COUNT(e.student_id) AS attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e
  ON s.student_id = e.student_id AND e.subject_name = sub.subject_name
GROUP BY s.student_id, s.student_name, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

**Complexity:** O(S × Sub + E) time where S = students, Sub = subjects, E = exam records. O(S × Sub) space for the Cartesian product.

**Follow-ups:**
- *Why CROSS JOIN instead of just joining Examinations?* → Because a student who never took a subject exam would simply not appear in Examinations. CROSS JOIN ensures we generate the full matrix of student-subject pairs first, then LEFT JOIN fills in the count (0 for missing pairs via COUNT of a NULL column).
- *Why does `COUNT(e.student_id)` return 0 for unmatched pairs?* → COUNT of a specific column counts non-NULL values. When the LEFT JOIN finds no match, `e.student_id` is NULL, so COUNT returns 0.


**Example:**

Input `Students` (1 row), `Subjects` (2 rows: Math, Physics), `Examinations` (1 row: student 1 / Math)

Output:
| student_id | subject_name | attended_exams |
|---|---|---|
| 1 | Math | 1 |
| 1 | Physics | 0 |

---

### Q13 — Managers with at Least 5 Direct Reports
**Difficulty:** Medium | **Category:** Basic Joins  
**Problem:** Find managers who have at least 5 employees directly reporting to them.

**Pattern:** Self-join + GROUP BY + HAVING.

**Solution (beats 70.87%):**
```sql
SELECT e.name
FROM Employee e, Employee m
WHERE e.id = m.managerId
GROUP BY m.managerId
HAVING COUNT(m.managerId) >= 5;
```

**Explicit JOIN version:**
```sql
SELECT e.name
FROM Employee e
JOIN Employee m ON e.id = m.managerId
GROUP BY e.id, e.name
HAVING COUNT(*) >= 5;
```

**Complexity:** O(n log n) time (sort + group). O(k) space where k = number of qualifying managers.

**Follow-ups:**
- *What if a manager also reports to someone? Does that affect the result?* → No. We're counting how many employees list a given manager_id, regardless of whether that manager also has their own manager.
- *How would you find managers with between 3 and 7 direct reports?* → `HAVING COUNT(*) BETWEEN 3 AND 7`.


**Example** *(threshold shown as ≥2 for compactness; real problem uses ≥5):*

Input `Employee`:
| id | name | managerId |
|---|---|---|
| 101 | John | NULL |
| 102 | Dan | 101 |
| 103 | James | 101 |

Output: `John` (2 direct reports)

---

### Q14 — Confirmation Rate
**Difficulty:** Medium | **Category:** Basic Joins  
**Problem:** For each user in Signups, compute the fraction of confirmation messages that had action = 'confirmed'. Users with no confirmations get rate 0.

**Pattern:** LEFT JOIN + conditional AVG/SUM.

**Solution (beats 87.43%):**
```sql
SELECT s.user_id,
  ROUND(SUM(CASE WHEN action = 'confirmed' THEN 1 ELSE 0 END) / COUNT(*), 2) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c ON s.user_id = c.user_id
GROUP BY s.user_id;
```

**Complexity:** O(n + m) time. O(k) space where k = distinct users.

**Follow-ups:**
- *What happens when a user has no rows in Confirmations after LEFT JOIN?* → COUNT(*) returns 1 (the NULL row from the LEFT JOIN), and SUM returns 0, giving 0/1 = 0.00. ✓
- *Alternative using AVG?* → `ROUND(AVG(CASE WHEN action = 'confirmed' THEN 1.0 ELSE 0 END), 2)` — but AVG ignores NULLs, so with LEFT JOIN producing NULLs, you'd need `COALESCE(action, 'timeout')` first.


**Example:**

Input `Signups` / `Confirmations`:
| user_id |   | user_id | action |
|---|---|---|---|
| 3 |  | 3 | confirmed |
| 3 |  | 3 | timeout |

Output: `user_id = 3, confirmation_rate = 0.50`

---

### Q15 — Not Boring Movies
**Difficulty:** Easy | **Category:** Basic Select  
**Problem:** Find movies with odd ID and description not equal to 'boring', ordered by rating descending.

**Pattern:** WHERE with modulo + string inequality + ORDER BY.

**Solution (beats 40.34%):**
```sql
SELECT id, movie, description, rating
FROM Cinema
WHERE id % 2 = 1 AND description <> 'boring'
ORDER BY rating DESC;
```

**Complexity:** O(n log n) time (sort). O(k) space.

**Follow-ups:**
- *How would you find even-ID non-boring movies?* → `WHERE id % 2 = 0 AND description <> 'boring'`.
- *What if description could be 'Boring' or 'BORING' (different case)?* → Use `LOWER(description) <> 'boring'` for case-insensitive comparison.


**Example:**

Input `Cinema`:
| id | movie | description | rating |
|---|---|---|---|
| 1 | War | great | 8.9 |
| 2 | Science | fiction | 8.5 |
| 3 | irish | boring | 6.2 |

Output: `id=1, movie=War, rating=8.9` (id 3 is odd but "boring"; id 2 is even)

---

### Q16 — Average Selling Price
**Difficulty:** Easy | **Category:** Aggregate Functions  
**Problem:** For each product, compute average price weighted by units sold. If no units were ever sold, return 0.

**Pattern:** LEFT JOIN with date range condition + weighted average + COALESCE.

**Solution (beats 78.25%):**
```sql
SELECT p.product_id,
  COALESCE(ROUND(SUM(units * price) / SUM(units), 2), 0) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
  ON p.product_id = u.product_id
  AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

**Complexity:** O(n + m) time. O(k) space where k = distinct products.

**Follow-ups:**
- *Why COALESCE(..., 0)?* → If a product has no sales at all, SUM(units) = 0 (NULL after LEFT JOIN), causing division by NULL. COALESCE replaces the NULL result with 0.
- *Why is the date range in the JOIN condition rather than WHERE?* → Putting it in the ON clause allows the LEFT JOIN to still return the product row with NULLs when no sale falls in any valid price window. Putting it in WHERE would turn it into an INNER JOIN behavior.


**Example:**

Input `Prices` (product 1, valid 2019-02-17 to 2019-03-17) / `UnitsSold` (product 1: 20 units on 2019-02-25)

Output: `product_id = 1, average_price = <price weighted by units>` (0 if no matching sales)

---

### Q17 — Project Employees I
**Difficulty:** Easy | **Category:** Aggregate Functions  
**Problem:** For each project, compute the average experience years of employees.

**Pattern:** JOIN + GROUP BY + AVG + ROUND.

**Solution (beats 32.55%):**
```sql
SELECT p.project_id,
  ROUND(AVG(e.experience_years), 2) AS average_years
FROM Project p
JOIN Employee e ON p.employee_id = e.employee_id
GROUP BY p.project_id;
```

**Complexity:** O(n + m) time. O(k) space where k = distinct projects.

**Follow-ups:**
- *What if you also want the project with the highest average experience?* → Wrap in a subquery and add `ORDER BY average_years DESC LIMIT 1`.
- *How would you get the total experience (not average) per project?* → Replace `AVG` with `SUM`.


**Example:**

Input `Project` (project 1: employees 1, 2) / `Employee` (1: 3yrs, 2: 5yrs)

Output: `project_id = 1, average_years = 4.00`

---

### Q18 — Percentage of Users Attended a Contest
**Difficulty:** Easy | **Category:** Aggregate Functions  
**Problem:** For each contest, compute the percentage of all users who registered.

**Pattern:** GROUP BY + scalar subquery for total user count.

**Solution (beats 96.08%):**
```sql
SELECT contest_id,
  ROUND(COUNT(user_id) * 100.0 / (SELECT COUNT(*) FROM Users), 2) AS percentage
FROM Register
GROUP BY contest_id
ORDER BY percentage DESC, contest_id;
```

**Complexity:** O(n + m) time where n = Register rows, m = Users rows. O(k) space where k = distinct contests.

**Follow-ups:**
- *Why multiply by 100.0 (not 100)?* → Forces floating-point division; integer division in some DB engines would truncate to 0 for all fractions.
- *Could you solve this without a subquery?* → Yes, using a cross join: `CROSS JOIN (SELECT COUNT(*) AS total FROM Users) t` then divide by `t.total`.


**Example:**

Input `Users` (2 rows total) / `Register` (contest 1: 1 registrant)

Output: `contest_id = 1, percentage = 50.00`

---

### Q19 — Queries Quality and Percentage
**Difficulty:** Easy | **Category:** Aggregate Functions  
**Problem:** Compute quality (avg of rating/position) and poor_query_percentage (% of queries with rating < 3) per query_name.

**Pattern:** GROUP BY + multiple aggregations using AVG and conditional SUM.

**Solution (beats 96.57%):**
```sql
SELECT query_name,
  ROUND(AVG(rating / position), 2) AS quality,
  ROUND(SUM(IF(rating < 3, 1, 0)) * 100.0 / COUNT(rating), 2) AS poor_query_percentage
FROM Queries
WHERE query_name IS NOT NULL
GROUP BY query_name;
```

**Complexity:** O(n log n) time (GROUP BY sort). O(k) space where k = distinct query names.

**Follow-ups:**
- *What does `IF(rating < 3, 1, 0)` do?* → MySQL shorthand for `CASE WHEN rating < 3 THEN 1 ELSE 0 END`. Returns 1 for poor queries, 0 otherwise — making SUM() count them.
- *Why add `WHERE query_name IS NOT NULL`?* → NULL query names would form a group that may not be meaningful; this filters them out to match expected output.


**Example:**

Input `Queries`:
| query_name | rating | position |
|---|---|---|
| Dog | 5 | 1 |
| Dog | 1 | 3 |

Output: `Dog, quality ≈ 2.67, poor_query_percentage = 50.00`

---

### Q20 — Monthly Transactions I
**Difficulty:** Medium | **Category:** Aggregate Functions  
**Problem:** For each month and country, get: total transaction count, approved count, total amount, approved amount.

**Pattern:** GROUP BY with DATE_FORMAT + conditional SUM (CASE WHEN).

**Solution (beats 97.59%):**
```sql
SELECT DATE_FORMAT(trans_date, '%Y-%m') AS month, country,
  COUNT(*) AS trans_count,
  SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END) AS approved_count,
  SUM(amount) AS trans_total_amount,
  SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total_amount
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```

> **Tip:** Use full expressions in GROUP BY (not positional `GROUP BY 1, 2`) for portability and clarity.

**Complexity:** O(n log n) time. O(k) space where k = distinct month-country combinations.

**Follow-ups:**
- *Why not just filter WHERE state = 'approved' for approved counts?* → That would exclude non-approved transactions from the total count. We need all transactions in the group, then conditionally aggregate the approved ones.
- *How would you also add an approval rate column?* → `ROUND(SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END) / COUNT(*), 2) AS approval_rate`.


**Example:**

Input `Transactions`:
| trans_date | country | state | amount |
|---|---|---|---|
| 2018-12-18 | US | approved | 1000 |
| 2018-12-19 | US | declined | 2000 |

Output: `month=2018-12, country=US, trans_count=2, approved_count=1, trans_total_amount=3000, approved_total_amount=1000`

---

### Q21 — Immediate Food Delivery II
**Difficulty:** Medium | **Category:** Aggregate Functions  
**Problem:** Find the percentage of customers whose FIRST order was an immediate delivery (order_date = customer_pref_delivery_date).

**Pattern:** Filter to first order per customer, then compute percentage.

**Your Solution (beats 30.34%):**
```sql
SELECT ROUND(AVG(temp.order_date = temp.customer_pref_delivery_date) * 100, 2) AS immediate_percentage
FROM (
  SELECT *, RANK() OVER (PARTITION BY customer_id ORDER BY order_date) AS od
  FROM Delivery
) temp
WHERE temp.od = 1;
```

**Claude's Solution (Solution 2) — using MIN subquery:**
```sql
SELECT ROUND(
  SUM(CASE WHEN order_date = customer_pref_delivery_date THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
  2
) AS immediate_percentage
FROM Delivery
WHERE (customer_id, order_date) IN (
  SELECT customer_id, MIN(order_date)
  FROM Delivery
  GROUP BY customer_id
);
```

> Your solution uses RANK() which is correct but a window function adds overhead. The MIN subquery is often faster as it avoids computing ranks for all rows. Note: `AVG(boolean_expression)` works in MySQL because `order_date = order_date` evaluates to 1 or 0 — clever but less readable.

**Complexity:**
- Your solution: O(n log n) for window function
- Claude's solution: O(n log n) for the GROUP BY in subquery

**Follow-ups:**
- *What if two orders are placed on the same day for the same customer (tied first order)?* → Both solutions handle this via MIN date match. RANK() would assign rank=1 to both tied rows; the subquery picks all rows with the minimum date.
- *What if you wanted the percentage for LAST orders instead?* → Change `MIN(order_date)` to `MAX(order_date)`, or `ORDER BY order_date DESC` in the window function.


**Example:**

Input `Delivery` (customer 1's first order: order_date = pref_delivery_date)

Output: `immediate_percentage = 100.00` if that customer's earliest order matched their preferred date.

---

### Q22 — Game Play Analysis IV
**Difficulty:** Medium | **Category:** Aggregate Functions  
**Problem:** Find the fraction of players who logged in again on the day after their first login.

**Pattern:** CTE for first login date + JOIN to check next-day activity.

**Solution (beats 59.70%):**
```sql
WITH first_login AS (
  SELECT player_id, MIN(event_date) AS login_date
  FROM Activity
  GROUP BY player_id
)
SELECT ROUND(
  COUNT(a.player_id) / (SELECT COUNT(*) FROM first_login),
  2
) AS fraction
FROM first_login f
JOIN Activity a
  ON f.player_id = a.player_id
  AND a.event_date = DATE_ADD(f.login_date, INTERVAL 1 DAY);
```

**Complexity:** O(n log n) time (GROUP BY + JOIN). O(k) space where k = distinct players.

**Follow-ups:**
- *Why divide by count of first_login rather than total Activity rows?* → The denominator should be total distinct players, not total activity events.
- *How would you find the fraction who played again within 7 days?* → `AND a.event_date BETWEEN DATE_ADD(f.login_date, INTERVAL 1 DAY) AND DATE_ADD(f.login_date, INTERVAL 7 DAY)`.


**Example:**

Input `Activity`:
| player_id | event_date |
|---|---|
| 1 | 2016-03-01 |
| 1 | 2016-03-02 |
| 2 | 2017-06-25 |

Output: `fraction = 0.50` (1 of 2 players returned the very next day)

---

### Q23 — Number of Unique Subjects Taught by Each Teacher
**Difficulty:** Easy | **Category:** Sorting & Grouping  
**Problem:** Count distinct subjects each teacher teaches (across all departments).

**Pattern:** GROUP BY + COUNT DISTINCT.

**Solution:**
```sql
SELECT teacher_id, COUNT(DISTINCT subject_id) AS cnt
FROM Teacher
GROUP BY teacher_id;
```

**Complexity:** O(n log n) time. O(k) space where k = distinct teachers.

**Follow-ups:**
- *Why COUNT DISTINCT and not just COUNT?* → A teacher might teach the same subject in multiple departments. We want unique subjects.
- *How would you find teachers who teach more than 3 distinct subjects?* → Add `HAVING COUNT(DISTINCT subject_id) > 3`.


**Example:**

Input `Teacher`:
| teacher_id | subject_id | dept_id |
|---|---|---|
| 1 | 2 | 3 |
| 1 | 2 | 4 |

Output: `teacher_id = 1, cnt = 1` (same subject, two departments)

---

### Q24 — User Activity for the Past 30 Days I
**Difficulty:** Easy | **Category:** Sorting & Grouping  
**Problem:** Count distinct active users per day, for days within 30 days of 2019-07-27 (inclusive).

**Pattern:** WHERE date range + GROUP BY day + COUNT DISTINCT.

**Solution (beats 79.07%):**
```sql
SELECT activity_date AS day, COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN DATE_SUB('2019-07-27', INTERVAL 29 DAY) AND '2019-07-27'
GROUP BY activity_date;
```

> `INTERVAL 29 DAY` (not 30) because BETWEEN is inclusive on both ends — 29 days before + the reference day = 30 days total.

**Complexity:** O(n log n) time. O(k) space where k = distinct active days in the window.

**Follow-ups:**
- *Why 29 days in the interval instead of 30?* → BETWEEN is inclusive. Day 0 (2019-07-27) + 29 preceding days = 30 total days.
- *What if you wanted a rolling 7-day active user count per day?* → Use a window function: `COUNT(DISTINCT user_id) OVER (ORDER BY activity_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.


**Example:**

Input `Activity`:
| user_id | activity_date |
|---|---|
| 1 | 2019-07-20 |
| 2 | 2019-07-20 |

Output: `day = 2019-07-20, active_users = 2`

---

### Q25 — Product Sales Analysis III
**Difficulty:** Medium | **Category:** Sorting & Grouping  
**Problem:** For each product, return its FIRST year of sale with the corresponding quantity and price.

**Pattern:** Rank within partition and filter rank = 1.

**Your Solution (beats 36.01%):**
```sql
SELECT product_id, year AS first_year, quantity, price
FROM (
  SELECT product_id, year, quantity, price,
    RANK() OVER (PARTITION BY product_id ORDER BY year ASC) AS result
  FROM Sales
) h
WHERE result = 1;
```

**Claude's Solution (Solution 2) — MIN year subquery join:**
```sql
SELECT product_id, year AS first_year, quantity, price
FROM Sales
WHERE (product_id, year) IN (
  SELECT product_id, MIN(year)
  FROM Sales
  GROUP BY product_id
);
```

> The MIN subquery approach avoids window function overhead and is typically faster. It also naturally handles ties (multiple records in the first year) correctly — returning all of them, same as RANK.

**Complexity:**
- Your solution: O(n log n) for window function
- Claude's solution: O(n log n) for GROUP BY, but usually faster in practice

**Follow-ups:**
- *What if a product has multiple records in its first year?* → Both solutions return all of them (RANK gives rank=1 to tied rows; the IN subquery matches all rows with the minimum year).
- *How would you get the LAST year's data instead?* → Change `MIN(year)` to `MAX(year)` or `ORDER BY year DESC`.


**Example:**

Input `Sales`:
| product_id | year | quantity | price |
|---|---|---|---|
| 100 | 2008 | 10 | 5000 |
| 100 | 2009 | 12 | 5000 |

Output: `product_id=100, first_year=2008, quantity=10, price=5000`

---

### Q26 — Classes With at Least 5 Students
**Difficulty:** Easy | **Category:** Sorting & Grouping  
**Problem:** Find all classes with 5 or more students.

**Pattern:** GROUP BY + HAVING with count threshold.

**Solution (beats 87.94%):**
```sql
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(student) >= 5;
```

**Complexity:** O(n log n) time. O(k) space where k = distinct classes.

**Follow-ups:**
- *Why HAVING and not WHERE?* → WHERE cannot reference aggregate functions. HAVING filters after aggregation.
- *How would you also show the student count next to each class?* → Add `COUNT(student) AS student_count` to the SELECT.


**Example** *(threshold shown as ≥2 for compactness; real problem uses ≥5):*

Input `Courses`:
| student | class |
|---|---|
| A | Math |
| B | Math |
| C | English |

Output: `class = Math`

---

### Q27 — Find Followers Count
**Difficulty:** Easy | **Category:** Sorting & Grouping  
**Problem:** Count distinct followers per user, ordered by user_id.

**Pattern:** GROUP BY + COUNT DISTINCT + ORDER BY.

**Solution (beats 89.65%):**
```sql
SELECT user_id, COUNT(DISTINCT follower_id) AS followers_count
FROM Followers
GROUP BY user_id
ORDER BY user_id ASC;
```

**Complexity:** O(n log n) time. O(k) space where k = distinct users.

**Follow-ups:**
- *Why COUNT DISTINCT?* → The problem says distinct followers; the data might have duplicate (user_id, follower_id) pairs.
- *How would you find users with more than 100 followers?* → Add `HAVING COUNT(DISTINCT follower_id) > 100`.


**Example:**

Input `Followers`:
| user_id | follower_id |
|---|---|
| 0 | 1 |
| 0 | 2 |

Output: `user_id = 0, followers_count = 2`

---

### Q28 — Biggest Single Number
**Difficulty:** Easy | **Category:** Sorting & Grouping  
**Problem:** Find the largest number that appears only once. If no such number exists, return NULL.

**Pattern:** CTE/subquery to filter unique numbers, then MAX.

**Solution (beats 49.72%):**
```sql
WITH singles AS (
  SELECT num FROM MyNumbers GROUP BY num HAVING COUNT(*) = 1
)
SELECT MAX(num) AS num FROM singles;
```

**Complexity:** O(n log n) time. O(k) space where k = distinct numbers.

**Follow-ups:**
- *Why wrap in a CTE/subquery instead of `HAVING COUNT(*) = 1 ORDER BY num DESC LIMIT 1`?* → `LIMIT 1` with no result returns zero rows, not NULL. `MAX()` on an empty set returns NULL as required.
- *What if all numbers appear more than once?* → The CTE is empty, MAX returns NULL. ✓


**Example:**

Input `MyNumbers`: `[8, 8, 3, 3, 1, 4, 5, 6]`

Output: `num = 6` (largest value appearing exactly once)

---

### Q29 — Customers Who Bought All Products
**Difficulty:** Medium | **Category:** Sorting & Grouping  
**Problem:** Find customers who have purchased every product in the Product table.

**Pattern:** GROUP BY + HAVING COUNT DISTINCT = total product count.

**Solution (beats 63.26%):**
```sql
SELECT customer_id
FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (SELECT COUNT(*) FROM Product);
```

**Complexity:** O(n log n + m) time where m = Product rows. O(k) space.

**Follow-ups:**
- *What if a customer bought a product not in the Product table?* → The count would still be compared against total products; extra purchases don't prevent the customer from qualifying.
- *How would you solve this with a NOT EXISTS (relational division) approach?* → `SELECT DISTINCT c.customer_id FROM Customer c WHERE NOT EXISTS (SELECT p.product_key FROM Product p WHERE NOT EXISTS (SELECT 1 FROM Customer c2 WHERE c2.customer_id = c.customer_id AND c2.product_key = p.product_key))`.


**Example:**

Input `Customer` (customer_id, product_key) / `Product` (2 total products: 5, 6):
| customer_id | product_key |
|---|---|
| 1 | 5 |
| 1 | 6 |
| 2 | 5 |

Output: `customer_id = 1`

---

### Q30 — The Number of Employees Which Report to Each Employee
**Difficulty:** Easy | **Category:** Advanced Select  
**Problem:** For each manager, find: number of direct reports and average age (rounded) of those reports.

**Pattern:** Self-join (employee reports to manager) + GROUP BY manager.

**Solution (beats 73.37%):**
```sql
SELECT m.employee_id, m.name, COUNT(e.reports_to) AS reports_count,
  ROUND(AVG(e.age)) AS average_age
FROM Employees e
JOIN Employees m ON e.reports_to = m.employee_id
GROUP BY m.employee_id, m.name
ORDER BY m.employee_id;
```

**Complexity:** O(n log n) time. O(k) space where k = distinct managers.

**Follow-ups:**
- *Why INNER JOIN and not LEFT JOIN here?* → We only want employees who have at least one direct report. If we used LEFT JOIN (from managers side), managers with 0 reports would appear with COUNT=0, which is excluded by the problem.
- *How would you include managers with zero reports?* → Reverse the join direction and use LEFT JOIN from the managers table.


**Example:**

Input `Employees`:
| employee_id | name | reports_to | age |
|---|---|---|---|
| 9 | Hercy | NULL | 43 |
| 6 | Alice | 9 | 41 |
| 4 | Bob | 9 | 36 |

Output: `employee_id=9, name=Hercy, reports_count=2, average_age=39`

---

### Q31 — Primary Department for Each Employee
**Difficulty:** Easy | **Category:** Advanced Select  
**Problem:** Each employee belongs to one or more departments. Return their primary department (flag='Y'), but if they're in only one department, return that one regardless of flag.

**Pattern:** UNION of two cases.

**Your Solution (beats 10.39%):**
```sql
SELECT employee_id, department_id
FROM Employee
WHERE primary_flag = 'Y'
UNION
SELECT employee_id, department_id
FROM Employee
GROUP BY employee_id
HAVING COUNT(employee_id) = 1;
```

**Claude's Solution (Solution 2) — window function approach:**
```sql
SELECT employee_id, department_id
FROM (
  SELECT employee_id, department_id, primary_flag,
    COUNT(*) OVER (PARTITION BY employee_id) AS dept_count
  FROM Employee
) t
WHERE primary_flag = 'Y' OR dept_count = 1;
```

> The window function approach scans the table once. Your UNION approach scans twice. Also, the window version avoids the UNION deduplication overhead. Both produce correct results.

**Complexity:**
- Your solution: O(n log n) — two passes + UNION dedup
- Claude's solution: O(n log n) — single pass with window

**Follow-ups:**
- *Why use UNION (not UNION ALL) in your solution?* → To avoid duplicating employees who are in only one department AND have primary_flag='Y' (they'd match both subqueries).
- *What if an employee has multiple departments all with primary_flag='Y'?* → The problem guarantees at most one 'Y' per employee, but if multiple existed, the first subquery would return all of them.


**Example:**

Input `Employee`:
| employee_id | department_id | primary_flag |
|---|---|---|
| 1 | 1 | N |
| 1 | 2 | Y |
| 2 | 1 | Y |

Output: `(1, 2)` and `(2, 1)` — employee 1's flagged dept, employee 2's only dept.

---

### Q32 — Triangle Judgement
**Difficulty:** Easy | **Category:** Advanced Select  
**Problem:** For each (x, y, z), determine if they can form a triangle. Triangle condition: sum of any two sides > third.

**Pattern:** CASE WHEN / IF for row-level computation.

**Solution:**
```sql
SELECT *, IF(x + y > z AND y + z > x AND z + x > y, 'Yes', 'No') AS triangle
FROM Triangle;
```

**Complexity:** O(n) time. O(n) space.

**Follow-ups:**
- *Why check all three conditions?* → For valid triangle: each side must be less than the sum of the other two. For positive integers, if x ≤ y ≤ z, only `x + y > z` is needed, but checking all three is safe and clear.
- *How would you do this with CASE WHEN?* → `CASE WHEN x + y > z AND y + z > x AND z + x > y THEN 'Yes' ELSE 'No' END`.


**Example:**

Input `Triangle`:
| x | y | z |
|---|---|---|
| 13 | 15 | 30 |
| 10 | 20 | 15 |

Output: `13,15,30 → No`, `10,20,15 → Yes`

---

### Q33 — Consecutive Numbers
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** Find all numbers that appear at least 3 times consecutively (in consecutive rows ordered by id).

**Pattern:** Window functions LAG + LEAD to compare with neighbors.

**Solution (beats 66%):**
```sql
WITH cte AS (
  SELECT id, num,
    LEAD(num) OVER (ORDER BY id) AS next_num,
    LAG(num) OVER (ORDER BY id) AS prev_num
  FROM Logs
)
SELECT DISTINCT num AS ConsecutiveNums
FROM cte
WHERE num = next_num AND num = prev_num;
```

**Complexity:** O(n log n) time (sort for window). O(n) space.

**Follow-ups:**
- *What if you wanted N consecutive occurrences (not just 3)?* → Use multiple LEAD/LAG calls for N-1 steps, or use a self-join approach comparing `id` differences.
- *Why DISTINCT?* → The same number might appear in multiple groups of 3 consecutive rows; DISTINCT returns it once.
- *What if consecutive means same date rather than sequential row?* → Change the ORDER BY in the window function from `id` to the date column.


**Example:**

Input `Logs`:
| id | num |
|---|---|
| 1 | 1 |
| 2 | 1 |
| 3 | 1 |
| 4 | 2 |

Output: `ConsecutiveNums = 1`

---

### Q34 — Product Price at a Given Date
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** Find the price of each product on 2019-08-16. Products with no price change before or on that date default to 10.

**Pattern:** ROW_NUMBER to get latest price change ≤ target date + LEFT JOIN with default.

**Solution (beats 98.35% — Claude's optimal solution):**
```sql
WITH ranked AS (
  SELECT product_id, new_price,
    ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS rnk
  FROM Products
  WHERE change_date <= '2019-08-16'
),
all_products AS (
  SELECT DISTINCT product_id FROM Products
)
SELECT ap.product_id, COALESCE(r.new_price, 10) AS price
FROM all_products ap
LEFT JOIN ranked r ON ap.product_id = r.product_id AND r.rnk = 1;
```

**Complexity:** O(n log n) time (window function sort). O(n) space.

**Follow-ups:**
- *Why ROW_NUMBER over RANK here?* → ROW_NUMBER assigns a unique rank even for ties, ensuring we get exactly one price per product. RANK would return multiple rows if two changes happen on the same date.
- *Why a separate CTE for all_products?* → To capture products that have no price change at or before the target date; they won't appear in the `ranked` CTE (filtered by WHERE), so we LEFT JOIN from the complete product list.
- *Alternative using NOT EXISTS or subquery with MAX?* → `WHERE change_date = (SELECT MAX(change_date) FROM Products p2 WHERE p2.product_id = p1.product_id AND change_date <= '2019-08-16')`.


**Example:**

Input `Products`:
| product_id | new_price | change_date |
|---|---|---|
| 1 | 20 | 2019-08-14 |
| 2 | 50 | 2019-08-14 |
| 1 | 30 | 2019-08-15 |
| 1 | 10 | 2019-08-20 |

Output (as of 2019-08-16): `product_id=1, price=30`, `product_id=2, price=50`

---

### Q35 — Last Person to Fit in the Bus
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** People board a bus in turn order. Bus capacity is 1000kg. Find the last person who can board (cumulative weight ≤ 1000).

**Pattern:** Cumulative SUM window function + filter + ORDER BY + LIMIT 1.

**Solution (beats 59.51%):**
```sql
WITH cte AS (
  SELECT person_name,
    SUM(weight) OVER (ORDER BY turn) AS totalw,
    turn
  FROM Queue
)
SELECT person_name
FROM cte
WHERE totalw <= 1000
ORDER BY turn DESC
LIMIT 1;
```

**Complexity:** O(n log n) time (sort for window). O(n) space.

**Follow-ups:**
- *Why ORDER BY turn DESC LIMIT 1 instead of MAX(turn)?* → We want the person's name, not just the turn number. Ordering by turn descending and taking the first matching row gives us the last person within the weight limit.
- *What if two people board on the same turn?* → The problem guarantees unique turns.


**Example:**

Input `Queue`:
| person_id | person_name | weight | turn |
|---|---|---|---|
| 5 | Alice | 250 | 1 |
| 4 | Bob | 175 | 2 |
| 3 | Alex | 350 | 3 |

Output: `person_name = Alex` (cumulative weight 775 ≤ 1000; add a 4th heavy passenger to exceed it)

---

### Q36 — Count Salary Categories
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** Count accounts in three salary categories: Low (<20000), Average (20000-50000), High (>50000). Return 0 (not NULL) for empty categories.

**Pattern:** UNION ALL with conditional SUM — ensures all three categories always appear.

**Solution (beats 77.35% — Claude's optimal solution):**
```sql
SELECT 'Low Salary' AS category,
  SUM(CASE WHEN income < 20000 THEN 1 ELSE 0 END) AS accounts_count
FROM Accounts
UNION ALL
SELECT 'Average Salary',
  SUM(CASE WHEN income BETWEEN 20000 AND 50000 THEN 1 ELSE 0 END)
FROM Accounts
UNION ALL
SELECT 'High Salary',
  SUM(CASE WHEN income > 50000 THEN 1 ELSE 0 END)
FROM Accounts;
```

**Complexity:** O(n) time (three scans, constant number of rows in output). O(1) output space.

**Follow-ups:**
- *Why not GROUP BY a CASE expression?* → `GROUP BY CASE WHEN income < 20000 THEN 'Low' ... END` would work, but if a category has 0 accounts, it simply won't appear in the result. UNION ALL guarantees all categories appear with 0 if needed.
- *Why UNION ALL over UNION?* → The three categories are mutually exclusive labels — no duplicates are possible. UNION ALL is faster as it skips the deduplication step.


**Example:**

Input `Accounts` (3 rows: incomes 10000, 25000, 60000)

Output:
| category | accounts_count |
|---|---|
| Low Salary | 1 |
| Average Salary | 1 |
| High Salary | 1 |

---

### Q37 — Employees Whose Manager Left the Company
**Difficulty:** Easy | **Category:** Advanced Select  
**Problem:** Find employees (salary < 30000) whose manager has left the company (manager_id not in the current Employees table).

**Pattern:** LEFT JOIN anti-join for missing manager.

**Solution (beats 98.69% — Claude's optimal solution):**
```sql
SELECT e.employee_id
FROM Employees e
LEFT JOIN Employees m ON e.manager_id = m.employee_id
WHERE e.salary < 30000
  AND e.manager_id IS NOT NULL
  AND m.employee_id IS NULL
ORDER BY e.employee_id;
```

**Complexity:** O(n log n) time. O(k) space.

**Follow-ups:**
- *Alternative using NOT IN?* → `WHERE salary < 30000 AND manager_id IS NOT NULL AND manager_id NOT IN (SELECT employee_id FROM Employees)` — be careful: if any employee_id is NULL, NOT IN returns no rows. The LEFT JOIN approach is safer.
- *Why check `manager_id IS NOT NULL`?* → To exclude employees with no manager at all (top-level). We only want employees who HAD a manager that no longer exists.


**Example:**

Input `Employees` (manager_id 13 exists) / employee with `manager_id = 13` doesn't exist in table:

Output: employees with `salary < 30000` and a non-existent `manager_id` — e.g. `employee_id = 11`

---

### Q38 — Exchange Seats
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** Swap every pair of adjacent students by seat. If the last seat is odd-numbered (total is odd), it stays.

**Pattern:** CASE WHEN with modulo and MAX check.

**Solution (beats 69.69%):**
```sql
SELECT
  CASE
    WHEN id = (SELECT MAX(id) FROM Seat) AND id % 2 = 1 THEN id
    WHEN id % 2 = 1 THEN id + 1
    ELSE id - 1
  END AS id,
  student
FROM Seat
ORDER BY id;
```

**Complexity:** O(n log n) time (sort). O(n) space.

**Follow-ups:**
- *How would this work with a window function LEAD/LAG instead?* → `CASE WHEN id % 2 = 1 THEN COALESCE(LEAD(student) OVER (ORDER BY id), student) ELSE LAG(student) OVER (ORDER BY id) END` — swap the student values directly instead of swapping IDs.
- *What if you had groups of 3 seats to rotate instead of 2?* → Use modulo 3 and adjust with +1, +1, -2 pattern.


**Example:**

Input `Seat`:
| id | student |
|---|---|
| 1 | Abbot |
| 2 | Doris |
| 3 | Emerson |

Output:
| id | student |
|---|---|
| 1 | Doris |
| 2 | Abbot |
| 3 | Emerson |

---

### Q39 — Movie Rating
**Difficulty:** Medium | **Category:** Advanced Select  
**Problem:** Part 1: Find the user who rated the most movies (ties broken by name). Part 2: Find the movie with the highest average rating in February 2020 (ties broken by title).

**Pattern:** Two separate GROUP BY queries combined with UNION ALL.

**Solution (beats 99.46%):**
```sql
(SELECT u.name AS results
 FROM MovieRating mr JOIN Users u ON mr.user_id = u.user_id
 GROUP BY mr.user_id
 ORDER BY COUNT(*) DESC, u.name ASC
 LIMIT 1)
UNION ALL
(SELECT m.title AS results
 FROM MovieRating mr JOIN Movies m ON mr.movie_id = m.movie_id
 WHERE mr.created_at BETWEEN '2020-02-01' AND '2020-02-29'
 GROUP BY mr.movie_id
 ORDER BY AVG(mr.rating) DESC, m.title ASC
 LIMIT 1);
```

**Complexity:** O(n log n) time. O(1) output (2 rows).

**Follow-ups:**
- *Why UNION ALL instead of UNION?* → The two results are a user name and a movie title — they serve different purposes but could theoretically be the same string. UNION ALL is safer to avoid accidentally merging them.
- *What does `BETWEEN '2020-02-01' AND '2020-02-29'` do?* → Filters to February 2020 inclusive. Could also use `YEAR(created_at) = 2020 AND MONTH(created_at) = 2`.


**Example:**

Input `MovieRating` / `Users` / `Movies` — one user rates 3 movies (most of anyone); one movie averages highest rating in Feb 2020.

Output (2 rows): `results = <top rater's name>` then `results = <top movie's title>`

---

### Q40 — Restaurant Growth
**Difficulty:** Medium | **Category:** Window Functions  
**Problem:** For each day (starting from the 7th), compute the 7-day rolling sum and average of customer amounts.

**Pattern:** Aggregate daily totals in CTE, then apply rolling window SUM/AVG. Filter to rows with a full 7-day window (rn >= 7).

**Solution (beats 73.19%):**
```sql
WITH daily AS (
  SELECT visited_on, SUM(amount) AS amount
  FROM Customer
  GROUP BY visited_on
),
windowed AS (
  SELECT visited_on,
    SUM(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS amount,
    ROUND(AVG(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS average_amount,
    ROW_NUMBER() OVER (ORDER BY visited_on) AS rn
  FROM daily
)
SELECT visited_on, amount, average_amount
FROM windowed
WHERE rn >= 7
ORDER BY visited_on;
```

**Complexity:** O(n log n) time. O(n) space.

**Follow-ups:**
- *Why aggregate to daily totals first?* → Multiple rows per day would cause the window function to count each row as a separate day, giving wrong results. Aggregating first ensures the window is over actual calendar days.
- *Why `rn >= 7` and not `rn > 6`?* → Same condition; rn >= 7 is more readable as "the 7th day onwards = first complete 7-day window".
- *How would you compute a 30-day rolling average?* → Change `6 PRECEDING` to `29 PRECEDING` and `rn >= 7` to `rn >= 30`.


**Example:**

Input `Customer` — 8 days of visits, each with a daily total amount.

Output: rolling 7-day `amount` and `average_amount` starting from day 7 (the first day with a full week of history).

---

### Q41 — Friend Requests II: Who Has the Most Friends
**Difficulty:** Medium | **Category:** Window Functions  
**Problem:** Each accepted friend request creates a bidirectional friendship. Find who has the most friends total.

**Pattern:** UNION ALL both directions to count each friendship twice (once per person), then GROUP BY and take max.

**Solution (beats 97.98%):**
```sql
SELECT id, COUNT(*) AS num
FROM (
  SELECT requester_id AS id FROM RequestAccepted
  UNION ALL
  SELECT accepter_id AS id FROM RequestAccepted
) combined
GROUP BY id
ORDER BY num DESC
LIMIT 1;
```

**Complexity:** O(n log n) time. O(n) space.

**Follow-ups:**
- *Why UNION ALL and not UNION here?* → UNION would remove duplicate entries for the same pair (if (A,B) and (B,A) both exist). UNION ALL correctly counts both sides of each friendship.
- *What if there's a tie for most friends?* → LIMIT 1 returns only one. To return all tied users, use a subquery: `WHERE num = (SELECT MAX(num) FROM ...)`.


**Example:**

Input `RequestAccepted`:
| requester_id | accepter_id |
|---|---|
| 1 | 2 |
| 1 | 3 |
| 2 | 3 |

Output: `id = 1, num = 2` (most friends)

---

### Q42 — Investments in 2016
**Difficulty:** Medium | **Category:** Subqueries  
**Problem:** Sum tiv_2016 for policyholders who: (1) share tiv_2015 with at least one other policyholder, AND (2) have a unique (lat, lon) combination.

**Pattern:** IN with grouped subqueries for each condition + tuple comparison.

**Solution (beats 55.17%):**
```sql
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (
  SELECT tiv_2015
  FROM Insurance
  GROUP BY tiv_2015
  HAVING COUNT(*) > 1
)
AND (lat, lon) IN (
  SELECT lat, lon
  FROM Insurance
  GROUP BY lat, lon
  HAVING COUNT(*) = 1
);
```

> **MySQL-specific:** `(col1, col2) IN (SELECT col1, col2 ...)` tuple comparison is MySQL-specific and very clean. In other DBs, use `EXISTS` or string concatenation.

**Complexity:** O(n log n) time (two subquery GROUP BYs + outer filter). O(n) space.

**Follow-ups:**
- *How would you rewrite the tuple comparison in standard SQL?* → `EXISTS (SELECT 1 FROM Insurance i2 WHERE i.lat = i2.lat AND i.lon = i2.lon AND i2.pid != i.pid)` but inverted with `NOT EXISTS` for the unique location condition.
- *What does "unique location" mean?* → Exactly one policyholder at that (lat, lon) pair — `HAVING COUNT(*) = 1`.


**Example:**

Input `Insurance`:
| pid | tiv_2015 | tiv_2016 | lat | lon |
|---|---|---|---|---|
| 1 | 10 | 5 | 10 | 10 |
| 2 | 10 | 5 | 20 | 20 |
| 3 | 20 | 5 | 10 | 10 |

Output: `tiv_2016 = 10` (pids 1 and 2 qualify: shared tiv_2015, unique lat/lon)

---

### Q43 — Department Top Three Salaries
**Difficulty:** Hard | **Category:** Subqueries / Window Functions  
**Problem:** Find employees who are in the top 3 salary earners for their department (ties count — if 3 people earn the same highest salary, all 3 are in top 3).

**Pattern:** DENSE_RANK() per department partition + filter rank ≤ 3.

**Solution (beats 85%):**
```sql
WITH RankedSalaries AS (
  SELECT e.name AS employee, e.salary, e.departmentId,
    DENSE_RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) AS salary_rank
  FROM Employee e
)
SELECT d.name AS Department, r.employee AS Employee, r.salary AS Salary
FROM Department d
JOIN RankedSalaries r ON r.departmentId = d.id
WHERE r.salary_rank <= 3;
```

**Complexity:** O(n log n) time (window function + sort). O(n) space.

**Follow-ups:**
- *Why DENSE_RANK over RANK or ROW_NUMBER?* → DENSE_RANK handles ties correctly: if two people share the 2nd highest salary, both get rank 2, and the next distinct salary gets rank 3. RANK would skip to rank 4. ROW_NUMBER would arbitrarily pick one.
- *What if a department has fewer than 3 unique salary levels?* → That's fine — `salary_rank <= 3` will just return all available employees.
- *How would you solve this without window functions (older SQL)?* → `WHERE (SELECT COUNT(DISTINCT e2.salary) FROM Employee e2 WHERE e2.departmentId = e.departmentId AND e2.salary > e.salary) < 3`.


**Example:**

Input `Employee` (dept 1: salaries 90000, 90000, 80000, 70000)

Output: top 3 distinct salary *levels* → all four employees appear (two tie at rank 1).

---

### Q44 — Fix Names in a Table
**Difficulty:** Easy | **Category:** String Functions  
**Problem:** Fix user names so only the first letter is capitalized and the rest are lowercase.

**Pattern:** CONCAT + UPPER(LEFT) + LOWER(RIGHT) string manipulation.

**Solution (beats 79.14%):**
```sql
SELECT user_id,
  CONCAT(UPPER(LEFT(name, 1)), LOWER(RIGHT(name, LENGTH(name) - 1))) AS name
FROM Users
ORDER BY user_id;
```

**Alternative using SUBSTRING:**
```sql
SELECT user_id,
  CONCAT(UPPER(LEFT(name, 1)), LOWER(SUBSTRING(name, 2))) AS name
FROM Users ORDER BY user_id;
```

**Complexity:** O(n) time. O(n) space.

**Follow-ups:**
- *What's the difference between `RIGHT(name, LENGTH(name)-1)` and `SUBSTRING(name, 2)`?* → Both return everything after the first character. SUBSTRING(name, 2) is cleaner and more readable.
- *What if the name is a single character?* → `RIGHT(name, 0)` returns empty string; `SUBSTRING(name, 2)` returns empty string. Both are safe.
- *How would you capitalize every word (title case)?* → MySQL doesn't have a built-in title case function. You'd need a stored function or complex REGEX_REPLACE (MySQL 8.0+).


**Example:**

Input `Users`: `name = 'aLice'`

Output: `name = 'Alice'`

---

### Q45 — Patients With a Condition
**Difficulty:** Easy | **Category:** String Functions  
**Problem:** Find patients with Type I Diabetes (condition code starts with 'DIAB1'). The conditions column is a space-separated list of codes.

**Pattern:** LIKE pattern matching for substring at start of string OR after a space.

**Solution (beats 91.68%):**
```sql
SELECT patient_id, patient_name, conditions
FROM Patients
WHERE conditions LIKE 'DIAB1%' OR conditions LIKE '% DIAB1%';
```

**Complexity:** O(n) time. O(k) space.

**Follow-ups:**
- *Why two LIKE conditions?* → `DIAB1%` catches it as the first code; `% DIAB1%` catches it as any subsequent code (preceded by a space). Together they cover all positions.
- *Why not just `LIKE '%DIAB1%'`?* → That would falsely match codes like 'XDIAB1' or 'RDIAB11'. The word-boundary approach ensures we match only whole codes starting with 'DIAB1'.
- *How would you solve this in MySQL 8.0+ with REGEXP?* → `WHERE conditions REGEXP '(^| )DIAB1'` — cleaner and handles the word boundary in one pattern.


**Example:**

Input `Patients`:
| patient_id | conditions |
|---|---|
| 1 | YFEV COUGH |
| 2 | DIAB100 MYOP |

Output: `patient_id = 2`

---

### Q46 — Delete Duplicate Emails
**Difficulty:** Easy | **Category:** String Functions  
**Problem:** Delete all duplicate email rows, keeping only the row with the smallest id for each email.

**Pattern:** DELETE with subquery — keep minimum id per email group.

**Solution (beats 91.54%):**
```sql
DELETE FROM Person
WHERE id NOT IN (
  SELECT * FROM (SELECT MIN(id) FROM Person GROUP BY email) tmp
);
```

> **MySQL quirk:** You cannot directly reference the same table in a DELETE subquery — you must wrap it in an additional subquery alias (the `tmp` layer). This forces MySQL to materialize the subquery first.

**Complexity:** O(n log n) time (GROUP BY + NOT IN check). O(n) space for subquery result.

**Follow-ups:**
- *Why the extra wrapping `SELECT * FROM (...) tmp`?* → MySQL doesn't allow modifying a table and selecting from it in the same query directly. The nested subquery forces evaluation before the DELETE, avoiding the "can't reopen table" error.
- *Alternative approach with self-join DELETE?* → `DELETE p1 FROM Person p1 JOIN Person p2 WHERE p1.email = p2.email AND p1.id > p2.id` — deletes rows where a smaller id exists for the same email.


**Example:**

Input `Person`:
| id | email |
|---|---|
| 1 | a@b.com |
| 2 | a@b.com |
| 3 | c@d.com |

Output (after DELETE): rows `id = 1, 3` remain.

---

### Q47 — Second Highest Salary
**Difficulty:** Medium | **Category:** Subqueries  
**Problem:** Find the second highest distinct salary. Return NULL if no second highest exists.

**Pattern:** Scalar subquery with DENSE_RANK or LIMIT/OFFSET, wrapped to return NULL if empty.

**Your Solution (beats 56.07%):**
```sql
SELECT (
  SELECT DISTINCT salary
  FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM Employee
  ) s
  WHERE dr = 2
) AS SecondHighestSalary;
```

**Claude's Solution (Solution 2) — cleaner LIMIT/OFFSET approach:**
```sql
SELECT (
  SELECT DISTINCT salary
  FROM Employee
  ORDER BY salary DESC
  LIMIT 1 OFFSET 1
) AS SecondHighestSalary;
```

> Wrapping in a scalar subquery is key — it returns NULL automatically if the inner query produces no rows (e.g., only one distinct salary exists). `LIMIT 1 OFFSET 1` skips the highest and takes the next one.

**Complexity:**
- Both: O(n log n) time (sort). O(1) output space.

**Follow-ups:**
- *Why wrap in an outer SELECT?* → `SELECT DISTINCT ... LIMIT 1 OFFSET 1` alone returns zero rows when no second salary exists. The scalar subquery `SELECT (...)` returns NULL for an empty subquery — matching the expected output.
- *How would you get the Nth highest salary?* → `LIMIT 1 OFFSET N-1` in the inner query.


**Example:**

Input `Employee.salary`: `[100, 200, 300]`

Output: `SecondHighestSalary = 200`

---

### Q48 — Group Sold Products By The Date
**Difficulty:** Easy | **Category:** String Functions  
**Problem:** For each date, count distinct products sold and list them in alphabetical order as a comma-separated string.

**Pattern:** GROUP BY date + COUNT DISTINCT + GROUP_CONCAT.

**Solution (beats 69.05%):**
```sql
SELECT sell_date,
  COUNT(DISTINCT product) AS num_sold,
  GROUP_CONCAT(DISTINCT product ORDER BY product ASC SEPARATOR ',') AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date ASC;
```

**Complexity:** O(n log n) time. O(n) space.

**Follow-ups:**
- *What is GROUP_CONCAT?* → A MySQL aggregate function that concatenates non-NULL values from a group into a single string. You can specify ORDER BY and SEPARATOR.
- *What's the default separator for GROUP_CONCAT?* → A comma `,`. Specifying `SEPARATOR ','` is explicit but redundant here; useful when you need a different separator.
- *Is there a GROUP_CONCAT equivalent in other databases?* → PostgreSQL uses `STRING_AGG(col, ',')`. SQL Server uses `STRING_AGG(col, ',')` (2017+). Both support `WITHIN GROUP (ORDER BY ...)`.


**Example:**

Input `Activities`:
| sell_date | product |
|---|---|
| 2020-05-30 | Headphone |
| 2020-05-30 | Basketball |

Output: `sell_date=2020-05-30, num_sold=2, products='Basketball,Headphone'`

---

### Q49 — List the Products Ordered in a Period
**Difficulty:** Easy | **Category:** String Functions  
**Problem:** Find products with total units ordered ≥ 100 in February 2020.

**Pattern:** JOIN + WHERE date filter + GROUP BY + HAVING.

**Solution (beats 49.83%):**
```sql
SELECT p.product_name, SUM(o.unit) AS unit
FROM Products p
JOIN Orders o USING (product_id)
WHERE YEAR(o.order_date) = 2020 AND MONTH(o.order_date) = 2
GROUP BY p.product_id
HAVING SUM(o.unit) >= 100;
```

**Claude's Solution (Solution 2) — using DATE_FORMAT for clarity:**
```sql
SELECT p.product_name, SUM(o.unit) AS unit
FROM Products p
JOIN Orders o USING (product_id)
WHERE DATE_FORMAT(o.order_date, '%Y-%m') = '2020-02'
GROUP BY p.product_id, p.product_name
HAVING SUM(o.unit) >= 100;
```

**Complexity:** O(n log n) time. O(k) space where k = qualifying products.

**Follow-ups:**
- *Why GROUP BY product_id and not product_name?* → Multiple products could theoretically share the same name. Grouping by the unique key (product_id) is safer.
- *What's the difference between `WHERE YEAR() AND MONTH()` vs `WHERE date BETWEEN '2020-02-01' AND '2020-02-29'`?* → Both work for Feb 2020. The BETWEEN approach is often faster as it can use an index on the date column; YEAR()/MONTH() functions prevent index usage in many databases.
- *What if an order spans multiple months?* → The problem treats order_date as a single day; each row is one order. No spanning issue.


**Example:**

Input `Products` / `Orders` (product 1: 50 units on 2020-02-05, 60 units on 2020-02-10)

Output: `product_name=<name>, unit=110` (≥100 in Feb 2020)

---

### Q50 — Find Users With Valid E-Mails
**Difficulty:** Easy | **Category:** Advanced String Functions  
**Problem:** Find users with valid email addresses matching: starts with a letter, followed by letters/digits/underscores/periods/dashes, ending with @leetcode.com.

**Pattern:** REGEXP / regex matching.

**Your Solution — PostgreSQL (beats 52.35%):**
```sql
SELECT user_id, name, mail
FROM Users
WHERE mail ~ '^[A-Za-z][A-Za-z0-9_.-]*@leetcode\.com$';
```

**MySQL equivalent:**
```sql
SELECT user_id, name, mail
FROM Users
WHERE mail REGEXP '^[A-Za-z][A-Za-z0-9_\\.\\-]*@leetcode\\.com$';
```

> `~` is PostgreSQL regex match operator. MySQL uses `REGEXP` or `RLIKE`. The pattern: `^` = start, `[A-Za-z]` = first char must be a letter, `[A-Za-z0-9_.-]*` = zero or more valid chars, `@leetcode\.com$` = must end with @leetcode.com exactly.

**Complexity:** O(n × L) time where L = average email length. O(k) space.

**Follow-ups:**
- *What does `\.` mean in regex?* → `.` in regex matches any character; `\.` escapes it to match a literal dot.
- *Why anchor with `^` and `$`?* → Without anchors, `REGEXP` would match emails that contain the pattern anywhere (e.g., 'x@leetcode.com@evil.com'). Anchors enforce the pattern matches the entire string.
- *How would you validate emails more strictly (e.g., no consecutive dots)?* → Add a negative lookahead `(?!\.\.)` or use a more complex pattern. SQL regex support varies by database.


**Example:**

Input `Users`:
| user_id | mail |
|---|---|
| 1 | Winston@leetcode.com |
| 2 | annie_______2@leetcode.com@leetcode.com |

Output: `user_id = 1` only (row 2 fails the `$` anchor — extra text after the domain).

---

## Quick Reference — Key Syntax Differences (MySQL vs MS SQL Server)

| Feature | MySQL | MS SQL Server |
|---|---|---|
| String length | `CHAR_LENGTH(s)` | `LEN(s)` |
| Date difference | `DATEDIFF(d1, d2)` → d1-d2 days | `DATEDIFF(unit, d1, d2)` |
| Add to date | `DATE_ADD(d, INTERVAL n DAY)` | `DATEADD(day, n, d)` |
| Format date | `DATE_FORMAT(d, '%Y-%m')` | `FORMAT(d, 'yyyy-MM')` |
| If/else shorthand | `IF(cond, a, b)` | `IIF(cond, a, b)` |
| Regex match | `col REGEXP 'pattern'` | Not built-in (use PATINDEX) |
| Concat strings | `GROUP_CONCAT(col)` | `STRING_AGG(col, ',')` |
| Tuple IN | `(a, b) IN (SELECT ...)` | Not supported (use EXISTS) |
| Limit rows | `LIMIT n` | `TOP n` or `FETCH FIRST n ROWS` |

---

## Quick Reference — Window Function Cheatsheet

```sql
-- Syntax
function_name() OVER (
  [PARTITION BY col1, col2]   -- reset for each group
  [ORDER BY col3 ASC/DESC]    -- defines row order within partition
  [ROWS/RANGE BETWEEN ...]    -- window frame
)

-- Ranking
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)  -- 1,2,3,4 (no ties)
RANK()       OVER (PARTITION BY dept ORDER BY salary DESC)  -- 1,2,2,4 (ties skip)
DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC)  -- 1,2,2,3 (ties no skip)

-- Offset
LAG(col, 1)  OVER (ORDER BY date)   -- previous row's value
LEAD(col, 1) OVER (ORDER BY date)   -- next row's value

-- Aggregate over window
SUM(amount)  OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- rolling sum
AVG(amount)  OVER (PARTITION BY dept)  -- dept average repeated on each row
COUNT(*)     OVER (PARTITION BY user_id)  -- group size on each row
```
