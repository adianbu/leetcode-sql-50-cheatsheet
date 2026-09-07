# Amazon SQL Interview Questions Cheatsheet
**Source:** datavidhya.com — Amazon-tagged SQL questions  
**Coverage:** 60 SQL questions across all difficulty levels  
**Format:** Pattern → Problem → Schema → Example I/O → SQL Solution → Complexity → Follow-ups

---

## Pattern Reference (Quick Index)

| # | Pattern | Questions |
|---|---------|-----------|
| 1 | WHERE Filter | 9, 11 |
| 2 | Aggregate Functions (SUM/AVG/COUNT) | 3, 8, 10, 13, 15, 17, 28, 34 |
| 3 | GROUP BY + HAVING | 2, 3, 4, 18, 29, 30 |
| 4 | INNER/LEFT JOIN | 6, 12, 14, 16, 27, 36, 37, 40 |
| 5 | Self Join | 7, 16, 21, 25, 26, 38 |
| 6 | Subquery | 2, 6, 8 |
| 7 | Window Functions (ROW_NUMBER/RANK/DENSE_RANK) | 20, 24, 31, 32, 42, 43, 46 |
| 8 | LAG / LEAD | 1, 5, 19, 35, 48, 51 |
| 9 | CTE (WITH) | 5, 19, 22, 23, 31, 32, 44 |
| 10 | UNION ALL | 1 |
| 11 | Running Total / Cumulative Sum | 20, 45 |
| 12 | Date Functions | 17, 18, 19, 27 |
| 13 | EXISTS / NOT EXISTS | 39, 40 |
| 14 | Recursive CTE | 21 |
| 15 | Gaps & Islands | 22, 50 |
| 16 | Conditional Aggregation (CASE WHEN) | 14, 47 |
| 17 | COALESCE / NULL Handling | 35, 39 |
| 18 | Anti-Join (LEFT JOIN + IS NULL) | 36, 37, 40 |
| 19 | Rolling Window (ROWS BETWEEN) | 45, 48, 51 |
| 20 | Pivot / CASE WHEN aggregation | 47 |

---

## Q1 — Country GDP Growth Rate
**Difficulty:** Hard  
**Pattern:** UNION ALL + LAG Window Function  
**Tags:** Window Functions, Mathematical Functions

### Problem
Two vendor feeds `gdp_df1` and `gdp_df2` hold country-year GDP history. UNION them and compute YoY growth rate (current − previous) / previous × 100, rounded to 2 decimal places. The earliest year per country returns NULL.

### Schema
```
gdp_df1 / gdp_df2: country (text), year (int), gdp (float)
```

### Example
**Input:**
```
gdp_df1: (USA,2018,20544.34), (USA,2019,21427.7), (China,2018,13894.04)
gdp_df2: (China,2019,14402.72), (India,2018,2713.61), (India,2019,2868.93)
```
**Output:**
```
Country | Year | GDP_growth_rate
China   | 2018 | NULL
China   | 2019 | 3.66
India   | 2018 | NULL
India   | 2019 | 5.72
USA     | 2018 | NULL
USA     | 2019 | 4.30
```

### SQL Solution
```sql
WITH combined AS (
    SELECT country, year, gdp FROM gdp_df1
    UNION ALL
    SELECT country, year, gdp FROM gdp_df2
),
with_lag AS (
    SELECT
        country AS Country,
        year    AS Year,
        gdp,
        LAG(gdp) OVER (PARTITION BY country ORDER BY year) AS prev_gdp
    FROM combined
)
SELECT
    Country,
    Year,
    ROUND((gdp - prev_gdp) / prev_gdp * 100, 2) AS GDP_growth_rate
FROM with_lag
ORDER BY Country, Year;
```

### Complexity
- **Time:** O(n log n) — window sort per country  
- **Space:** O(n) — CTE materialization

### Follow-ups
**Q: What if there are duplicate rows for the same country-year across the two feeds?**  
A: Add a deduplication step: `SELECT country, year, AVG(gdp) AS gdp FROM combined GROUP BY country, year` before the LAG step, or use `SELECT DISTINCT`.

**Q: How would you handle missing years (gaps in the time series)?**  
A: LAG already handles gaps by looking at the immediately preceding row. If you need truly calendar-adjacent years, join against a date spine with `CROSS JOIN UNNEST(SEQUENCE(...))`.

**Q: What does UNION ALL vs UNION do here?**  
A: `UNION ALL` keeps all rows including duplicates; `UNION` deduplicates. The problem requires UNION ALL — if a country-year appears in both feeds, both rows are included.

**Q: How would you compute a 3-year CAGR instead of YoY?**  
A: Use `LAG(gdp, 3)` to get the GDP from 3 years ago, then `(gdp / lag_gdp)^(1/3) - 1`.

---

## Q2 — Duplicate Product Listing Detection
**Difficulty:** Medium  
**Pattern:** GROUP BY + HAVING + Subquery  
**Tags:** Subqueries, Aggregate Functions

### Problem
Count how many distinct vendors in `duplicate_product_listing` have posted ≥ 2 listings with the same `(product_name, description)` pair.

### Schema
```
duplicate_product_listing: listing_id, vendor_id, product_name, description
```

### Example
**Input:**
```
listing_id | vendor_id | product_name      | description
101        | 501       | Bluetooth Speaker | A portable speaker...
102        | 501       | Bluetooth Speaker | A portable speaker...
103        | 502       | Wireless Earbuds  | Comfortable earbuds...
104        | 502       | Wireless Earbuds  | Comfortable earbuds...
```
**Output:**
```
duplicate_vendors
2
```

### SQL Solution
```sql
SELECT COUNT(DISTINCT vendor_id) AS duplicate_vendors
FROM (
    SELECT vendor_id
    FROM duplicate_product_listing
    GROUP BY vendor_id, product_name, description
    HAVING COUNT(*) >= 2
) t;
```

### Complexity
- **Time:** O(n log n) for GROUP BY  
- **Space:** O(d) where d = distinct (vendor, product, desc) groups

### Follow-ups
**Q: Why `COUNT(DISTINCT vendor_id)` in the outer query?**  
A: A vendor could have multiple different duplicate groups; we only want to count that vendor once.

**Q: How would you return vendor IDs along with how many duplicate groups they have?**  
```sql
SELECT vendor_id, COUNT(*) AS duplicate_groups
FROM (
    SELECT vendor_id, product_name, description
    FROM duplicate_product_listing
    GROUP BY vendor_id, product_name, description
    HAVING COUNT(*) >= 2
) t
GROUP BY vendor_id;
```

**Q: What if NULLs appear in product_name or description?**  
A: In SQL, NULL ≠ NULL in GROUP BY — two rows with NULL description are treated as the same group in standard SQL (GROUP BY treats NULLs as equal). Verify this behavior in your DB.

---

## Q3 — Duplicate Job Posting Detection
**Difficulty:** Easy  
**Pattern:** GROUP BY + HAVING + COUNT DISTINCT  
**Tags:** Distinct Functions, Aggregate Functions

### Problem
Count companies in `rjp_listings` that have ≥ 2 job postings with the same `(title, description)`.

### Schema
```
rjp_listings: job_id, company_id, title, description
```

### Example
**Input:**
```
job_id | company_id | title              | description
248    | 827        | Business Analyst   | Business analyst evaluates...
149    | 845        | Business Analyst   | Business analyst evaluates...
945    | 345        | Data Analyst       | Data analyst reviews data...
164    | 345        | Data Analyst       | Data analyst reviews data...
```
**Output:**
```
duplicate_companies
1
```

### SQL Solution
```sql
SELECT COUNT(DISTINCT company_id) AS duplicate_companies
FROM (
    SELECT company_id
    FROM rjp_listings
    GROUP BY company_id, title, description
    HAVING COUNT(*) >= 2
) t;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(d)

### Follow-ups
**Q: What's the difference between this and Q2 (Duplicate Product)?**  
A: Structurally identical — same pattern, different table/columns. Companies vs vendors; job postings vs product listings.

**Q: How do you find the actual duplicate pairs (not just count)?**  
```sql
SELECT company_id, title, description, COUNT(*) AS occurrences
FROM rjp_listings
GROUP BY company_id, title, description
HAVING COUNT(*) >= 2;
```

**Q: How would you delete duplicates and keep only one?**  
```sql
DELETE FROM rjp_listings
WHERE job_id NOT IN (
    SELECT MIN(job_id)
    FROM rjp_listings
    GROUP BY company_id, title, description
);
```

---

## Q4 — Customer Download Trend Analysis
**Difficulty:** Easy  
**Pattern:** Conditional Aggregation + Comparison  
**Tags:** Aggregate Functions

### Problem
Find dates where the number of downloads by free users exceeds downloads by paid users.

### Schema
```
user_downloads: user_id, date, downloads, user_type ('free'/'paid')
```

### Example
**Input:**
```
date       | user_type | downloads
2020-01-01 | free      | 15
2020-01-01 | paid      | 10
2020-01-02 | free      | 8
2020-01-02 | paid      | 12
```
**Output:**
```
date
2020-01-01
```

### SQL Solution
```sql
SELECT date
FROM user_downloads
GROUP BY date
HAVING SUM(CASE WHEN user_type = 'free' THEN downloads ELSE 0 END)
     > SUM(CASE WHEN user_type = 'paid' THEN downloads ELSE 0 END)
ORDER BY date;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(d) where d = distinct dates

### Follow-ups
**Q: How would you also return the download counts for each group?**  
```sql
SELECT date,
       SUM(CASE WHEN user_type='free' THEN downloads ELSE 0 END) AS free_downloads,
       SUM(CASE WHEN user_type='paid' THEN downloads ELSE 0 END) AS paid_downloads
FROM user_downloads
GROUP BY date
HAVING SUM(CASE WHEN user_type='free' THEN downloads ELSE 0 END) 
     > SUM(CASE WHEN user_type='paid' THEN downloads ELSE 0 END);
```

**Q: How would you compute the ratio of free to paid downloads?**  
A: `SUM(free) / NULLIF(SUM(paid), 0)` — use NULLIF to avoid division by zero.

---

## Q5 — Month-over-Month Revenue Change
**Difficulty:** Hard  
**Pattern:** DATE_FORMAT + GROUP BY + LAG Window Function  
**Tags:** CTEs, Window Functions

### Problem
From `cmr_percentage_rev`, compute total revenue per calendar month and the % change vs prior month. Earliest month gets NULL for change.

### Schema
```
cmr_percentage_rev: id, created_at (date), value (float), purchase_id
```

### Example
**Input:**
```
id | created_at | value  | purchase_id
1  | 2019-01-03 | 120000 | 43
2  | 2019-01-20 | 80000  | 36
3  | 2019-02-10 | 250000 | 25
4  | 2019-03-05 | 150000 | 19
5  | 2019-03-22 | 50000  | 12
```
**Output:**
```
year_month | percentage_change
2019-01    | NULL
2019-02    | 25.00
2019-03    | -20.00
```

### SQL Solution
```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS year_month,
        SUM(value) AS total_revenue
    FROM cmr_percentage_rev
    GROUP BY DATE_FORMAT(created_at, '%Y-%m')
),
with_lag AS (
    SELECT
        year_month,
        total_revenue,
        LAG(total_revenue) OVER (ORDER BY year_month) AS prev_revenue
    FROM monthly
)
SELECT
    year_month,
    ROUND((total_revenue - prev_revenue) / prev_revenue * 100, 2) AS percentage_change
FROM with_lag
ORDER BY year_month;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(m) where m = number of distinct months

### Follow-ups
**Q: How does this differ from Q1 (GDP Growth Rate)?**  
A: Same LAG pattern, but here we first aggregate by month with GROUP BY, then apply LAG. In Q1 the data was already one row per country-year.

**Q: How would you handle months with zero transactions (gaps)?**  
A: Join against a date spine (sequence of months) and use COALESCE(total_revenue, 0) for missing months.

**Q: How would you calculate a 3-month rolling average instead?**  
```sql
AVG(total_revenue) OVER (ORDER BY year_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

**Q: What's the difference between ROWS and RANGE in window frames?**  
A: ROWS counts physical rows; RANGE uses the ORDER BY column value and includes all rows with the same value. For revenue calculations, ROWS is usually preferred.

---

## Q6 — Cross-Department Salary Comparison
**Difficulty:** Hard  
**Pattern:** JOIN + Subquery (scalar) + HAVING  
**Tags:** Inner Joins, Subqueries

### Problem
Find departments where the department's average salary is higher than the overall company average salary.

### Schema
```
employees: emp_id, name, salary, department_id
departments: dept_id, dept_name
```

### Example
**Input:**
```
employees: (1,Alice,90000,1),(2,Bob,60000,1),(3,Carol,120000,2),(4,Dave,50000,2)
departments: (1,Engineering),(2,Sales)
```
**Output:**
```
dept_name   | avg_salary
Engineering | 75000
```

### SQL Solution
```sql
SELECT d.dept_name, ROUND(AVG(e.salary), 2) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.dept_id
GROUP BY d.dept_name
HAVING AVG(e.salary) > (SELECT AVG(salary) FROM employees)
ORDER BY avg_salary DESC;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(d) where d = departments

### Follow-ups
**Q: Can you avoid the scalar subquery?**  
```sql
WITH company_avg AS (SELECT AVG(salary) AS avg_sal FROM employees)
SELECT d.dept_name, AVG(e.salary) AS dept_avg
FROM employees e
JOIN departments d ON e.department_id = d.dept_id
CROSS JOIN company_avg
GROUP BY d.dept_name, company_avg.avg_sal
HAVING AVG(e.salary) > company_avg.avg_sal;
```

**Q: How to also show the difference from the company average?**  
A: `AVG(e.salary) - (SELECT AVG(salary) FROM employees) AS diff_from_avg`

---

## Q7 — Japanese City Population Sum
**Difficulty:** Easy  
**Pattern:** WHERE Filter + SUM  
**Tags:** Aggregate Functions

### Problem
Sum the population of all Japanese cities from a `city` table.

### Schema
```
city: id, name, country_code, district, population
```

### Example
**Output:** Single number — total population where country_code = 'JPN'

### SQL Solution
```sql
SELECT SUM(population)
FROM city
WHERE country_code = 'JPN';
```

### Complexity
- **Time:** O(n) — full scan with filter  
- **Space:** O(1)

### Follow-ups
**Q: How would you get the average instead?**  
`SELECT AVG(population) FROM city WHERE country_code = 'JPN'`

**Q: How do you find the top 5 most populated cities in Japan?**  
```sql
SELECT name, population FROM city 
WHERE country_code = 'JPN' 
ORDER BY population DESC LIMIT 5;
```

---

## Q8 — Product Revenue Calculation
**Difficulty:** Easy  
**Pattern:** JOIN + GROUP BY + SUM  
**Tags:** Mathematical Functions

### Problem
Calculate total revenue per product (revenue = units_sold × price).

### Schema
```
sales: sale_id, product_id, units_sold, sale_date
products: product_id, product_name, price
```

### Example
**Output:**
```
product_name | total_revenue
Widget A     | 5000
Widget B     | 3200
```

### SQL Solution
```sql
SELECT p.product_name, SUM(s.units_sold * p.price) AS total_revenue
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.product_name
ORDER BY total_revenue DESC;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(p) where p = products

### Follow-ups
**Q: How would you find products with zero sales?**  
```sql
SELECT p.product_name, COALESCE(SUM(s.units_sold * p.price), 0) AS total_revenue
FROM products p
LEFT JOIN sales s ON p.product_id = s.product_id
GROUP BY p.product_name;
```

---

## Q9 — Eco-Friendly Product Filtering
**Difficulty:** Easy  
**Pattern:** WHERE Filter (multi-condition)  
**Tags:** Filter

### Problem
Find products that are both low-fat AND recyclable.

### Schema
```
Products: product_id, product_name, low_fats (Y/N), recyclable (Y/N)
```

### SQL Solution
```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

### Complexity
- **Time:** O(n)  
- **Space:** O(1)

### Follow-ups
**Q: How would you count by category?**  
```sql
SELECT category, COUNT(*) AS eco_count
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y'
GROUP BY category;
```

---

## Q10 — First Customer Order Analysis
**Difficulty:** Medium  
**Pattern:** GROUP BY + MIN + JOIN  
**Tags:** Aggregate Functions

### Problem
Find each customer's first order (earliest order date) and return customer details.

### Schema
```
orders: order_id, customer_id, order_date, amount
customers: customer_id, name, email
```

### SQL Solution
```sql
SELECT c.customer_id, c.name, o.order_id, o.order_date, o.amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN (
    SELECT customer_id, MIN(order_date) AS first_order_date
    FROM orders
    GROUP BY customer_id
) f ON o.customer_id = f.customer_id AND o.order_date = f.first_order_date;
```

**Alternative using ROW_NUMBER:**
```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS rn
    FROM orders
)
SELECT c.customer_id, c.name, r.order_id, r.order_date, r.amount
FROM customers c
JOIN ranked r ON c.customer_id = r.customer_id
WHERE r.rn = 1;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(n)

### Follow-ups
**Q: What if two orders have the same earliest date for a customer?**  
A: MIN approach returns both; ROW_NUMBER approach returns one (add a tiebreaker like `ORDER BY order_date, order_id`).

---

## Q11 — Highest Revenue Products
**Difficulty:** Medium  
**Pattern:** GROUP BY + SUM + ORDER BY + LIMIT  
**Tags:** Order By

### Problem
Find the top 5 products by total revenue.

### SQL Solution
```sql
SELECT p.product_name, SUM(s.units_sold * p.price) AS total_revenue
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.product_name
ORDER BY total_revenue DESC
LIMIT 5;
```

### Follow-ups
**Q: How do you handle ties at position 5?**  
```sql
-- Use DENSE_RANK to include all tied products
WITH ranked AS (
    SELECT p.product_name, SUM(s.units_sold * p.price) AS total_revenue,
           DENSE_RANK() OVER (ORDER BY SUM(s.units_sold * p.price) DESC) AS rnk
    FROM sales s JOIN products p ON s.product_id = p.product_id
    GROUP BY p.product_name
)
SELECT product_name, total_revenue FROM ranked WHERE rnk <= 5;
```

---

## Q12 — Product Sales Order Analysis
**Difficulty:** Easy  
**Pattern:** JOIN + GROUP BY + Aggregation  
**Tags:** Aggregate Functions

### Problem
Analyze product order history — count orders and total units per product.

### SQL Solution
```sql
SELECT p.product_name,
       COUNT(s.sale_id) AS order_count,
       SUM(s.units_sold) AS total_units
FROM products p
LEFT JOIN sales s ON p.product_id = s.product_id
GROUP BY p.product_name
ORDER BY order_count DESC;
```

---

## Q13 — Highest Paying Job Titles
**Difficulty:** Medium  
**Pattern:** JOIN + GROUP BY + AVG + ORDER BY  
**Tags:** Inner Joins, Mathematical Functions

### Problem
Find job titles ranked by average salary (top earners).

### Schema
```
salaries: emp_id, salary, title
employees: emp_id, name, department
```

### SQL Solution
```sql
SELECT e.title, ROUND(AVG(s.salary), 2) AS avg_salary
FROM salaries s
JOIN employees e ON s.emp_id = e.emp_id
GROUP BY e.title
ORDER BY avg_salary DESC;
```

### Follow-ups
**Q: How do you find the single highest-paying title?**  
Add `LIMIT 1` or wrap in `WHERE rnk = 1` with RANK().

**Q: How do you filter to titles with at least 5 employees?**  
Add `HAVING COUNT(*) >= 5`.

---

## Q14 — Returning User Detection
**Difficulty:** Medium  
**Pattern:** Self Join / LAG on user events  
**Tags:** Mathematical Functions, Self Joins

### Problem
Identify users who made a second purchase within 7 days of any previous purchase (return customers).

### Schema
```
purchases: purchase_id, user_id, purchase_date
```

### SQL Solution
```sql
SELECT DISTINCT a.user_id
FROM purchases a
JOIN purchases b ON a.user_id = b.user_id
  AND b.purchase_date > a.purchase_date
  AND DATEDIFF(b.purchase_date, a.purchase_date) <= 7
ORDER BY a.user_id;
```

**Alternative using LAG:**
```sql
WITH with_prev AS (
    SELECT user_id, purchase_date,
           LAG(purchase_date) OVER (PARTITION BY user_id ORDER BY purchase_date) AS prev_date
    FROM purchases
)
SELECT DISTINCT user_id
FROM with_prev
WHERE DATEDIFF(purchase_date, prev_date) <= 7;
```

### Complexity
- **Time:** O(n²) for self join; O(n log n) for LAG approach  
- **Space:** O(n)

### Follow-ups
**Q: How would you find the time gap between consecutive purchases?**  
A: Use LAG and compute `DATEDIFF(purchase_date, prev_date)`.

---

## Q15 — Shipments Per Month
**Difficulty:** Medium  
**Pattern:** DATE Extraction + COUNT + GROUP BY  
**Tags:** Date/Time Functions, Aggregate Functions

### Problem
Count the number of shipments per year-month, sorted chronologically.

### Schema
```
shipments: shipment_id, ship_date, destination, weight
```

### SQL Solution
```sql
SELECT DATE_FORMAT(ship_date, '%Y-%m') AS year_month,
       COUNT(*) AS shipment_count
FROM shipments
GROUP BY DATE_FORMAT(ship_date, '%Y-%m')
ORDER BY year_month ASC;
```

### Follow-ups
**Q: How do you also include months with zero shipments?**  
A: Generate a date spine and LEFT JOIN.

**Q: How do you get cumulative shipments over time?**  
```sql
WITH monthly AS (
    SELECT DATE_FORMAT(ship_date,'%Y-%m') AS ym, COUNT(*) AS cnt
    FROM shipments GROUP BY 1
)
SELECT ym, cnt, SUM(cnt) OVER (ORDER BY ym) AS cumulative
FROM monthly;
```

---

## Q16 — Cumulative Percentage of Total Orders
**Difficulty:** Medium  
**Pattern:** Window Function + Running Sum  
**Tags:** Window Functions

### Problem
For each product, compute the cumulative percentage of total orders over time using window functions.

### Schema
```
orders: order_id, product_id, order_date, quantity
```

### SQL Solution
```sql
WITH product_orders AS (
    SELECT product_id, order_date, COUNT(*) AS order_count
    FROM orders
    GROUP BY product_id, order_date
),
cumulative AS (
    SELECT
        product_id,
        order_date,
        order_count,
        SUM(order_count) OVER (PARTITION BY product_id ORDER BY order_date) AS cumulative_orders,
        SUM(order_count) OVER (PARTITION BY product_id) AS total_orders
    FROM product_orders
)
SELECT
    product_id,
    order_date,
    ROUND(100.0 * cumulative_orders / total_orders, 2) AS cumulative_pct
FROM cumulative
ORDER BY product_id, order_date;
```

### Follow-ups
**Q: What is PERCENT_RANK() and how does it differ?**  
A: `PERCENT_RANK() = (rank - 1) / (total_rows - 1)` — it's rank-based, not value-based. Our solution is value-based (cumulative sum of actual order counts).

---

## Q17 — Users Who Made Purchases Every Month of 2023
**Difficulty:** Hard  
**Pattern:** DATE Extraction + HAVING COUNT(DISTINCT month) = 12  
**Tags:** Date/Time Functions, Aggregate Functions

### Problem
Find users who made at least one purchase in EVERY month of 2023.

### Schema
```
purchases: purchase_id, user_id, purchase_date, amount
```

### SQL Solution
```sql
SELECT user_id
FROM purchases
WHERE YEAR(purchase_date) = 2023
GROUP BY user_id
HAVING COUNT(DISTINCT MONTH(purchase_date)) = 12
ORDER BY user_id;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(u) where u = users

### Follow-ups
**Q: How would you find users active in at least 6 months?**  
Change `HAVING COUNT(DISTINCT MONTH(purchase_date)) >= 6`

**Q: How would you extend this to identify the months each user was active?**  
```sql
SELECT user_id, GROUP_CONCAT(DISTINCT MONTH(purchase_date) ORDER BY MONTH(purchase_date)) AS active_months
FROM purchases WHERE YEAR(purchase_date) = 2023
GROUP BY user_id;
```

**Q: What if a user made purchases in all 12 months but across multiple years?**  
A: The `WHERE YEAR = 2023` filter prevents this — it restricts to 2023 only.

---

## Q18 — Find Gaps in Sequential IDs
**Difficulty:** Medium  
**Pattern:** LEAD Window Function + Gap Detection  
**Tags:** Window Functions

### Problem
Identify missing IDs in a sequence. Report the start and end of each gap.

### Schema
```
sequential_data: id (integer, sequential)
```

### Example
**Input:** ids = 1, 2, 4, 7, 8, 10  
**Output:**
```
gap_start | gap_end
3         | 3
5         | 6
9         | 9
```

### SQL Solution
```sql
WITH ordered AS (
    SELECT id,
           LEAD(id) OVER (ORDER BY id) AS next_id
    FROM sequential_data
)
SELECT id + 1 AS gap_start, next_id - 1 AS gap_end
FROM ordered
WHERE next_id - id > 1;
```

### Complexity
- **Time:** O(n log n)  
- **Space:** O(n)

### Follow-ups
**Q: How would you count total missing IDs?**  
`SELECT SUM(next_id - id - 1) FROM ordered WHERE next_id - id > 1`

**Q: How do you also detect gaps at the start (if IDs should start from 1)?**  
```sql
SELECT 1 AS gap_start, MIN(id) - 1 AS gap_end FROM sequential_data WHERE MIN(id) > 1
UNION ALL
-- ... the LEAD-based gaps
```

---

## Q19 — Month-over-Month Revenue Change *(see Q5)*

---

## Q20 — Recursive Manager Chain
**Difficulty:** Hard  
**Pattern:** Recursive CTE (Org Hierarchy)  
**Tags:** Recursive CTEs

### Problem
For each employee, traverse up the manager chain and return all ancestor managers (skip-level manager = manager's manager).

### Schema
```
employees: emp_id, name, manager_id (NULL for CEO)
```

### Example
**Input:** Alice→Bob→Carol (Carol is CEO, manager_id=NULL)  
**Output:** Alice's chain: [Bob, Carol]

### SQL Solution
```sql
WITH RECURSIVE org AS (
    -- Base case: all employees
    SELECT emp_id, name, manager_id, 0 AS level
    FROM employees
    UNION ALL
    -- Recursive: join to parent
    SELECT e.emp_id, e.name, m.manager_id, org.level + 1
    FROM org
    JOIN employees m ON org.manager_id = m.emp_id
    WHERE org.manager_id IS NOT NULL
)
SELECT DISTINCT emp_id, name, manager_id AS ancestor_id, level
FROM org
WHERE level > 0
ORDER BY emp_id, level;
```

**Skip-Level Manager (manager's manager) only:**
```sql
SELECT e.emp_id, e.name,
       m.manager_id AS skip_manager_id
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE m.manager_id IS NOT NULL;
```

### Complexity
- **Time:** O(n × d) where d = org depth  
- **Space:** O(n × d)

### Follow-ups
**Q: What prevents infinite loops in recursive CTEs?**  
A: The `WHERE org.manager_id IS NOT NULL` termination condition. Many DBs also have `MAX_RECURSION` settings.

**Q: How do you get the depth of each employee in the org chart?**  
A: The `level` counter in the recursive CTE does this.

---

## Q21 — Identify Price Gouging (3-Sigma Rule)
**Difficulty:** Hard  
**Pattern:** Statistical Window Functions (AVG, STDDEV)  
**Tags:** Window Functions

### Problem
Flag products whose price is more than 3 standard deviations above the category mean (price gouging detection).

### Schema
```
products: product_id, product_name, category, price
```

### SQL Solution
```sql
WITH stats AS (
    SELECT category,
           AVG(price) AS avg_price,
           STDDEV(price) AS std_price
    FROM products
    GROUP BY category
)
SELECT p.product_id, p.product_name, p.category, p.price
FROM products p
JOIN stats s ON p.category = s.category
WHERE p.price > s.avg_price + 3 * s.std_price
ORDER BY p.category, p.price DESC;
```

### Follow-ups
**Q: What is the 3-sigma rule / Empirical Rule?**  
A: In a normal distribution, ~99.7% of values fall within 3 standard deviations of the mean. Values beyond ±3σ are statistical outliers.

**Q: How would you use IQR (interquartile range) instead for non-normal distributions?**  
```sql
WITH percentiles AS (
    SELECT category,
           PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY price) AS q1,
           PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY price) AS q3
    FROM products GROUP BY category
)
SELECT p.* FROM products p
JOIN percentiles pe ON p.category = pe.category
WHERE p.price > pe.q3 + 1.5 * (pe.q3 - pe.q1);  -- IQR outlier
```

---

## Q22 — Late Shipment Impact on Revenue
**Difficulty:** Medium  
**Pattern:** JOIN + Date Comparison + Aggregation  
**Tags:** Date/Time Functions

### Problem
Find total revenue lost from orders where shipment was late (shipped after promised date), broken down by month.

### Schema
```
orders: order_id, customer_id, order_date, promised_date, revenue
shipments: shipment_id, order_id, ship_date
```

### SQL Solution
```sql
SELECT DATE_FORMAT(o.order_date, '%Y-%m') AS month,
       SUM(o.revenue) AS lost_revenue,
       COUNT(*) AS late_orders
FROM orders o
JOIN shipments s ON o.order_id = s.order_id
WHERE s.ship_date > o.promised_date
GROUP BY DATE_FORMAT(o.order_date, '%Y-%m')
ORDER BY month;
```

### Follow-ups
**Q: How do you find the percentage of orders that are late?**  
```sql
SELECT COUNT(CASE WHEN s.ship_date > o.promised_date THEN 1 END) * 100.0 / COUNT(*) AS late_pct
FROM orders o JOIN shipments s ON o.order_id = s.order_id;
```

---

## Q23 — Product Hierarchy and Ranking
**Difficulty:** Medium  
**Pattern:** DENSE_RANK + PARTITION BY  
**Tags:** Window Functions

### Problem
Rank products within each category by total sales volume; return top 3 per category.

### SQL Solution
```sql
WITH ranked AS (
    SELECT p.category, p.product_name,
           SUM(s.units_sold) AS total_units,
           DENSE_RANK() OVER (PARTITION BY p.category ORDER BY SUM(s.units_sold) DESC) AS rnk
    FROM products p
    JOIN sales s ON p.product_id = s.product_id
    GROUP BY p.category, p.product_name
)
SELECT category, product_name, total_units, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY category, rnk;
```

### Follow-ups
**Q: When to use RANK vs DENSE_RANK vs ROW_NUMBER?**  
A:
- `ROW_NUMBER()` — unique rank, no ties (1, 2, 3, 4)
- `RANK()` — ties share rank, gaps after (1, 2, 2, 4)
- `DENSE_RANK()` — ties share rank, no gaps (1, 2, 2, 3) ← use for "top N" with ties

---

## Q24 — Employee Skip-Level Manager
**Difficulty:** Medium  
**Pattern:** Self Join (two levels)  
**Tags:** Self Joins

### Problem
Find each employee's skip-level manager (their manager's manager).

### Schema
```
employees: emp_id, name, manager_id
```

### SQL Solution
```sql
SELECT e.emp_id, e.name,
       m.name AS direct_manager,
       gm.name AS skip_manager
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
JOIN employees gm ON m.manager_id = gm.emp_id;
```

### Follow-ups
**Q: How do you find employees with no manager (top-level)?**  
`WHERE manager_id IS NULL`

**Q: How do you find all employees under a specific manager (full subtree)?**  
Use a recursive CTE traversing downward.

---

## Q25 — Product Cross-Sell Pairs
**Difficulty:** Medium  
**Pattern:** Self Join on orders + GROUP BY pair  
**Tags:** Self Joins, Aggregate Functions

### Problem
Find product pairs that are frequently bought together (co-purchase analysis).

### Schema
```
orders: order_id, product_id, customer_id
```

### SQL Solution
```sql
SELECT a.product_id AS product_1,
       b.product_id AS product_2,
       COUNT(DISTINCT a.order_id) AS co_purchase_count
FROM orders a
JOIN orders b ON a.order_id = b.order_id AND a.product_id < b.product_id
GROUP BY a.product_id, b.product_id
ORDER BY co_purchase_count DESC
LIMIT 10;
```

### Complexity
- **Time:** O(n²) in the worst case (many items per order)  
- **Space:** O(p²) where p = distinct products

### Follow-ups
**Q: Why `a.product_id < b.product_id`?**  
A: To avoid duplicate pairs — (A,B) and (B,A) are the same pair. `<` ensures we only get each pair once.

**Q: How would you find items frequently bought with a specific product?**  
```sql
WHERE a.product_id = 'TARGET_PRODUCT_ID'
```

---

## Q26 — Overdue Project Employees
**Difficulty:** Medium  
**Pattern:** JOIN + Date Comparison  
**Tags:** Date/Time Functions, Aggregate Functions

### Problem
Identify employees working on projects that are overdue (end_date < today but status ≠ complete).

### Schema
```
employees: emp_id, name
projects: project_id, name, end_date, status
employee_projects: emp_id, project_id
```

### SQL Solution
```sql
SELECT DISTINCT e.emp_id, e.name
FROM employees e
JOIN employee_projects ep ON e.emp_id = ep.emp_id
JOIN projects p ON ep.project_id = p.project_id
WHERE p.end_date < CURDATE()
  AND p.status != 'complete'
ORDER BY e.name;
```

---

## Q27 — Average Star Rating Per Product Per Month
**Difficulty:** Easy  
**Pattern:** GROUP BY (product + month) + AVG  
**Tags:** Date/Time Functions, Aggregate Functions

### Problem
Calculate average star ratings for products grouped by product and month.

### Schema
```
reviews: review_id, product_id, stars (1-5), review_date
```

### SQL Solution
```sql
SELECT product_id,
       DATE_FORMAT(review_date, '%Y-%m') AS review_month,
       ROUND(AVG(stars), 2) AS avg_stars
FROM reviews
GROUP BY product_id, DATE_FORMAT(review_date, '%Y-%m')
ORDER BY product_id, review_month;
```

---

## Q28 — Customers with Purchases on Multiple Distinct Days
**Difficulty:** Medium  
**Pattern:** GROUP BY + COUNT(DISTINCT date)  
**Tags:** Date/Time Functions, Aggregate Functions

### Problem
Identify customers who made at least one purchase on 3 or more distinct dates.

### SQL Solution
```sql
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING COUNT(DISTINCT purchase_date) >= 3
ORDER BY customer_id;
```

### Follow-ups
**Q: How would you also return the list of purchase dates?**  
```sql
SELECT customer_id, GROUP_CONCAT(DISTINCT purchase_date ORDER BY purchase_date) AS dates
FROM purchases
GROUP BY customer_id
HAVING COUNT(DISTINCT purchase_date) >= 3;
```

---

## Q29 — Customers Who Bought Only a Single Item
**Difficulty:** Easy  
**Pattern:** GROUP BY + HAVING COUNT  
**Tags:** Inner Joins, Aggregate Functions

### Problem
Find customers who have exactly one distinct product in their purchase history.

### SQL Solution
```sql
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = 1
ORDER BY customer_id;
```

---

## Q30 — Category Sales Metrics
**Difficulty:** Medium  
**Pattern:** GROUP BY + Multiple Aggregations  
**Tags:** Aggregate Functions

### Problem
Compute total revenue, average order value, and order count per product category.

### SQL Solution
```sql
SELECT p.category,
       SUM(s.units_sold * p.price) AS total_revenue,
       ROUND(AVG(s.units_sold * p.price), 2) AS avg_order_value,
       COUNT(s.sale_id) AS order_count
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;
```

---

## Q31 — Active Prime Members by Marketplace
**Difficulty:** Hard  
**Pattern:** CTE + GROUP BY + Conditional Filter  
**Tags:** CTEs, Date/Time Functions

### Problem
Find number of active Prime members as of end of 2020 per marketplace.

### Schema
```
prime_memberships: member_id, marketplace, start_date, end_date
```

### SQL Solution
```sql
SELECT marketplace, COUNT(*) AS active_members
FROM prime_memberships
WHERE start_date <= '2020-12-31'
  AND (end_date IS NULL OR end_date > '2020-12-31')
GROUP BY marketplace
ORDER BY marketplace;
```

### Follow-ups
**Q: How do you track monthly active member count over time?**  
A: Generate a date spine of months and join to memberships using BETWEEN.

---

## Q32 — Maximize Prime Item Inventory
**Difficulty:** Hard  
**Pattern:** Knapsack-style — ORDER BY priority + LIMIT  
**Tags:** Conditional Logic, CTEs

### Problem
A 500k sq-ft warehouse fills prime batches first (up to 500k), then non-prime with remaining space.

### Schema
```
inventory: item_type ('prime'/'non_prime'), square_footage
```

### SQL Solution
```sql
WITH space AS (
    SELECT
        SUM(CASE WHEN item_type = 'prime_eligible' THEN square_footage ELSE 0 END) AS prime_total,
        SUM(CASE WHEN item_type = 'not_prime' THEN square_footage ELSE 0 END) AS np_total,
        500000 AS capacity
),
prime_used AS (
    SELECT LEAST(prime_total, capacity) AS prime_space_used,
           capacity - LEAST(prime_total, capacity) AS remaining
    FROM space
)
SELECT 'prime_eligible' AS item_type,
       FLOOR(prime_space_used / (SELECT AVG(square_footage) FROM inventory WHERE item_type='prime_eligible')) AS item_count
FROM prime_used
UNION ALL
SELECT 'not_prime',
       FLOOR(remaining / (SELECT AVG(square_footage) FROM inventory WHERE item_type='not_prime'))
FROM prime_used;
```

---

## Q33 — Department Salary Aggregation
**Difficulty:** Medium  
**Pattern:** GROUP BY + Multiple Aggregations  
**Tags:** Aggregate Functions

### Problem
Aggregate employee salary data per department: total, average, min, max.

### SQL Solution
```sql
SELECT department,
       COUNT(*) AS headcount,
       SUM(salary) AS total_salary,
       ROUND(AVG(salary), 2) AS avg_salary,
       MIN(salary) AS min_salary,
       MAX(salary) AS max_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;
```

---

## Q34 — Neighborhoods with Zero Users
**Difficulty:** Easy  
**Pattern:** Anti-Join (LEFT JOIN + WHERE IS NULL)  
**Tags:** Inner Joins, Null Handling

### Problem
Find all neighborhoods that have no users in the users table.

### Schema
```
neighborhoods: neighborhood_id, neighborhood_name, city_id
users: user_id, neighborhood_id
```

### SQL Solution
```sql
SELECT n.neighborhood_name
FROM neighborhoods n
LEFT JOIN users u ON n.neighborhood_id = u.neighborhood_id
WHERE u.neighborhood_id IS NULL
ORDER BY n.neighborhood_name;
```

### Follow-ups
**Q: What's the alternative using NOT EXISTS?**  
```sql
SELECT neighborhood_name FROM neighborhoods n
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.neighborhood_id = n.neighborhood_id);
```

**Q: When is LEFT JOIN + IS NULL faster than NOT EXISTS?**  
A: Generally NOT EXISTS is better when the subquery is correlated and an index exists; LEFT JOIN can be faster for large outer tables with few nulls. Plan accordingly.

---

## Q35 — Product Groups with No Sales in US
**Difficulty:** Medium  
**Pattern:** Anti-Join + Filter  
**Tags:** Inner Joins, Null Handling

### Problem
Find product groups that had no sales in the US region.

### SQL Solution
```sql
SELECT DISTINCT p.product_group
FROM products p
WHERE p.product_group NOT IN (
    SELECT DISTINCT p2.product_group
    FROM products p2
    JOIN sales s ON p2.product_id = s.product_id
    WHERE s.region = 'US'
);
```

**Alternative using LEFT JOIN:**
```sql
SELECT DISTINCT p.product_group
FROM products p
LEFT JOIN (
    SELECT DISTINCT p2.product_group
    FROM products p2
    JOIN sales s ON p2.product_id = s.product_id
    WHERE s.region = 'US'
) us_sales ON p.product_group = us_sales.product_group
WHERE us_sales.product_group IS NULL;
```

---

## Q36 — Returning Active Users Within 7 Days
**Difficulty:** Medium  
**Pattern:** Self Join + DATEDIFF  
**Tags:** Inner Joins, Date/Time Functions

### Problem
Find users who made another purchase within 7 days of a previous purchase.

### SQL Solution
```sql
SELECT DISTINCT a.user_id
FROM purchases a
JOIN purchases b
  ON a.user_id = b.user_id
 AND b.purchase_date BETWEEN a.purchase_date + INTERVAL 1 DAY
                         AND a.purchase_date + INTERVAL 7 DAY
ORDER BY a.user_id;
```

---

## Q37 — Find Products Never Reviewed
**Difficulty:** Easy  
**Pattern:** Anti-Join (LEFT JOIN + IS NULL)  
**Tags:** Inner Joins, Null Handling

### Problem
Find products that have no reviews in the reviews table.

### SQL Solution
```sql
SELECT p.product_id, p.product_name
FROM products p
LEFT JOIN reviews r ON p.product_id = r.product_id
WHERE r.product_id IS NULL
ORDER BY p.product_id;
```

---

## Q38 — User Repeat Purchases (Self Join)
**Difficulty:** Medium  
**Pattern:** Self Join on purchases  
**Tags:** Self Joins, Date/Time Functions

### Problem
Find users who made a purchase of the SAME product more than once.

### SQL Solution
```sql
SELECT customer_id, product_id, COUNT(*) AS purchase_count
FROM purchases
GROUP BY customer_id, product_id
HAVING COUNT(*) > 1
ORDER BY purchase_count DESC;
```

---

## Q39 — Highest Cost Orders
**Difficulty:** Medium  
**Pattern:** JOIN + GROUP BY + ORDER BY  
**Tags:** Aggregate Functions

### Problem
For each customer, find their highest cost order (max order value).

### SQL Solution
```sql
SELECT c.customer_id, c.name,
       MAX(o.total_amount) AS highest_order
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
ORDER BY highest_order DESC;
```

---

## Q40 — Product Recommendation Based on Co-Purchase
**Difficulty:** Hard  
**Pattern:** Self Join (co-occurrence) + COUNT  
**Tags:** Inner Joins, Aggregate Functions

### Problem
Recommend products by finding what else customers buy when they buy product X.

### SQL Solution
```sql
-- Find what's bought with product X
SELECT b.product_id AS recommended,
       COUNT(*) AS co_purchases
FROM orders a
JOIN orders b ON a.order_id = b.order_id
  AND a.product_id = 'PRODUCT_X'
  AND b.product_id != 'PRODUCT_X'
GROUP BY b.product_id
ORDER BY co_purchases DESC
LIMIT 5;
```

---

## Q41 — NULL Value Handling in Joins
**Difficulty:** Medium  
**Pattern:** COALESCE + LEFT JOIN  
**Tags:** Null Handling, Inner Joins

### Problem
Join employee and salary tables; return 0 for employees with no salary record.

### SQL Solution
```sql
SELECT e.emp_id, e.name,
       COALESCE(s.salary, 0) AS salary
FROM employees e
LEFT JOIN salaries s ON e.emp_id = s.emp_id
ORDER BY e.emp_id;
```

### Follow-ups
**Q: What is COALESCE?**  
A: Returns the first non-NULL value in its argument list. `COALESCE(a, b, c)` → a if not null, else b, else c.

**Q: What's the difference between COALESCE and IFNULL?**  
A: IFNULL takes exactly 2 args; COALESCE takes unlimited args. Both are equivalent for 2-arg use.

---

## Q42 — Semi-Join Existence Check
**Difficulty:** Medium  
**Pattern:** EXISTS subquery  
**Tags:** Subqueries

### Problem
Return customers who have placed at least one order (semi-join pattern).

### SQL Solution
```sql
-- EXISTS (semi-join)
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- Equivalent with INNER JOIN (but may duplicate customers)
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
```

### Follow-ups
**Q: Why use EXISTS over IN?**  
A: EXISTS short-circuits at first match (faster for large subquery results); IN materializes the entire subquery first. EXISTS also handles NULLs better.

---

## Q43 — Consecutive Numbers (3+ Times in a Row)
**Difficulty:** Medium  
**Pattern:** LAG + Self Join or ROW_NUMBER - value trick  
**Tags:** Window Functions

### Problem
Find all numbers that appear consecutively at least 3 times in the `Logs` table.

### Schema
```
Logs: id, num
```

### SQL Solution
```sql
SELECT DISTINCT l1.num AS ConsecutiveNums
FROM Logs l1
JOIN Logs l2 ON l1.id + 1 = l2.id AND l1.num = l2.num
JOIN Logs l3 ON l1.id + 2 = l3.id AND l1.num = l3.num;
```

**Alternative using ROW_NUMBER trick:**
```sql
WITH numbered AS (
    SELECT num,
           ROW_NUMBER() OVER (ORDER BY id) - 
           ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM Logs
)
SELECT DISTINCT num AS ConsecutiveNums
FROM numbered
GROUP BY num, grp
HAVING COUNT(*) >= 3;
```

---

## Q44 — Running Total Sales
**Difficulty:** Medium  
**Pattern:** SUM OVER (ORDER BY) — Running Total  
**Tags:** Window Functions

### Problem
Compute the cumulative total sales amount by date.

### SQL Solution
```sql
SELECT sale_date, daily_total,
       SUM(daily_total) OVER (ORDER BY sale_date) AS running_total
FROM (
    SELECT sale_date, SUM(amount) AS daily_total
    FROM sales GROUP BY sale_date
) t
ORDER BY sale_date;
```

---

## Q45 — Top N Products Per Category
**Difficulty:** Medium  
**Pattern:** DENSE_RANK + PARTITION BY + WHERE rnk <= N  
**Tags:** Window Functions

### Problem
Find the top 3 products by revenue in each category.

### SQL Solution
```sql
WITH ranked AS (
    SELECT p.category, p.product_name,
           SUM(s.units_sold * p.price) AS revenue,
           DENSE_RANK() OVER (PARTITION BY p.category ORDER BY SUM(s.units_sold * p.price) DESC) AS rnk
    FROM products p JOIN sales s ON p.product_id = s.product_id
    GROUP BY p.category, p.product_name
)
SELECT category, product_name, revenue
FROM ranked WHERE rnk <= 3
ORDER BY category, rnk;
```

---

## Q46 — Nth Highest Salary
**Difficulty:** Medium  
**Pattern:** DENSE_RANK or LIMIT/OFFSET  
**Tags:** Window Functions

### Problem
Find the Nth highest salary from the `Employee` table.

### SQL Solution
```sql
-- Using DENSE_RANK (handles ties, works across all DBs)
SELECT DISTINCT salary
FROM (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM Employee
) t
WHERE rnk = N;

-- Using LIMIT/OFFSET (simple, MySQL/PostgreSQL)
SELECT DISTINCT salary FROM Employee
ORDER BY salary DESC
LIMIT 1 OFFSET N-1;
```

### Follow-ups
**Q: What if fewer than N distinct salaries exist?**  
A: Both approaches return NULL/empty. Wrap in `SELECT IFNULL((subquery), NULL)`.

---

## Q47 — Alternate Records
**Difficulty:** Easy  
**Pattern:** ROW_NUMBER + Modulo  
**Tags:** Window Functions

### Problem
Return every other row (odd-position rows) from a result set.

### SQL Solution
```sql
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn
    FROM table_name
) t
WHERE rn % 2 = 1;  -- 1 for odd rows, 0 for even
```

---

## Q48 — International Phone Calls Percentage
**Difficulty:** Medium  
**Pattern:** Conditional Aggregation  
**Tags:** Aggregate Functions

### Problem
Find percentage of calls that are international (caller and receiver in different countries).

### Schema
```
phone_calls: caller_id, receiver_id, duration
users: user_id, country
```

### SQL Solution
```sql
SELECT ROUND(
    100.0 * SUM(CASE WHEN c.country != r.country THEN 1 ELSE 0 END) / COUNT(*),
    2
) AS intl_call_pct
FROM phone_calls pc
JOIN users c ON pc.caller_id = c.user_id
JOIN users r ON pc.receiver_id = r.user_id;
```

---

## Q49 — Highest Grossing Items Per Category
**Difficulty:** Hard  
**Pattern:** DENSE_RANK + PARTITION BY category  
**Tags:** Window Functions

### Problem
Find the top 2 highest-grossing items per category in 2022.

### Schema
```
product_spend: category, product, user_id, spend, transaction_date
```

### SQL Solution
```sql
WITH category_totals AS (
    SELECT category, product, SUM(spend) AS total_spend
    FROM product_spend
    WHERE YEAR(transaction_date) = 2022
    GROUP BY category, product
),
ranked AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY category ORDER BY total_spend DESC) AS rnk
    FROM category_totals
)
SELECT category, product, total_spend
FROM ranked WHERE rnk <= 2
ORDER BY category, rnk;
```

---

## Q50 — Top Two Products Per Category
**Difficulty:** Hard  
**Pattern:** DENSE_RANK <= 2 + PARTITION BY  
**Tags:** Window Functions

*See Q49 — same pattern with `rnk <= 2`.*

---

## Q51 — 3-Month Rolling Average Revenue
**Difficulty:** Hard  
**Pattern:** AVG OVER ROWS BETWEEN 2 PRECEDING  
**Tags:** Window Functions

### Problem
Compute the 3-month rolling average of monthly revenue.

### SQL Solution
```sql
WITH monthly AS (
    SELECT DATE_FORMAT(sale_date, '%Y-%m') AS yr_month,
           SUM(amount) AS monthly_revenue
    FROM sales
    GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
)
SELECT yr_month, monthly_revenue,
       ROUND(AVG(monthly_revenue) OVER (
           ORDER BY yr_month
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ), 2) AS rolling_3m_avg
FROM monthly
ORDER BY yr_month;
```

### Follow-ups
**Q: What's the difference between ROWS BETWEEN 2 PRECEDING and RANGE BETWEEN INTERVAL '2' MONTH PRECEDING?**  
A: ROWS counts physical rows; RANGE uses the actual value range. For monthly data they're equivalent only if no months are missing. Use ROWS for physical windows, RANGE for value-based.

---

## Q52 — Products with 3 Consecutive Months of Increasing Sales
**Difficulty:** Hard  
**Pattern:** LAG + Comparison Chain  
**Tags:** Window Functions

### Problem
Find products whose monthly sales increased for 3 consecutive months.

### SQL Solution
```sql
WITH monthly AS (
    SELECT product_id,
           DATE_FORMAT(sale_date, '%Y-%m') AS yr_month,
           SUM(amount) AS monthly_sales
    FROM sales GROUP BY product_id, DATE_FORMAT(sale_date, '%Y-%m')
),
with_lags AS (
    SELECT product_id, yr_month, monthly_sales,
           LAG(monthly_sales, 1) OVER (PARTITION BY product_id ORDER BY yr_month) AS prev1,
           LAG(monthly_sales, 2) OVER (PARTITION BY product_id ORDER BY yr_month) AS prev2
    FROM monthly
)
SELECT DISTINCT product_id
FROM with_lags
WHERE monthly_sales > prev1 AND prev1 > prev2;
```

---

## Q53 — Revenue Anomaly Detection (±200% of Moving Average)
**Difficulty:** Hard  
**Pattern:** Rolling Average + Threshold Comparison  
**Tags:** Window Functions

### Problem
Flag days where revenue is more than 200% above the 7-day moving average.

### SQL Solution
```sql
WITH daily AS (
    SELECT sale_date, SUM(amount) AS revenue
    FROM sales GROUP BY sale_date
),
with_avg AS (
    SELECT sale_date, revenue,
           AVG(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING) AS rolling_avg
    FROM daily
)
SELECT sale_date, revenue, rolling_avg
FROM with_avg
WHERE revenue > 2 * rolling_avg
ORDER BY sale_date;
```

---

## Q54 — Server Utilization Time
**Difficulty:** Hard  
**Pattern:** Date/Time Overlap + SUM  
**Tags:** Window Functions, Date/Time Functions

### Problem
Calculate total uptime for each server from start/stop event logs.

### Schema
```
server_events: server_id, event_type ('start'/'stop'), timestamp
```

### SQL Solution
```sql
WITH sessions AS (
    SELECT server_id,
           timestamp AS start_time,
           LEAD(timestamp) OVER (PARTITION BY server_id ORDER BY timestamp) AS stop_time,
           event_type
    FROM server_events
)
SELECT server_id,
       SUM(TIMESTAMPDIFF(SECOND, start_time, stop_time)) AS total_uptime_seconds
FROM sessions
WHERE event_type = 'start' AND stop_time IS NOT NULL
GROUP BY server_id;
```

---

## Q55 — Top 3 Month-over-Month Revenue Growth
**Difficulty:** Hard  
**Pattern:** LAG + RANK on growth rate  
**Tags:** Window Functions

### Problem
Find the top 3 months with the highest revenue growth rate compared to the prior month.

### SQL Solution
```sql
WITH monthly AS (
    SELECT DATE_FORMAT(sale_date, '%Y-%m') AS yr_month,
           SUM(amount) AS revenue
    FROM sales GROUP BY 1
),
growth AS (
    SELECT yr_month, revenue,
           LAG(revenue) OVER (ORDER BY yr_month) AS prev_revenue,
           ROUND((revenue - LAG(revenue) OVER (ORDER BY yr_month)) 
                 / LAG(revenue) OVER (ORDER BY yr_month) * 100, 2) AS growth_pct
    FROM monthly
)
SELECT yr_month, revenue, growth_pct
FROM growth
WHERE growth_pct IS NOT NULL
ORDER BY growth_pct DESC
LIMIT 3;
```

---

## Q56 — Reorder Point Stock Calculation
**Difficulty:** Medium  
**Pattern:** Aggregation + Business Rule Threshold  
**Tags:** Aggregate Functions

### Problem
Flag products that need reordering (current stock < reorder_point = avg_daily_sales × lead_time_days).

### Schema
```
inventory: product_id, current_stock, lead_time_days
sales: product_id, sale_date, units_sold
```

### SQL Solution
```sql
WITH daily_avg AS (
    SELECT product_id, AVG(units_sold) AS avg_daily_sales
    FROM sales GROUP BY product_id
)
SELECT i.product_id,
       i.current_stock,
       ROUND(d.avg_daily_sales * i.lead_time_days, 0) AS reorder_point,
       CASE WHEN i.current_stock < d.avg_daily_sales * i.lead_time_days
            THEN 'REORDER' ELSE 'OK' END AS status
FROM inventory i
JOIN daily_avg d ON i.product_id = d.product_id
ORDER BY i.product_id;
```

---

## Q57 — Warehouse Inventory Optimization
**Difficulty:** Hard  
**Pattern:** GROUP BY + ORDER BY + Capacity Logic  
**Tags:** Subqueries, Order By

### Problem
Maximize items stored in a warehouse by prioritizing item types. Fill prime-eligible items first (up to capacity), then fill remaining space with non-prime.

### SQL Solution
```sql
WITH totals AS (
    SELECT
        SUM(CASE WHEN item_type = 'prime_eligible' THEN square_footage ELSE 0 END) AS prime_sqft,
        SUM(CASE WHEN item_type = 'not_prime' THEN square_footage ELSE 0 END) AS np_sqft
    FROM inventory
),
allocation AS (
    SELECT
        LEAST(prime_sqft, 500000) AS prime_alloc,
        GREATEST(500000 - LEAST(prime_sqft, 500000), 0) AS remaining
    FROM totals
)
SELECT
    'prime_eligible' AS item_type,
    FLOOR(prime_alloc / (SELECT square_footage FROM inventory WHERE item_type='prime_eligible' LIMIT 1)) AS items
FROM allocation
UNION ALL
SELECT 'not_prime',
    FLOOR(remaining / (SELECT square_footage FROM inventory WHERE item_type='not_prime' LIMIT 1))
FROM allocation;
```

---

## Q58 — Kimball E-Commerce Star Schema Query
**Difficulty:** Hard  
**Pattern:** Star Schema JOIN (fact + dimension tables)  
**Tags:** Aggregate Functions, Inner Joins

### Problem
Query a star schema to compute total revenue by region and product category for Q4 2022.

### Schema
```
fact_orders: order_id, date_key, product_key, customer_key, revenue
dim_date: date_key, year, quarter, month
dim_product: product_key, product_name, category
dim_customer: customer_key, customer_name, region
```

### SQL Solution
```sql
SELECT
    dc.region,
    dp.category,
    SUM(fo.revenue) AS total_revenue
FROM fact_orders fo
JOIN dim_date dd ON fo.date_key = dd.date_key
JOIN dim_product dp ON fo.product_key = dp.product_key
JOIN dim_customer dc ON fo.customer_key = dc.customer_key
WHERE dd.year = 2022 AND dd.quarter = 4
GROUP BY dc.region, dp.category
ORDER BY dc.region, total_revenue DESC;
```

### Follow-ups
**Q: What is a Star Schema?**  
A: A central fact table (transactions) surrounded by dimension tables (date, product, customer). Facts contain foreign keys to dimensions + measures (revenue, quantity).

**Q: What is a Snowflake Schema?**  
A: Like Star but dimensions are normalized (e.g., product → category → department tables). More storage-efficient but requires more joins.

---

## Q59 — Forward Fill Sensor Readings
**Difficulty:** Easy  
**Pattern:** LAST_VALUE IGNORE NULLS or LAG chain  
**Tags:** Window Functions, Null Handling

### Problem
Replace NULL values in sensor readings with the most recent non-NULL value (LOCF — Last Observation Carried Forward).

### Schema
```
sensor_readings: reading_id, sensor_id, reading_time, value (may be NULL)
```

### SQL Solution (PostgreSQL / Spark SQL — supports IGNORE NULLS)
```sql
SELECT reading_id, sensor_id, reading_time,
       LAST_VALUE(value IGNORE NULLS) OVER (
           PARTITION BY sensor_id
           ORDER BY reading_time
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS filled_value
FROM sensor_readings;
```

**MySQL alternative (LAG chain):**
```sql
SELECT reading_id, sensor_id, reading_time,
       COALESCE(value,
           LAG(value, 1) IGNORE NULLS OVER (PARTITION BY sensor_id ORDER BY reading_time),
           LAG(value, 2) IGNORE NULLS OVER (PARTITION BY sensor_id ORDER BY reading_time)
       ) AS filled_value
FROM sensor_readings;
```

### Follow-ups
**Q: What is LOCF?**  
A: Last Observation Carried Forward — a common imputation technique for time series data where missing values are replaced with the last valid observation.

**Q: How do you do forward-fill AND backward-fill?**  
A: LOCF forward-fills; use `FIRST_VALUE ... FOLLOWING` for backward fill (NOCB — next observation carried backward).

---

## Q60 — Peak Concurrent Users
**Difficulty:** Hard  
**Pattern:** Event Overlap / Timeline Sweep  
**Tags:** Window Functions

### Problem
Find the maximum number of concurrent active sessions at any point in time.

### Schema
```
sessions: session_id, user_id, start_time, end_time
```

### SQL Solution
```sql
WITH events AS (
    SELECT start_time AS event_time, 1 AS delta FROM sessions
    UNION ALL
    SELECT end_time, -1 FROM sessions
),
running AS (
    SELECT event_time,
           SUM(delta) OVER (ORDER BY event_time, delta DESC) AS concurrent_users
    FROM events
)
SELECT MAX(concurrent_users) AS peak_concurrent_users
FROM running;
```

### Follow-ups
**Q: How would you find the time window when peak occurred?**  
```sql
SELECT event_time, concurrent_users
FROM running
WHERE concurrent_users = (SELECT MAX(concurrent_users) FROM running);
```

---

## Summary: SQL Patterns by Question Difficulty

### Easy (Quick wins — simple SQL)
| Q# | Question | Core Pattern |
|----|----------|-------------|
| Q7 | Japanese City Population Sum | WHERE + SUM |
| Q9 | Eco-Friendly Product Filtering | WHERE multi-condition |
| Q12 | Product Sales Order Analysis | JOIN + GROUP BY |
| Q29 | Customers Single Item | HAVING COUNT = 1 |
| Q34 | Neighborhoods Zero Users | LEFT JOIN + IS NULL |
| Q37 | Products Never Reviewed | LEFT JOIN + IS NULL |
| Q47 | Alternate Records | ROW_NUMBER % 2 |

### Medium (Core interview SQL)
| Q# | Question | Core Pattern |
|----|----------|-------------|
| Q2 | Duplicate Product Listing | GROUP BY HAVING + Subquery |
| Q3 | Duplicate Job Posting | GROUP BY HAVING COUNT |
| Q10 | First Customer Order | MIN + JOIN / ROW_NUMBER |
| Q11 | Highest Revenue Products | SUM + ORDER + LIMIT |
| Q14 | Returning User Detection | Self Join / LAG |
| Q15 | Shipments Per Month | DATE_FORMAT + COUNT |
| Q16 | Cumulative % of Orders | SUM OVER + PERCENT |
| Q18 | Find Gaps in IDs | LEAD + gap detection |
| Q25 | Product Cross-Sell | Self Join (pair filter) |
| Q28 | Customers Multi-Day | COUNT(DISTINCT date) |
| Q38 | User Repeat Purchases | GROUP BY HAVING COUNT > 1 |
| Q42 | Semi-Join Existence | EXISTS subquery |
| Q43 | Consecutive Numbers | Self Join triple / ROW_NUMBER trick |
| Q46 | Nth Highest Salary | DENSE_RANK = N |

### Hard (Advanced SQL)
| Q# | Question | Core Pattern |
|----|----------|-------------|
| Q1 | Country GDP Growth Rate | LAG + UNION ALL |
| Q5 | Month-over-Month Revenue | LAG + DATE_FORMAT + CTE |
| Q17 | Every Month of 2023 | HAVING COUNT(DISTINCT month) = 12 |
| Q20 | Recursive Manager Chain | Recursive CTE |
| Q21 | Price Gouging (3-Sigma) | AVG + STDDEV window |
| Q31 | Prime Members Active | Date range filter |
| Q32 | Maximize Prime Inventory | CASE WHEN capacity logic |
| Q49 | Highest Grossing per Category | DENSE_RANK partition |
| Q51 | 3-Month Rolling Avg | ROWS BETWEEN 2 PRECEDING |
| Q52 | 3 Consecutive Months Growth | LAG chain comparison |
| Q53 | Revenue Anomaly Detection | Rolling avg + threshold |
| Q54 | Server Utilization | LEAD for session pairing |
| Q55 | Top 3 MoM Growth | LAG + RANK on growth |
| Q60 | Peak Concurrent Users | Event timeline sweep |

---

## Common Follow-up Themes

### 1. Handling Ties
Always ask: "What if multiple rows have the same value?" → Use `DENSE_RANK` over `RANK` or `ROW_NUMBER` for top-N-with-ties.

### 2. NULL Handling
- `COALESCE(col, default)` for replacing NULLs
- `IS NULL` / `IS NOT NULL` for filtering
- NULLs propagate: `NULL + 5 = NULL`, `NULL = NULL` is FALSE (use `IS NULL`)

### 3. Performance & Optimization
- **Indexes**: Filter columns (WHERE), join columns (ON), ORDER BY columns
- **CTEs vs Subqueries**: CTEs are generally more readable; subqueries can be more efficient for single-use
- **Window vs GROUP BY**: Window functions don't reduce row count; GROUP BY does
- **EXISTS vs IN**: EXISTS short-circuits; IN can be slower with large subquery results

### 4. Date Handling
- `DATE_FORMAT(col, '%Y-%m')` — MySQL/Spark monthly grouping
- `TO_CHAR(col, 'YYYY-MM')` — PostgreSQL equivalent
- `DATEDIFF(d1, d2)` — MySQL; `d1 - d2` in PostgreSQL for integer days
- `INTERVAL '1' DAY` — add/subtract time periods

### 5. Aggregation Patterns
- Running total: `SUM(x) OVER (ORDER BY date)`
- Rolling average: `AVG(x) OVER (ROWS BETWEEN N PRECEDING AND CURRENT ROW)`
- YoY/MoM: `LAG(x) OVER (PARTITION BY entity ORDER BY time_period)`
- Conditional count: `SUM(CASE WHEN condition THEN 1 ELSE 0 END)`

---

## Q61 — Flatten Nested Structures (Order Header + Line Items)
**Difficulty:** Hard  
**Pattern:** LEFT JOIN + GROUP BY + STRING_AGG / GROUP_CONCAT  
**Tags:** Aggregate Functions, String Functions, Null Handling

### Problem
An order system stores headers in `orders` and line items in `order_items`. Produce one flattened row per order with distinct item count, total quantity, total line amount, and a comma-separated product list sorted by item_id ascending. Orders with zero line items still appear with NULLs for quantity/amount/product_list.

### Schema
```
orders:      order_id (INT), customer_id (VARCHAR), order_date (DATE)
order_items: order_id (INT), item_id (INT), product_name (VARCHAR),
             quantity (INT), unit_price (DECIMAL)
```

### Example
**Input:**
```
orders: (101, C001, 2024-01-15)
order_items: (101,1,Laptop,1,899.99), (101,2,Mouse,2,25.00)
```
**Output:**
```
order_id | customer_id | order_date | total_items | total_quantity | total_amount | product_list
101      | C001        | 2024-01-15 | 2           | 3              | 949.99       | Laptop,Mouse
```

### SQL Solution (PostgreSQL)
```sql
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    COUNT(oi.item_id)                                              AS total_items,
    SUM(oi.quantity)                                               AS total_quantity,
    ROUND(SUM(oi.quantity * oi.unit_price)::NUMERIC, 2)            AS total_amount,
    STRING_AGG(oi.product_name, ',' ORDER BY oi.item_id)          AS product_list
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.customer_id, o.order_date
ORDER BY o.order_id;
```

**MySQL equivalent** (replace STRING_AGG with GROUP_CONCAT):
```sql
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    COUNT(oi.item_id)                                                         AS total_items,
    SUM(oi.quantity)                                                          AS total_quantity,
    ROUND(SUM(oi.quantity * oi.unit_price), 2)                                AS total_amount,
    GROUP_CONCAT(oi.product_name ORDER BY oi.item_id SEPARATOR ',')          AS product_list
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.customer_id, o.order_date
ORDER BY o.order_id;
```

### Complexity
- **Time:** O(n log n) — sort inside STRING_AGG/GROUP_CONCAT  
- **Space:** O(n) — one output row per order

### Follow-ups
**Q: Why LEFT JOIN instead of INNER JOIN?**  
A: Orders with zero line items must still appear with `total_items = 0` and NULLs. INNER JOIN would silently drop those orders.

**Q: Why does `COUNT(oi.item_id)` return 0 for orders with no items?**  
A: `COUNT(col)` ignores NULLs — when there are no matching `order_items` rows, `oi.item_id` is NULL and COUNT returns 0. `COUNT(*)` would return 1 (the NULL-padded row).

**Q: How do you handle the NULL total_quantity / total_amount for empty orders?**  
A: `SUM(NULL)` already returns NULL in SQL, so no extra CASE WHEN needed. The constraint says those columns should be NULL, which is the natural behavior.

**Q: What if product names need deduplication in product_list?**  
- PostgreSQL: `STRING_AGG(DISTINCT product_name, ',' ORDER BY product_name)`  
- MySQL: `GROUP_CONCAT(DISTINCT product_name ORDER BY product_name SEPARATOR ',')`

**Q: How would you limit product_list to 255 characters?**  
- PostgreSQL: `LEFT(STRING_AGG(...), 255)`  
- MySQL: `GROUP_CONCAT(... SEPARATOR ',' LIMIT ... )` — or set `group_concat_max_len = 255`

**Q: How do you also return the most expensive item in each order?**  
```sql
MAX(oi.unit_price) AS max_item_price
```
Add to the SELECT with the existing GROUP BY — no extra join needed.

---

## ─── PYTHON / PYSPARK SECTION ─── (Questions 62–77)

> These questions have Python-style slugs (pandas-apply-lambda, list-comprehension, etc.) but run on a PostgreSQL editor. Each question is solved in **three ways: PySpark · Python (pandas) · SQL**, all in the same style.

---

### Quick-Reference: PySpark vs Pandas vs SQL

| Operation | PySpark | Pandas | SQL |
|-----------|---------|--------|-----|
| GROUP BY agg | `df.groupBy().agg(F.sum(),F.avg())` | `df.groupby().agg({"col":["sum","mean"]})` | `GROUP BY … SUM(), AVG()` |
| CASE WHEN | `F.when(cond,v).otherwise(d)` | `np.where(cond,v,d)` / `pd.cut()` | `CASE WHEN … END` |
| CROSS JOIN | `df1.crossJoin(df2)` | `df1.merge(df2, how="cross")` | `CROSS JOIN` |
| UNION ALL | `df1.unionAll(df2)` | `pd.concat([df1,df2])` | `UNION ALL` |
| String split | `F.split(col," ")[0]` | `df.col.str.split(" ").str[0]` | `SPLIT_PART(col,' ',1)` |
| Date diff | `F.datediff(d2,d1)` | `(d2-d1).dt.days` | `d2-d1` (integer days) |
| Window RANK | `F.rank().over(Window...)` | `df.groupby().rank(method="min")` | `RANK() OVER (PARTITION BY …)` |
| ROW_NUMBER dedup | `F.row_number().over(w)` → filter==1 | `df.drop_duplicates(subset, keep="first")` | `ROW_NUMBER() OVER (…)` → WHERE rn=1 |
| Forward fill | `F.last(col,ignorenulls=True).over(w)` | `df.groupby().ffill()` | `LAST_VALUE(col IGNORE NULLS) OVER` |
| Explode/UNNEST | `F.explode(F.split(col,","))` | `df.col.str.split(",").explode()` | `UNNEST(STRING_TO_ARRAY(col,','))` |
| Pivot | `.pivot("cat",vals).agg(F.sum())` | `df.pivot_table(values,index,columns,aggfunc)` | `SUM(CASE WHEN cat='X' THEN val END)` |
| Median | `F.percentile_approx(col,0.5)` | `df.groupby().median()` | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col)` |
| Std dev (sample) | `F.stddev_samp(col)` | `df.groupby().std(ddof=1)` | `STDDEV_SAMP(col)` |

---

## Q62 — Product Sales Aggregation (dataframe-groupby-aggregation)
**Difficulty:** Medium | **Patterns:** `groupBy/groupby + agg`, `NULL handling`, `COUNT(*)` 

### Problem
`sales(sale_id, product_id, price, quantity, sale_date)`. Return per product: `total_sales`=SUM(price×qty), `avg_price`=AVG(price), `transaction_count`=COUNT(*). NULLs excluded from SUM/AVG but counted. Order by total_sales DESC.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    sales
    .groupBy("product_id")
    .agg(
        F.round(F.sum(F.col("price") * F.col("quantity")), 2).alias("total_sales"),
        F.round(F.avg("price"), 2).alias("avg_price"),
        F.count("*").alias("transaction_count")
    )
    .orderBy(F.desc("total_sales"))
)
```

### Python (pandas)
```python
import pandas as pd
import numpy as np

sales["revenue"] = sales["price"] * sales["quantity"]   # NaN * qty = NaN, skipped by sum

result = (
    sales
    .groupby("product_id", as_index=False)
    .agg(
        total_sales=("revenue", "sum"),
        avg_price=("price", "mean"),
        transaction_count=("sale_id", "count")           # counts all rows
    )
    .assign(total_sales=lambda d: d.total_sales.round(2),
            avg_price=lambda d: d.avg_price.round(2))
    .sort_values("total_sales", ascending=False)
    .reset_index(drop=True)
)
```

### SQL (PostgreSQL)
```sql
SELECT product_id,
    ROUND(SUM(price * quantity)::NUMERIC, 2) AS total_sales,
    ROUND(AVG(price)::NUMERIC, 2)            AS avg_price,
    COUNT(*)                                  AS transaction_count
FROM sales GROUP BY product_id ORDER BY total_sales DESC;
```

### Complexity — O(n) scan + O(k log k) sort | Space O(k)

### Follow-ups
**Q: How do NULLs propagate in pandas `price * quantity`?**  
A: `NaN * number = NaN` in NumPy; pandas `sum()` skips NaN by default (`skipna=True`).

**Q: How do you count ALL rows including NULL-price rows in pandas?**  
A: Use a non-nullable column for count (e.g. `sale_id`). `count()` in pandas skips NaN, so always count on the primary key.

**Q: How do you add a HAVING filter (transactions > 5) in each approach?**  
- PySpark: `.filter(F.col("transaction_count") > 5)` after `.agg()`  
- pandas: `.query("transaction_count > 5")`  
- SQL: `HAVING COUNT(*) > 5`

---

## Q63 — Employee Profile Extraction (pandas-apply-lambda)
**Difficulty:** Medium | **Patterns:** `String extraction`, `Date math`, `CASE WHEN`

### Problem
`employees(emp_id, full_name, email, hire_date, salary, department)`. Derive: `first_name`, `years_employed` ((2024-01-01−hire_date)/365.25 rounded 2dp), `salary_band` (junior/mid/senior). Order by emp_id.

### PySpark
```python
from pyspark.sql import functions as F
from datetime import date

result = (
    employees
    .withColumn("first_name", F.split("full_name", " ")[0])
    .withColumn("years_employed",
        F.round(F.datediff(F.lit(date(2024,1,1)), F.col("hire_date").cast("date")) / 365.25, 2))
    .withColumn("salary_band",
        F.when(F.col("salary") < 60000, "junior")
         .when(F.col("salary") < 100000, "mid")
         .otherwise("senior"))
    .select("emp_id","first_name","years_employed","salary_band")
    .orderBy("emp_id")
)
```

### Python (pandas)
```python
import pandas as pd
import numpy as np

cutoff = pd.Timestamp("2024-01-01")
employees["hire_date"] = pd.to_datetime(employees["hire_date"])

result = employees.assign(
    first_name=employees["full_name"].str.split(" ").str[0],
    years_employed=((cutoff - employees["hire_date"]).dt.days / 365.25).round(2),
    salary_band=pd.cut(
        employees["salary"],
        bins=[-np.inf, 60000, 100000, np.inf],
        labels=["junior", "mid", "senior"]
    )
)[["emp_id","first_name","years_employed","salary_band"]].sort_values("emp_id")
```

### SQL (PostgreSQL)
```sql
SELECT emp_id,
    SPLIT_PART(full_name,' ',1) AS first_name,
    ROUND((DATE '2024-01-01' - hire_date)::NUMERIC / 365.25, 2) AS years_employed,
    CASE WHEN salary < 60000 THEN 'junior' WHEN salary < 100000 THEN 'mid' ELSE 'senior' END AS salary_band
FROM employees ORDER BY emp_id;
```

### Complexity — O(n) | Space O(n)

### Follow-ups
**Q: What does pandas `pd.cut()` do and when should you use it?**  
A: Bins a numeric series into labeled intervals — perfect for CASE WHEN range logic. `bins` are the cutpoints, `labels` are the category names.

**Q: How does `str.split(" ").str[0]` handle leading spaces?**  
A: If the name starts with a space, element [0] is an empty string. Use `.str.strip().str.split(" ").str[0]` for safety.

**Q: How do you extract last name in pandas?**  
```python
employees["full_name"].str.split(" ").str[-1]
```

---

## Q64 — Promotion Price Matrix (list-comprehension)
**Difficulty:** Easy | **Patterns:** `CROSS JOIN`, `Computed column`

### Problem
Pair every product with every promotion. `discounted_price = base_price × (1 − discount_pct/100)` rounded 2dp. Order by product_id, promo_id.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    products.crossJoin(promotions)
    .withColumn("discounted_price",
        F.round(F.col("base_price") * (1 - F.col("discount_pct") / 100), 2))
    .select("product_id","promo_id","discounted_price")
    .orderBy("product_id","promo_id")
)
```

### Python (pandas)
```python
# pandas >= 1.2 supports how="cross" in merge
result = (
    products.merge(promotions, how="cross")
    .assign(discounted_price=lambda d: (d.base_price * (1 - d.discount_pct / 100)).round(2))
    [["product_id","promo_id","discounted_price"]]
    .sort_values(["product_id","promo_id"])
    .reset_index(drop=True)
)

# For older pandas, use a key column trick:
products["_key"] = 1; promotions["_key"] = 1
result = products.merge(promotions, on="_key").drop("_key",axis=1)
```

### SQL (PostgreSQL)
```sql
SELECT p.product_id, pr.promo_id,
    ROUND((p.base_price*(1-pr.discount_pct/100.0))::NUMERIC,2) AS discounted_price
FROM products p CROSS JOIN promotions pr ORDER BY p.product_id, pr.promo_id;
```

### Complexity — O(p × m) | Space O(p × m)

### Follow-ups
**Q: What's the list comprehension equivalent in pure Python (hence the slug name)?**  
```python
[(p, pr, round(p["base_price"] * (1-pr["discount_pct"]/100), 2))
 for p in products for pr in promotions]
```
SQL CROSS JOIN is the set-based equivalent of Python's nested for-loop.

**Q: How do you filter cross join results to only discounted_price ≥ 50?**  
- PySpark: `.filter(F.col("discounted_price") >= 50)`  
- pandas: `.query("discounted_price >= 50")`  
- SQL: `WHERE discounted_price >= 50` (in subquery) or `HAVING`

---

## Q65 — Regional Sales by Category (concat-vs-merge)
**Difficulty:** Medium | **Patterns:** `UNION ALL / concat`, `LEFT JOIN`, `groupBy + agg`

### Problem
UNION ALL `sales_north` + `sales_south`, LEFT JOIN `products` for category. Per region+category: `total_sales`, `avg_sale_amount`, `transaction_count`. NULL amounts excluded from aggs. Order by region ASC, total_sales DESC.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    sales_north.unionAll(sales_south)
    .join(products.select("product_id","category"), on="product_id", how="left")
    .groupBy("region","category")
    .agg(
        F.round(F.sum("amount"),2).alias("total_sales"),
        F.round(F.avg("amount"),2).alias("avg_sale_amount"),
        F.count("*").alias("transaction_count")
    )
    .orderBy(F.asc("region"), F.desc("total_sales"))
)
```

### Python (pandas)
```python
import pandas as pd

all_sales = pd.concat([sales_north, sales_south], ignore_index=True)
merged = all_sales.merge(products[["product_id","category"]], on="product_id", how="left")

result = (
    merged
    .groupby(["region","category"], as_index=False)
    .agg(
        total_sales=("amount","sum"),
        avg_sale_amount=("amount","mean"),
        transaction_count=("sale_id","count")
    )
    .assign(total_sales=lambda d: d.total_sales.round(2),
            avg_sale_amount=lambda d: d.avg_sale_amount.round(2))
    .sort_values(["region","total_sales"], ascending=[True,False])
    .reset_index(drop=True)
)
```

### SQL (PostgreSQL)
```sql
WITH all_sales AS (
    SELECT sale_id,product_id,region,amount FROM sales_north
    UNION ALL SELECT sale_id,product_id,region,amount FROM sales_south
)
SELECT s.region, p.category,
    ROUND(SUM(s.amount)::NUMERIC,2) AS total_sales,
    ROUND(AVG(s.amount)::NUMERIC,2) AS avg_sale_amount,
    COUNT(*) AS transaction_count
FROM all_sales s LEFT JOIN products p ON s.product_id=p.product_id
GROUP BY s.region,p.category ORDER BY s.region ASC,total_sales DESC;
```

### Complexity — O(n log n) | Space O(n)

### Follow-ups
**Q: `pd.concat` vs `df.append` — which to use?**  
A: `pd.concat` — `df.append` was deprecated in pandas 1.4 and removed in 2.0.

**Q: Why `ignore_index=True` in `pd.concat`?**  
A: Resets the row index so you don't get duplicate index values from both DataFrames.

**Q: How do you dedup after UNION ALL in pandas (equivalent of UNION)?**  
```python
pd.concat([df1, df2]).drop_duplicates().reset_index(drop=True)
```

---

## Q66 — Order Classification & Priority Score (conditional-column-creation)
**Difficulty:** Easy | **Patterns:** `F.when / np.select`, `Computed score`

### Problem
`orders(order_id, customer_id, order_date, amount, shipping_type)`. Add `order_size`, `is_express` (1/0), `priority_score` = amount/100 + 10×is_express rounded 2dp. Order by priority_score DESC.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    orders
    .withColumn("order_size",
        F.when(F.col("amount")<100,"small").when(F.col("amount")<=500,"medium").otherwise("large"))
    .withColumn("is_express",
        (F.col("shipping_type")=="express").cast("int"))
    .withColumn("priority_score",
        F.round(F.col("amount")/100 + 10*(F.col("shipping_type")=="express").cast("int"),2))
    .orderBy(F.desc("priority_score"))
)
```

### Python (pandas)
```python
import pandas as pd
import numpy as np

df = orders.copy()
df["is_express"] = (df["shipping_type"] == "express").astype(int)

df["order_size"] = np.select(
    [df["amount"] < 100, df["amount"] <= 500],
    ["small", "medium"],
    default="large"
)

df["priority_score"] = (df["amount"] / 100 + 10 * df["is_express"]).round(2)
result = df.sort_values("priority_score", ascending=False).reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
SELECT *,
    CASE WHEN amount<100 THEN 'small' WHEN amount<=500 THEN 'medium' ELSE 'large' END AS order_size,
    CASE WHEN shipping_type='express' THEN 1 ELSE 0 END AS is_express,
    ROUND((amount/100.0+10*CASE WHEN shipping_type='express' THEN 1 ELSE 0 END)::NUMERIC,2) AS priority_score
FROM orders ORDER BY priority_score DESC;
```

### Complexity — O(n log n) | Space O(n)

### Follow-ups
**Q: What is `np.select` and when is it better than chained `np.where`?**  
A: `np.select(conditions_list, choices_list, default)` — cleaner than nested `np.where` for 3+ categories. Reads like a CASE WHEN.

**Q: Can you use a column alias in the same pandas `assign()` call?**  
A: Yes — if you pass a callable (lambda), it receives the DataFrame after previous assignments in the same call:
```python
df.assign(is_express=lambda d: (d.shipping_type=="express").astype(int),
          priority_score=lambda d: (d.amount/100 + 10*d.is_express).round(2))
```

---

## Q67 — Survey Answer Mapping (survey-answer-mapping)
**Difficulty:** Easy | **Patterns:** `map / replace`, `pd.cut / np.select`

### Problem
`survey_responses(response_id, question_id, answer, respondent_age)`. Map answer→numeric, age→group. Order by response_id.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    survey_responses
    .withColumn("answer_numeric",
        F.when(F.col("answer")=="Yes",1.0).when(F.col("answer")=="No",0.0)
         .when(F.col("answer")=="Maybe",0.5).otherwise(None))
    .withColumn("age_group",
        F.when(F.col("respondent_age")<30,"under_30")
         .when(F.col("respondent_age")<=50,"30_to_50").otherwise("over_50"))
    .orderBy("response_id")
)
```

### Python (pandas)
```python
import pandas as pd
import numpy as np

answer_map = {"Yes": 1.0, "No": 0.0, "Maybe": 0.5}

result = survey_responses.assign(
    answer_numeric=survey_responses["answer"].map(answer_map),   # unmapped → NaN
    age_group=np.select(
        [survey_responses["respondent_age"] < 30,
         survey_responses["respondent_age"] <= 50],
        ["under_30", "30_to_50"],
        default="over_50"
    )
).sort_values("response_id").reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
SELECT response_id, question_id, answer, respondent_age,
    CASE answer WHEN 'Yes' THEN 1.0 WHEN 'No' THEN 0.0 WHEN 'Maybe' THEN 0.5 END AS answer_numeric,
    CASE WHEN respondent_age<30 THEN 'under_30' WHEN respondent_age<=50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM survey_responses ORDER BY response_id;
```

### Complexity — O(n) | Space O(n)

### Follow-ups
**Q: What does `Series.map(dict)` return for keys not in the dict?**  
A: `NaN` — which is the pandas equivalent of SQL NULL. Use `.fillna()` or a default if you want a different value.

**Q: How does `pd.cut()` differ from `np.select()` for age groups?**  
```python
pd.cut(df["respondent_age"], bins=[0,29,50,np.inf], labels=["under_30","30_to_50","over_50"])
```
`pd.cut` is cleaner for ordered numeric bins; `np.select` is more flexible for arbitrary conditions.

---

## Q68 — Subject Statistics (statistical-calculations)
**Difficulty:** Medium | **Patterns:** `stddev`, `median`, `multi-agg groupby`

### Problem
`test_results(test_id, student_id, subject, score, test_date)`. Per subject: `student_count` (distinct), `avg_score`, `min_score`, `max_score`, `std_dev` (sample), `median_score`. Order by subject.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    test_results.groupBy("subject")
    .agg(
        F.countDistinct("student_id").alias("student_count"),
        F.round(F.avg("score"),2).alias("avg_score"),
        F.min("score").alias("min_score"),
        F.max("score").alias("max_score"),
        F.round(F.stddev_samp("score"),2).alias("std_dev"),
        F.round(F.percentile_approx("score",0.5),2).alias("median_score")
    )
    .orderBy("subject")
)
```

### Python (pandas)
```python
import pandas as pd

result = (
    test_results.groupby("subject")
    .agg(
        student_count=("student_id","nunique"),
        avg_score=("score","mean"),
        min_score=("score","min"),
        max_score=("score","max"),
        std_dev=("score", lambda x: x.std(ddof=1)),    # sample stddev
        median_score=("score","median")
    )
    .round(2)
    .reset_index()
    .sort_values("subject")
)
```

### SQL (PostgreSQL)
```sql
SELECT subject,
    COUNT(DISTINCT student_id) AS student_count,
    ROUND(AVG(score)::NUMERIC,2) AS avg_score,
    MIN(score) AS min_score, MAX(score) AS max_score,
    ROUND(STDDEV_SAMP(score)::NUMERIC,2) AS std_dev,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY score)::NUMERIC,2) AS median_score
FROM test_results GROUP BY subject ORDER BY subject;
```

### Complexity — O(n log n) for median | Space O(n)

### Follow-ups
**Q: Why `ddof=1` in pandas `.std()`?**  
A: pandas `.std()` uses `ddof=1` (sample std) by default. Pass `ddof=0` for population std. This matches SQL `STDDEV_SAMP` vs `STDDEV_POP`.

**Q: Is pandas `.median()` exact? How about PySpark `percentile_approx`?**  
A: pandas median is exact (sorts the series). PySpark `percentile_approx` is approximate by design — use `F.expr("percentile(score,0.5)")` for exact (forces a shuffle).

---

## Q69 — Quarterly Revenue Pivot (pivot-aggregations)
**Difficulty:** Hard | **Patterns:** `pivot / pivot_table`, `Conditional aggregation`

### Problem
`sales(sale_id, region, product_category, quarter, revenue, quantity)`. Pivot to one row per (region, product_category) with q1–q4 revenue and quantity. Order by region, product_category.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    sales.groupBy("region","product_category")
    .agg(
        F.round(F.sum(F.when(F.col("quarter")=="Q1",F.col("revenue"))),2).alias("q1_revenue"),
        F.sum(F.when(F.col("quarter")=="Q1",F.col("quantity"))).alias("q1_quantity"),
        F.round(F.sum(F.when(F.col("quarter")=="Q2",F.col("revenue"))),2).alias("q2_revenue"),
        F.sum(F.when(F.col("quarter")=="Q2",F.col("quantity"))).alias("q2_quantity"),
        F.round(F.sum(F.when(F.col("quarter")=="Q3",F.col("revenue"))),2).alias("q3_revenue"),
        F.sum(F.when(F.col("quarter")=="Q3",F.col("quantity"))).alias("q3_quantity"),
        F.round(F.sum(F.when(F.col("quarter")=="Q4",F.col("revenue"))),2).alias("q4_revenue"),
        F.sum(F.when(F.col("quarter")=="Q4",F.col("quantity"))).alias("q4_quantity"),
    )
    .orderBy("region","product_category")
)
```

### Python (pandas)
```python
import pandas as pd

rev_pivot = sales.pivot_table(
    values="revenue", index=["region","product_category"],
    columns="quarter", aggfunc="sum"
).round(2).add_prefix("q").rename(columns=lambda c: c.replace("q","q").replace("Q","") + "_revenue")
# Simpler approach:
rev = sales.pivot_table(values="revenue",index=["region","product_category"],columns="quarter",aggfunc="sum").round(2)
qty = sales.pivot_table(values="quantity",index=["region","product_category"],columns="quarter",aggfunc="sum")

rev.columns = [f"q{c[1]}_revenue" for c in rev.columns]
qty.columns = [f"q{c[1]}_quantity" for c in qty.columns]

result = (
    rev.join(qty)[["q1_revenue","q1_quantity","q2_revenue","q2_quantity",
                   "q3_revenue","q3_quantity","q4_revenue","q4_quantity"]]
    .reset_index()
    .sort_values(["region","product_category"])
)
```

### SQL (PostgreSQL)
```sql
SELECT region,product_category,
    ROUND(SUM(CASE WHEN quarter='Q1' THEN revenue END)::NUMERIC,2) AS q1_revenue,
    SUM(CASE WHEN quarter='Q1' THEN quantity END) AS q1_quantity,
    ROUND(SUM(CASE WHEN quarter='Q2' THEN revenue END)::NUMERIC,2) AS q2_revenue,
    SUM(CASE WHEN quarter='Q2' THEN quantity END) AS q2_quantity,
    ROUND(SUM(CASE WHEN quarter='Q3' THEN revenue END)::NUMERIC,2) AS q3_revenue,
    SUM(CASE WHEN quarter='Q3' THEN quantity END) AS q3_quantity,
    ROUND(SUM(CASE WHEN quarter='Q4' THEN revenue END)::NUMERIC,2) AS q4_revenue,
    SUM(CASE WHEN quarter='Q4' THEN quantity END) AS q4_quantity
FROM sales GROUP BY region,product_category ORDER BY region,product_category;
```

### Complexity — O(n) agg + O(k log k) sort | Space O(k)

### Follow-ups
**Q: `pivot_table` vs `pivot` in pandas?**  
A: `pivot` requires unique (index, columns) pairs — errors on duplicates. `pivot_table` aggregates duplicates with `aggfunc`. Always use `pivot_table` for real data.

**Q: How do you flatten multi-level column headers after `pivot_table`?**  
```python
df.columns = ["_".join(str(c) for c in col).strip("_") for col in df.columns]
```

**Q: When does PySpark `.pivot()` scan data twice?**  
A: When you omit the values list — Spark does a first pass to collect distinct values, then a second pass to aggregate. Always pass `["Q1","Q2","Q3","Q4"]` explicitly.

---

## Q70 — Order Data Cleaning (data-cleaning)
**Difficulty:** Hard | **Patterns:** `dedup`, `ROW_NUMBER`, `validation flags`

### Problem
`raw_orders(order_id, customer_email, product_name, quantity, unit_price, order_date, status)`. Keep first per order_id. Add `is_valid_email` (1/0), `is_clean` (all checks pass). Order by order_id.

### PySpark
```python
from pyspark.sql import functions as F, Window

w = Window.partitionBy("order_id").orderBy("order_id")
result = (
    raw_orders
    .withColumn("rn", F.row_number().over(w)).filter(F.col("rn")==1).drop("rn")
    .withColumn("is_valid_email", (F.instr("customer_email","@")>0).cast("int"))
    .withColumn("is_clean",
        F.when(
            (F.instr("customer_email","@")>0) & (F.col("quantity")>0) &
            F.col("unit_price").isNotNull() & (F.col("order_date").cast("date")<=F.lit("2024-12-31").cast("date")),
            1).otherwise(0))
    .orderBy("order_id")
)
```

### Python (pandas)
```python
import pandas as pd

df = (raw_orders
      .sort_values("order_id")
      .drop_duplicates(subset="order_id", keep="first")
      .reset_index(drop=True))

df["order_date"] = pd.to_datetime(df["order_date"])
df["is_valid_email"] = df["customer_email"].str.contains("@", na=False).astype(int)
df["is_clean"] = (
    (df["is_valid_email"] == 1) &
    (df["quantity"] > 0) &
    (df["unit_price"].notna()) &
    (df["order_date"] <= pd.Timestamp("2024-12-31"))
).astype(int)

result = df.sort_values("order_id").reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
WITH deduped AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) rn FROM raw_orders
), clean_check AS (
    SELECT *, CASE WHEN POSITION('@' IN customer_email)>0 THEN 1 ELSE 0 END AS is_valid_email
    FROM deduped WHERE rn=1
)
SELECT *, CASE WHEN is_valid_email=1 AND quantity>0 AND unit_price IS NOT NULL
               AND order_date<=DATE '2024-12-31' THEN 1 ELSE 0 END AS is_clean
FROM clean_check ORDER BY order_id;
```

### Complexity — O(n log n) | Space O(n)

### Follow-ups
**Q: `drop_duplicates(keep="first")` vs ROW_NUMBER in SQL — same result?**  
A: Only if the DataFrame is sorted first. `drop_duplicates` keeps the first occurrence in DataFrame row order, which is insertion order only if you sort first.

**Q: How do you find what percentage of orders pass the clean check?**  
- pandas: `df["is_clean"].mean() * 100`  
- PySpark: `result.agg((F.sum("is_clean")/F.count("*")*100).alias("pct_clean"))`

---

## Q71 — Review Text Features (text-features)
**Difficulty:** Hard | **Patterns:** `String functions`, `Keyword counting`, `Sentiment score`

### Problem
`reviews(review_id, product_id, review_text, review_date)`. Compute: `word_count`, `char_count`, `avg_word_length`, `contains_negative` (1/0), `sentiment_score`. Order by review_id.

### PySpark
```python
from pyspark.sql import functions as F

pos_kws = ["great","good","excellent","amazing","love","best"]
neg_kws = ["bad","terrible","awful","worst","poor","hate"]

pos_score = sum(F.when(F.lower("review_text").contains(k),1).otherwise(0) for k in pos_kws)
neg_score = sum(F.when(F.lower("review_text").contains(k),1).otherwise(0) for k in neg_kws)

result = (
    reviews
    .withColumn("words", F.split(F.trim("review_text"), r"\s+"))
    .withColumn("word_count", F.size("words"))
    .withColumn("char_count", F.length("review_text"))
    .withColumn("avg_word_length", F.round(F.col("char_count")/F.col("word_count"),2))
    .withColumn("contains_negative",
        F.when(F.col("review_text").rlike(r"(?i)(bad|terrible|awful|worst|poor|hate)"),1).otherwise(0))
    .withColumn("sentiment_score", pos_score - neg_score)
    .drop("words").orderBy("review_id")
)
```

### Python (pandas)
```python
import pandas as pd

pos_kws = ["great","good","excellent","amazing","love","best"]
neg_kws = ["bad","terrible","awful","worst","poor","hate"]

df = reviews.copy()
df["text_lower"] = df["review_text"].str.lower()
df["word_count"]  = df["review_text"].str.strip().str.split().str.len()
df["char_count"]  = df["review_text"].str.len()
df["avg_word_length"] = (df["char_count"] / df["word_count"]).round(2)
df["contains_negative"] = df["text_lower"].str.contains("|".join(neg_kws), regex=True).astype(int)
df["pos_score"] = sum(df["text_lower"].str.contains(k).astype(int) for k in pos_kws)
df["neg_score"] = sum(df["text_lower"].str.contains(k).astype(int) for k in neg_kws)
df["sentiment_score"] = df["pos_score"] - df["neg_score"]

result = df.drop(columns=["text_lower","pos_score","neg_score"]).sort_values("review_id")
```

### SQL (PostgreSQL)
```sql
SELECT review_id, product_id, review_text, review_date,
    ARRAY_LENGTH(STRING_TO_ARRAY(TRIM(review_text),' '),1) AS word_count,
    LENGTH(review_text) AS char_count,
    ROUND(LENGTH(review_text)::NUMERIC/NULLIF(ARRAY_LENGTH(STRING_TO_ARRAY(TRIM(review_text),' '),1),0),2) AS avg_word_length,
    CASE WHEN LOWER(review_text) ~* '\y(bad|terrible|awful|worst|poor|hate)\y' THEN 1 ELSE 0 END AS contains_negative,
    (CASE WHEN review_text ILIKE '%great%' THEN 1 ELSE 0 END +
     CASE WHEN review_text ILIKE '%good%' THEN 1 ELSE 0 END +
     CASE WHEN review_text ILIKE '%excellent%' THEN 1 ELSE 0 END +
     CASE WHEN review_text ILIKE '%amazing%' THEN 1 ELSE 0 END +
     CASE WHEN review_text ILIKE '%love%' THEN 1 ELSE 0 END +
     CASE WHEN review_text ILIKE '%best%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%bad%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%terrible%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%awful%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%worst%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%poor%' THEN 1 ELSE 0 END -
     CASE WHEN review_text ILIKE '%hate%' THEN 1 ELSE 0 END) AS sentiment_score
FROM reviews ORDER BY review_id;
```

### Complexity — O(n × k) keyword checks | Space O(n)

### Follow-ups
**Q: How does `str.contains("|".join(keywords))` work in pandas?**  
A: Joins keywords with regex OR (`|`), then uses `str.contains(regex=True)` — equivalent to `LIKE '%kw1%' OR LIKE '%kw2%'`.

**Q: How do you count keyword frequency (not just presence)?**  
```python
df["review_text"].str.lower().str.count(r"\bgreat\b")  # exact word matches
```

---

## Q72 — Metadata String Parsing (string-parsing-json)
**Difficulty:** Medium | **Patterns:** `explode`, `Key-value parsing`, `Long→Wide pivot`

### Problem
`product_listings(listing_id, product_name, metadata, list_date)` where metadata = `color=red|size=M|weight=1.5kg`. Extract `color`, `size`, `weight_value` (numeric). Order by listing_id.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    product_listings
    .withColumn("pair", F.explode(F.split("metadata", r"\|")))
    .withColumn("key", F.split("pair","=")[0])
    .withColumn("val", F.split("pair","=")[1])
    .groupBy("listing_id","product_name","list_date")
    .agg(
        F.coalesce(F.max(F.when(F.col("key")=="color",F.col("val"))),F.lit("")).alias("color"),
        F.coalesce(F.max(F.when(F.col("key")=="size",F.col("val"))),F.lit("")).alias("size"),
        F.max(F.when(F.col("key")=="weight",
              F.regexp_extract("val",r"[0-9]+\.?[0-9]*",0).cast("double"))).alias("weight_value")
    )
    .orderBy("listing_id")
)
```

### Python (pandas)
```python
import pandas as pd
import re

def parse_meta(meta):
    kv = {}
    if pd.isna(meta): return {}
    for pair in meta.split("|"):
        if "=" in pair:
            k, v = pair.split("=", 1)
            kv[k.strip()] = v.strip()
    return kv

parsed = product_listings["metadata"].apply(parse_meta).apply(pd.Series)

result = product_listings[["listing_id","product_name","list_date"]].copy()
result["color"]  = parsed.get("color", pd.Series([""] * len(parsed), index=parsed.index)).fillna("")
result["size"]   = parsed.get("size",  pd.Series([""] * len(parsed), index=parsed.index)).fillna("")
result["weight_value"] = parsed.get("weight", pd.Series([None]*len(parsed), index=parsed.index)) \
    .apply(lambda v: float(re.search(r"[0-9]+\.?[0-9]*", str(v)).group()) if pd.notna(v) else None)
result = result.sort_values("listing_id").reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
WITH kv AS (
    SELECT listing_id, product_name, list_date,
           TRIM(SPLIT_PART(pair,'=',1)) AS key, TRIM(SPLIT_PART(pair,'=',2)) AS val
    FROM product_listings, UNNEST(STRING_TO_ARRAY(metadata,'|')) pair
)
SELECT listing_id, product_name, list_date,
    MAX(CASE WHEN key='color' THEN val ELSE '' END) AS color,
    MAX(CASE WHEN key='size'  THEN val ELSE '' END) AS size,
    MAX(CASE WHEN key='weight' THEN CAST(REGEXP_REPLACE(val,'[^0-9.]','','g') AS NUMERIC) END) AS weight_value
FROM kv GROUP BY listing_id, product_name, list_date ORDER BY listing_id;
```

### Complexity — O(n × m) | Space O(n × m)

### Follow-ups
**Q: When would you store metadata as JSON instead of pipe-delimited?**  
A: JSON is self-describing, handles nesting, and has native support in PostgreSQL (`JSONB`), pandas (`pd.read_json`), and PySpark (`F.from_json`). Pipe-delimited is only suitable for flat, fixed-schema attributes.

**Q: How do you parse JSON metadata in pandas?**  
```python
import json
df["metadata"].apply(json.loads)  # if metadata is valid JSON string
```

---

## Q73 — User Activity Analysis (date-time-manipulation)
**Difficulty:** Medium | **Patterns:** `Date math`, `datediff`, `Activity frequency`

### Problem
`user_activity(user_id, activity_date, activity_type)`. Per user: `first_activity`, `last_activity`, `active_days` (distinct), `tenure_days`, `activity_frequency` = active_days/GREATEST(tenure_days,1) rounded 2dp.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    user_activity.groupBy("user_id")
    .agg(
        F.min("activity_date").alias("first_activity"),
        F.max("activity_date").alias("last_activity"),
        F.countDistinct("activity_date").alias("active_days"),
        F.datediff(F.max("activity_date"),F.min("activity_date")).alias("tenure_days")
    )
    .withColumn("activity_frequency",
        F.round(F.col("active_days")/F.greatest(F.col("tenure_days"),F.lit(1)),2))
    .orderBy("user_id")
)
```

### Python (pandas)
```python
import pandas as pd

ua = user_activity.copy()
ua["activity_date"] = pd.to_datetime(ua["activity_date"])

agg = ua.groupby("user_id").agg(
    first_activity=("activity_date","min"),
    last_activity=("activity_date","max"),
    active_days=("activity_date","nunique")
).reset_index()

agg["tenure_days"] = (agg["last_activity"] - agg["first_activity"]).dt.days
agg["activity_frequency"] = (
    agg["active_days"] / agg["tenure_days"].clip(lower=1)
).round(2)

result = agg.sort_values("user_id").reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
SELECT user_id,
    MIN(activity_date) AS first_activity, MAX(activity_date) AS last_activity,
    COUNT(DISTINCT activity_date) AS active_days,
    MAX(activity_date)-MIN(activity_date) AS tenure_days,
    ROUND(COUNT(DISTINCT activity_date)::NUMERIC/GREATEST(MAX(activity_date)-MIN(activity_date),1),2) AS activity_frequency
FROM user_activity GROUP BY user_id ORDER BY user_id;
```

### Complexity — O(n log n) | Space O(k)

### Follow-ups
**Q: What does `.clip(lower=1)` do in pandas?**  
A: Replaces values less than 1 with 1 — equivalent to `GREATEST(tenure_days, 1)` in SQL. Prevents division by zero for users active only one day.

**Q: How do you compute rolling 7-day active users in pandas?**  
```python
ua.set_index("activity_date").groupby("user_id")["user_id"].resample("7D").count()
```

---

## Q74 — Store Sales Ranking (rank-dense-rank)
**Difficulty:** Medium | **Patterns:** `Window RANK/DENSE_RANK`, `rank(method=…)`

### Problem
`store_sales(sale_id, store_id, sale_date, amount)`. Add `sale_rank` and `sale_dense_rank` per store. Order by store_id, amount DESC, sale_date.

### PySpark
```python
from pyspark.sql import functions as F, Window

w = Window.partitionBy("store_id").orderBy(F.desc("amount"))
result = (
    store_sales
    .withColumn("sale_rank",       F.rank().over(w))
    .withColumn("sale_dense_rank", F.dense_rank().over(w))
    .orderBy("store_id", F.desc("amount"), "sale_date")
)
```

### Python (pandas)
```python
import pandas as pd

df = store_sales.sort_values(["store_id","amount","sale_date"], ascending=[True,False,True])

df["sale_rank"] = df.groupby("store_id")["amount"].rank(method="min", ascending=False).astype(int)
df["sale_dense_rank"] = df.groupby("store_id")["amount"].rank(method="dense", ascending=False).astype(int)

result = df.sort_values(["store_id", "amount","sale_date"], ascending=[True,False,True]).reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
SELECT sale_id, store_id, sale_date, amount,
    RANK()       OVER (PARTITION BY store_id ORDER BY amount DESC) AS sale_rank,
    DENSE_RANK() OVER (PARTITION BY store_id ORDER BY amount DESC) AS sale_dense_rank
FROM store_sales ORDER BY store_id, amount DESC, sale_date;
```

### Complexity — O(n log n) | Space O(n)

### Follow-ups
**Q: pandas `rank(method=…)` options vs SQL equivalents?**  
| pandas method | SQL equivalent |
|--------------|----------------|
| `"average"` | — (average of tied ranks) |
| `"min"` | `RANK()` |
| `"dense"` | `DENSE_RANK()` |
| `"first"` | `ROW_NUMBER()` (by appearance order) |
| `"max"` | — (max of tied ranks) |

**Q: How do you get only top-3 per store in pandas?**  
```python
df[df["sale_dense_rank"] <= 3]
```

---

## Q75 — Forward Fill Sensor Readings (forward-fill-sensor-readings)
**Difficulty:** Easy | **Patterns:** `LOCF`, `ffill`, `last(ignorenulls=True)`

### Problem
`sensor_readings(sensor_id, reading_date, value)`. Replace NULL value with most recent prior non-null per sensor. Leading NULLs stay NULL.

### PySpark
```python
from pyspark.sql import functions as F, Window

w = Window.partitionBy("sensor_id").orderBy("reading_date") \
          .rowsBetween(Window.unboundedPreceding, Window.currentRow)

result = (
    sensor_readings
    .withColumn("value", F.last("value", ignorenulls=True).over(w))
    .orderBy("sensor_id","reading_date")
)
```

### Python (pandas)
```python
import pandas as pd

df = sensor_readings.sort_values(["sensor_id","reading_date"]).copy()
df["value"] = df.groupby("sensor_id")["value"].ffill()  # leading NULLs stay NaN
result = df.reset_index(drop=True)
```

### SQL (PostgreSQL)
```sql
SELECT sensor_id, reading_date,
    LAST_VALUE(value IGNORE NULLS)
        OVER (PARTITION BY sensor_id ORDER BY reading_date
              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS value
FROM sensor_readings ORDER BY sensor_id, reading_date;
```

### Complexity — O(n log n) | Space O(n)

### Follow-ups
**Q: What does pandas `groupby().ffill()` do?**  
A: Within each group (per sensor), propagates the last non-NaN value forward. Leading NaN values (no prior non-null) remain NaN — matching the problem requirement.

**Q: How do you backward fill (use next non-null value) in pandas?**  
```python
df.groupby("sensor_id")["value"].bfill()
```

**Q: How do you fill remaining NaN after ffill (fill leading NULLs with a default)?**  
```python
df["value"].ffill().fillna(0)   # replace any remaining NaN with 0
```

---

## Q76 — Department Salary Summary (department-salary-aggregation)
**Difficulty:** Medium | **Patterns:** `groupBy`, `NULL-safe agg`, `COUNT(*) vs count(col)`

### Problem
`employees(emp_id, name, department, salary, city)`. Per department: `total_salary`, `avg_salary` (2dp), `employee_count` (all rows incl. NULL salary). Order by department.

### PySpark
```python
from pyspark.sql import functions as F

result = (
    employees.groupBy("department")
    .agg(
        F.sum("salary").alias("total_salary"),
        F.round(F.avg("salary"),2).alias("avg_salary"),
        F.count("*").alias("employee_count")
    )
    .orderBy("department")
)
```

### Python (pandas)
```python
import pandas as pd

result = (
    employees.groupby("department", as_index=False)
    .agg(
        total_salary=("salary","sum"),        # sum skips NaN
        avg_salary=("salary","mean"),         # mean skips NaN
        employee_count=("emp_id","count")     # count non-null emp_ids = all rows (PK never null)
    )
    .assign(avg_salary=lambda d: d.avg_salary.round(2))
    .sort_values("department")
    .reset_index(drop=True)
)
```

### SQL (PostgreSQL)
```sql
SELECT department,
    SUM(salary) AS total_salary,
    ROUND(AVG(salary)::NUMERIC,2) AS avg_salary,
    COUNT(*) AS employee_count
FROM employees GROUP BY department ORDER BY department;
```

### Complexity — O(n log n) | Space O(k)

### Follow-ups
**Q: How do you count employees WITH vs WITHOUT a salary in pandas?**  
```python
employees.groupby("department")["salary"].agg(["count","size"])
# count = non-NaN; size = all rows
```

**Q: How do you add a HAVING equivalent in pandas (filter after groupby)?**  
```python
result.query("employee_count > 2")
```

---

## Q77 — Index Strategy Scoring (index-strategy-performance)
**Difficulty:** Hard | **Patterns:** `explode`, `UNNEST equivalent`, `Custom sort`, `Priority score`

### Problem
`slow_queries(query_id, table_name, where_columns, join_columns, order_columns, execution_time_ms, row_count)`. Explode comma-separated columns per usage_type. Aggregate frequency, avg_execution_time, priority_score = freq×avg_time/1000 rounded 2dp. Order by score DESC, then usage_type (order→where→join), then column_name.

### PySpark
```python
from pyspark.sql import functions as F

def explode_usage(df, col_name, label):
    return (df.filter(F.col(col_name).isNotNull() & (F.col(col_name) != ""))
              .withColumn("column_name", F.explode(F.split(F.trim(F.col(col_name)), r",\s*")))
              .withColumn("usage_type", F.lit(label))
              .select("query_id","table_name","column_name","usage_type","execution_time_ms"))

exploded = (explode_usage(slow_queries,"where_columns","where")
            .unionAll(explode_usage(slow_queries,"join_columns","join"))
            .unionAll(explode_usage(slow_queries,"order_columns","order")))

result = (
    exploded.groupBy("table_name","column_name","usage_type")
    .agg(
        F.count("*").alias("frequency"),
        F.avg("execution_time_ms").alias("avg_execution_time"),
        F.round(F.count("*")*F.avg("execution_time_ms")/1000,2).alias("priority_score")
    )
    .withColumn("ut_ord", F.when(F.col("usage_type")=="order",1)
                            .when(F.col("usage_type")=="where",2).otherwise(3))
    .orderBy(F.desc("priority_score"),"ut_ord","column_name")
    .drop("ut_ord")
)
```

### Python (pandas)
```python
import pandas as pd

def explode_col(df, col, label):
    tmp = df[df[col].notna() & (df[col]!="")][["query_id","table_name","execution_time_ms",col]].copy()
    tmp["column_name"] = tmp[col].str.split(r",\s*", regex=True)
    tmp = tmp.explode("column_name")
    tmp["column_name"] = tmp["column_name"].str.strip()
    tmp["usage_type"] = label
    return tmp[["query_id","table_name","column_name","usage_type","execution_time_ms"]]

exploded = pd.concat([
    explode_col(slow_queries,"where_columns","where"),
    explode_col(slow_queries,"join_columns","join"),
    explode_col(slow_queries,"order_columns","order"),
], ignore_index=True)

result = (
    exploded.groupby(["table_name","column_name","usage_type"], as_index=False)
    .agg(frequency=("query_id","count"),
         avg_execution_time=("execution_time_ms","mean"))
    .assign(priority_score=lambda d: (d.frequency*d.avg_execution_time/1000).round(2))
)

usage_order = {"order":1,"where":2,"join":3}
result["ut_ord"] = result["usage_type"].map(usage_order)
result = (result.sort_values(["priority_score","ut_ord","column_name"], ascending=[False,True,True])
                .drop(columns="ut_ord").reset_index(drop=True))
```

### SQL (PostgreSQL)
```sql
WITH exploded AS (
    SELECT query_id,table_name,TRIM(col) AS column_name,'where' AS usage_type,execution_time_ms
    FROM slow_queries, UNNEST(STRING_TO_ARRAY(where_columns,',')) col WHERE where_columns IS NOT NULL AND where_columns<>''
    UNION ALL
    SELECT query_id,table_name,TRIM(col),'join',execution_time_ms
    FROM slow_queries, UNNEST(STRING_TO_ARRAY(join_columns,',')) col WHERE join_columns IS NOT NULL AND join_columns<>''
    UNION ALL
    SELECT query_id,table_name,TRIM(col),'order',execution_time_ms
    FROM slow_queries, UNNEST(STRING_TO_ARRAY(order_columns,',')) col WHERE order_columns IS NOT NULL AND order_columns<>''
)
SELECT table_name,column_name,usage_type,
    COUNT(*) AS frequency, AVG(execution_time_ms) AS avg_execution_time,
    ROUND((COUNT(*)*AVG(execution_time_ms)/1000)::NUMERIC,2) AS priority_score
FROM exploded GROUP BY table_name,column_name,usage_type
ORDER BY priority_score DESC,
    CASE usage_type WHEN 'order' THEN 1 WHEN 'where' THEN 2 ELSE 3 END,
    column_name;
```

### Complexity — O(n × m × log(nm)) | Space O(n × m)

### Follow-ups
**Q: What does pandas `.explode()` do?**  
A: Converts a column containing lists into multiple rows — one row per list element. The equivalent of SQL `UNNEST`.

**Q: How do you deduplicate column names within a single query in pandas?**  
```python
tmp["column_name"] = tmp[col].str.split(r",\s*").apply(lambda x: list(dict.fromkeys(x)))
tmp = tmp.explode("column_name")  # after dedup within each query
```

**Q: How would you visualize which (table, column) pairs have the highest priority scores?**  
```python
import matplotlib.pyplot as plt
result.head(10).plot(x="column_name", y="priority_score", kind="barh")
plt.title("Top Index Candidates by Priority Score")
plt.show()
```

---

## Master Reference — All Three Languages

| Pattern | PySpark | pandas | SQL |
|---------|---------|--------|-----|
| Select columns | `.select("a","b")` | `df[["a","b"]]` | `SELECT a, b` |
| Filter rows | `.filter(cond)` | `df[cond]` / `.query()` | `WHERE` |
| Add column | `.withColumn("n", expr)` | `df["n"] = expr` / `.assign()` | `SELECT …, expr AS n` |
| Group + agg | `.groupBy().agg()` | `.groupby().agg()` | `GROUP BY … agg()` |
| Sort | `.orderBy(F.desc("col"))` | `.sort_values("col", ascending=False)` | `ORDER BY col DESC` |
| Join | `.join(other, on, how)` | `.merge(other, on, how)` | `JOIN … ON … ` |
| Union | `.unionAll()` | `pd.concat([df1,df2])` | `UNION ALL` |
| Dedup | `row_number().over(w)==1` | `.drop_duplicates()` | `ROW_NUMBER() … = 1` |
| Cross join | `.crossJoin()` | `.merge(how="cross")` | `CROSS JOIN` |
| Explode list | `F.explode(F.split(col,","))` | `col.str.split(",").explode()` | `UNNEST(STRING_TO_ARRAY(col,","))` |
| Pivot | `.pivot(cat,vals).agg()` | `.pivot_table(values,index,cols,aggfunc)` | `SUM(CASE WHEN cat=X THEN val END)` |
| Forward fill | `F.last(col,ignorenulls=True).over(w)` | `.groupby().ffill()` | `LAST_VALUE(col IGNORE NULLS) OVER` |
| Rank | `F.rank().over(w)` | `.groupby().rank(method="min")` | `RANK() OVER` |
| Dense Rank | `F.dense_rank().over(w)` | `.groupby().rank(method="dense")` | `DENSE_RANK() OVER` |
| Std dev (sample) | `F.stddev_samp()` | `.std(ddof=1)` | `STDDEV_SAMP()` |
| Median | `F.percentile_approx(col,0.5)` | `.median()` | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col)` |
| String split | `F.split(col," ")[0]` | `col.str.split(" ").str[0]` | `SPLIT_PART(col,' ',1)` |
| Regex match | `col.rlike(pattern)` | `col.str.match(pattern)` / `.str.contains()` | `col ~ pattern` / `LIKE` |
| Date diff | `F.datediff(end,start)` | `(end-start).dt.days` | `end - start` (integer days in PG) |
| Null coalesce | `F.coalesce(col,F.lit(0))` | `col.fillna(0)` | `COALESCE(col,0)` |
