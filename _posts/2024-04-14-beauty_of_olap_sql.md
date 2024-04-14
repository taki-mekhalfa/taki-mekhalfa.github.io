---
layout: post
title: "The beauty of OLAP SQL"
categories: misc
---

In my daily job, I develop backends for analytical dashboards that show graphs to users to help them better understand their data.
I want to share with you a real case scenario using SQL to write powerful OLAP queries.

Imagine you sell Pizza brands, and you are interested in comparing with your competitors to see how much of the pizza shelf your products are occupying. Some interesting questions for you would be:
* What is your share of assortment in the pizza category? Is **50%** of distributed pizza products on Walmart yours? Is it only **5%**?
* On average, how many of your pizza products are distributed in Kroger? You have **5** different products but only **2** are available on average? Maybe more/less?
* How is your share of the assortment evolving in time? Are you **+10%** or **-5%** with respect to the last year? last month?
* Who are the competitors detaining the majority of the shelf? How well do you compare with them?
* Who is putting new products the market? Who is putting less?
* Which of your brands is successful? Which one needs to be taken out of the market? or enhanced with ads/better prices, etc...


The data looks like the following (you can find it in a parquet file in [here](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/1-olap/olap.parquet)):

```
┌───────┬──────────────┬──────────┬───────┬──────────┬─────────┬──────────────────┐
│ p_id  ┆ manufacturer ┆ category ┆ brand ┆ retailer ┆ period  ┆ distributed_days │
│ ---   ┆ ---          ┆ ---      ┆ ---   ┆ ---      ┆ ---     ┆ ---              │
│ i64   ┆ i64          ┆ i64      ┆ i64   ┆ i64      ┆ str     ┆ i64              │
╞═══════╪══════════════╪══════════╪═══════╪══════════╪═════════╪══════════════════╡
│ 6581  ┆ 61           ┆ 20       ┆ 273   ┆ 1        ┆ 2023_52 ┆ 31407            │
│ 8563  ┆ 34           ┆ 4        ┆ 274   ┆ 1        ┆ 2023_52 ┆ 27515            │
│ 8411  ┆ 11           ┆ 6        ┆ 276   ┆ 1        ┆ 2023_52 ┆ 30693            │
│ 20163 ┆ 21           ┆ 4        ┆ 277   ┆ 1        ┆ 2023_52 ┆ 61               │
│ 5669  ┆ 56           ┆ 4        ┆ 278   ┆ 1        ┆ 2023_52 ┆ 20665            │
│ …     ┆ …            ┆ …        ┆ …     ┆ …        ┆ …       ┆ …                │
│ 8932  ┆ 2            ┆ 23       ┆ 2     ┆ 1        ┆ 2024_1  ┆ 25584            │
│ 8927  ┆ 2            ┆ 2        ┆ 2     ┆ 1        ┆ 2024_1  ┆ 27163            │
│ 8923  ┆ 2            ┆ 14       ┆ 2     ┆ 1        ┆ 2024_1  ┆ 21691            │
│ 8918  ┆ 2            ┆ 3        ┆ 2     ┆ 1        ┆ 2024_1  ┆ 31407            │
│ 8910  ┆ 2            ┆ 3        ┆ 2     ┆ 1        ┆ 2024_1  ┆ 17               │
└───────┴──────────────┴──────────┴───────┴──────────┴─────────┴──────────────────┘
```

This is anonymized data and represents a subset of scraped products from retailers websites (Walmart, Kroger, ...).
In this specific case, we are interested in analyzing the share of assortment of each brand manufacturer with respect to different variables.

Each line contains data about a product:
* Unique ID of the product
* The Manufacturer's ID
* Categorization of the product : `Toothpaste`, `Deodorants`, `Sauces`, ...
* Brand of the product, for example: Dove, Pepsi, KitKat, ...
* Retailer and the week where data was scraped. Data here represents Last week of 2023 and first week of 2024.
* Distributed days: The number of the days we've seen the product on the website during the period and over all shops of that retailer. For example if Walmart has **1000** shops in the US and the product was available on all shops during the entire week, the number of distributed days would be **1000 * 7 = 7000** days. We will see how is this used to compute share of assortment later.

In the following I will be using **[DuckDB](https://duckdb.org/)**. An amazing and powerful embedded query engine.

#### 1) How many unique products distributed by manufacturer for both weeks over all retailers?

This an easy query, we just have to take distinct products and group by the manufacturer

```sql
SELECT
    manufacturer,
    COUNT(DISTINCT p_id) AS nb_unique_products -- the number of unique products
FROM
    product_details
GROUP BY
    ALL -- A powerful construct provided by DuckDB
ORDER BY
    nb_unique_products DESC
LIMIT
    10;
-- ┌──────────────┬────────────────────┐
-- │ manufacturer │ nb_unique_products │
-- │    int64     │       int64        │
-- ├──────────────┼────────────────────┤
-- │            7 │               3463 │ <-- The big players
-- │           34 │               2688 │
-- │            2 │               2368 │
-- │           26 │               1227 │ 
-- │           39 │                978 │
-- │            6 │                783 │
-- │           58 │                727 │
-- │           21 │                499 │
-- │           57 │                372 │ <-- Let's say this is me
-- │            3 │                333 │ 
-- ├──────────────┴────────────────────┤
-- │ 10 rows                 2 columns │
-- └───────────────────────────────────┘
```

#### 2) How are my products distributed over the different categories?

Filter on my products only, and then count the number of unique products grouping by the product category.


It's good to see the percentage each category accounts for. A window function is an amazing tool.
It applies an aggregate like computation over a window of rows.

<u>Important</u>: The rows considered by a window function are those produced WHERE, GROUP BY, and HAVING clauses if any. You can think of it as if the window function was run the last.

`nb_unique_products / SUM(nb_unique_products) OVER ()` means:

> Take the current row's `nb_unique_products`and divide it by
> the sum of `nb_unique_products` of all rows in the window. The window here is all categories. 

```sql
SELECT
    category,
    COUNT(DISTINCT p_id) AS nb_unique_products,
    ROUND(
        100 * nb_unique_products / SUM(nb_unique_products) OVER (),
        2
    ) AS "% of products"
FROM
    product_details
WHERE
    manufacturer = 57
GROUP BY
    category
ORDER BY
    nb_unique_products DESC

-- ┌──────────┬────────────────────┬───────────────┐
-- │ category │ nb_unique_products │ % of products │
-- │  int64   │       int64        │    double     │
-- ├──────────┼────────────────────┼───────────────┤
-- │        4 │                203 │         54.57 │< products concentrated on
-- │        3 │                 50 │         13.44 │< these two categories.
-- │       18 │                 23 │          6.18 │
-- │       16 │                 22 │          5.91 │
-- │       22 │                 22 │          5.91 │
-- │        2 │                 22 │          5.91 │
-- │        1 │                 18 │          4.84 │
-- │       11 │                 10 │          2.69 │
-- │        5 │                  2 │          0.54 │
-- └──────────┴────────────────────┴───────────────┘
```

Now, imagine you are responsible for categories **1**, **2**, **3** and **4** only.
You don't want to see other categories. You might be tempted by doing:

```sql
SELECT
    category,
    COUNT(DISTINCT p_id) AS nb_unique_products,
    ROUND(
        100 * nb_unique_products / SUM(nb_unique_products) OVER (),
        2
    ) AS "% of products"
FROM
    product_details
WHERE
    manufacturer = 57
    AND category IN (1, 2, 3, 4) -- You added this
GROUP BY
    category
ORDER BY
    nb_unique_products DESC;

-- which gives:
-- ┌──────────┬────────────────────┬───────────────┐
-- │ category │ nb_unique_products │ % of products │
-- │  int64   │       int64        │    double     │
-- ├──────────┼────────────────────┼───────────────┤
-- │        4 │                203 │         69.28 │
-- │        3 │                 50 │         17.06 │
-- │        2 │                 22 │          7.51 │
-- │        1 │                 18 │          6.14 │
-- └──────────┴────────────────────┴───────────────┘
```

What happened? Share of products went from: ***54.57%*** to ***69.28%***.
As I said, window function is evaluated on rows after the `WHERE` and `GROUP BY` clauses were executed. In this case, it means you are considering the share on only these categories (**1, 2, 3 and 4**).

There are two ways of doing this:

* Use a sub query:
```sql
SELECT
    *
FROM
   (<PREVIOUS QUERY>)
WHERE
    category IN (1, 2, 3, 4);
```
* Use **QUALIFY** provided by DuckDB:
```sql
SELECT
    category,
    COUNT(DISTINCT p_id) AS nb_unique_products,
    ROUND(
        100 * nb_unique_products / SUM(nb_unique_products) OVER (),
        2
    ) AS "% of products"
FROM
    product_details
WHERE
    manufacturer = 57 
GROUP BY
    category
QUALIFY -- At last, filter on window function result using the following:
    category IN (1, 2, 3, 4)
ORDER BY
    nb_unique_products DESC
-- ┌──────────┬────────────────────┬───────────────┐
-- │ category │ nb_unique_products │ % of products │
-- │  int64   │       int64        │    double     │
-- ├──────────┼────────────────────┼───────────────┤
-- │        4 │                203 │         54.57 │
-- │        3 │                 50 │         13.44 │
-- │        2 │                 22 │          5.91 │
-- │        1 │                 18 │          4.84 │
-- └──────────┴────────────────────┴───────────────┘
```

#### 3) Since I am competing on categories 3 and 4, who am I competing with?

We have seen that we are distributing products mainly in two catagories (for example: `Toothpaste` and `Mouthwash`)
I should care more about competitors on these categories only.

Let's start simple:

```sql
SELECT 
    category,
    manufacturer,
    COUNT( DISTINCT p_id) as nb_unique_products
FROM product_details
WHERE category IN (3, 4)
GROUP BY ALL
ORDER BY category, nb_unique_products DESC
LIMIT 5;
-- ┌──────────┬──────────────┬────────────────────┐
-- │ category │ manufacturer │ nb_unique_products │
-- │  int64   │    int64     │       int64        │
-- ├──────────┼──────────────┼────────────────────┤
-- │        3 │            2 │                196 │
-- │        3 │           39 │                185 │
-- │        3 │           41 │                124 │
-- │        3 │           67 │                119 │
-- │        3 │           34 │                 92 │
-- └──────────┴──────────────┴────────────────────┘
```
This gives us the number of unique products distributed by all manufacturers in categories 3 and 4.

Let's try to only get the **top 3** manufacturers on each category, which are competitors we need to worry about the most.

```sql
SELECT
    category,
    manufacturer,
    COUNT(DISTINCT p_id) as nb_unique_products, -- over the category+manufacturer,
    row_number() OVER (
        PARTITION BY category -- rank inside the category
        ORDER BY -- more products = better position 
            nb_unique_products DESC
    ) as position,
    ROUND(
        100 * nb_unique_products / SUM(nb_unique_products) OVER (
            PARTITION BY category
        ),
        2
    ) as "% of the market"
FROM
    product_details
WHERE
    category IN (3, 4)
GROUP BY
    category,
    manufacturer 
QUALIFY 
    position <= 3 -- top 3 competitors
    OR manufacturer = 57 -- and me please
ORDER BY
    category,
    position
-- ┌──────────┬──────────────┬────────────────────┬──────────┬─────────────────┐
-- │ category │ manufacturer │ nb_unique_products │ position │ % of the market │
-- │  int64   │    int64     │       int64        │  int64   │     double      │
-- ├──────────┼──────────────┼────────────────────┼──────────┼─────────────────┤
-- │        3 │            2 │                196 │        1 │           18.23 │
-- │        3 │           39 │                185 │        2 │           17.21 │
-- │        3 │           41 │                124 │        3 │           11.53 │
-- │        3 │           57 │                 50 │        7 │            4.65 │< this is me
-- │        4 │           34 │                452 │        1 │           23.12 │
-- │        4 │            7 │                303 │        2 │            15.5 │
-- │        4 │            2 │                278 │        3 │           14.22 │
-- │        4 │           57 │                203 │        5 │           10.38 │< this is me
-- └──────────┴──────────────┴────────────────────┴──────────┴─────────────────┘
```

So, this allows you to see for example that you are 5th on category 4 and allows you to see
who are the ones with the highest share on it.

Let's deconstruct this query:
1. Filter on categories **3** and **4**: `WHERE`
2. Get the number of unique products per manufacturer per category: `GROUP BY`
3. Rank each manufacturer on the category: `row_number()` over a window *partitioned by* category and *ordered* by the number of unique products we compute at **step 1**. We have to partition by category to not take both categories into account. So each manufacturer will have 2 positions: one on **category 2** and another one on **category 3**
4. Just like **step 3**, compute the share of products for a manufacturer on a given category.
5. `QUALIFY` to only return **top 3** manufacturers

#### 4) Distributed products at multi-levels?

Imagine you want the number of your unique products:
* Per retailer per category
* Per retailer for all categories
* Per category for all retailers
* For all retailers for all categories

You can compute each one independently and union everything but there is a simpler way.

```sql
SELECT
    COALESCE(retailer, 0) AS retailer, -- convert nulls to a readable id
    COALESCE(category, 0) AS category, -- convert nulls to a readable id
    COUNT(DISTINCT p_id) AS nb_unique_products
FROM
    product_details
GROUP BY
    GROUPING SETS (
        -- per retailer per category
        (category, retailer),
        -- per category for all retailers
        (category),
        -- per retailer for all categories
        (retailer),
        -- for all categories for all retailers
        () 
    )
ORDER BY
    ALL
-- ┌──────────┬──────────┬────────────────────┐
-- │ retailer │ category │ nb_unique_products │
-- │  int64   │  int64   │       int64        │
-- ├──────────┼──────────┼────────────────────┤
-- │        0 │        0 │              17428 │
-- │        0 │        1 │                136 │
-- │        0 │        2 │               2105 │
-- │        0 │        3 │               1075 │
-- │        0 │        4 │               1955 │
-- │        · │        · │                 ·  │
-- │        · │        · │                 ·  │
-- │        · │        · │                 ·  │
-- │        2 │       11 │                346 │
-- ├──────────┴──────────┴────────────────────┤
-- │ x rows (xx shown)              3 columns │
-- └──────────────────────────────────────────┘
```

Grouping sets is equivalent to the union of different `GROUP BY`s.

The result will contain `NULL` for the dimension not included in the grouping set. So results for the group `(category)` will have `retailer` set as `NULL`.

If you are using all possible grouping sets, than you can simplify syntax by using `CUBE`:

```sql
SELECT
    COALESCE(retailer, 0) AS retailer,
    COALESCE(category, 0) AS category,
    COUNT(DISTINCT p_id) AS nb_unique_products
FROM
    product_details
GROUP BY
    -- This is equivalent to all possible grouping sets:
    -- (category, retailer), (category), (retailer), (category), ()
    CUBE(retailer, category)
ORDER BY
    ALL
```

`ROLLUP` is another grouping construct but it works differently than `CUBE` by restricting the 
possible grouping sets. `ROLLUP(x, y)` is defined as:
* `(x, y)`
* `(x)`
* `()`

`(y)` would be included by `CUBE` but not by `ROLLUP`.

This is more useful in cases when there is a hierarchical relationship between
the different dimensions as in: 

- `SUM(population) GROUP BY ROLLUP(county, region, city)`

In which case the grouping set `(region)` won't make a lof of sense as a region is linked to one country and that would be equivalent to `(country, region)`.

#### 5) Since I am competing on categories 3 and 4, what's my share of assortment (SOA)?

We have seen a query to look at the number of unique distributed products in categories **3 and 4**.

Having more unique distributed products is not completely correlated with your share of assortment.

Imagine you have **15** unique frozen food products and
a competitor who has only **3** unique frozen food products, and there is only you too on the market.


If at a given retailer (Walmart for e.g.), they always put **2** of you competitor's products and **1** of your products. In this case you will only have ***1/3=33%*** of SOA while your competitor will have ***2/3=77%*** of the frozen food shelf. Although you detain a big share on the number of unique products ***15/18=83%***, you are being surpassed by your competitor.

Share of assortment can be defined as:

```
SOA = SUM(distributed days of my products) / SUM(distributed days of my products + competitors' products)
```

Here is a query to get top 3 competitors in categories 3 and 4 with respect to SOA:

```sql
WITH soa_per_category_per_manufacturer AS (
    SELECT
        category,
        manufacturer,
        SUM(distributed_days) as _total_ddays, -- total ddays per each manufacturer
        ROUND(
            100 * _total_ddays / SUM(_total_ddays) OVER (PARTITION BY category),
            2
        ) as soa -- this is exactly the SOA formula,
    FROM
        product_details
    WHERE
        category IN (3, 4)
    GROUP BY
        category,
        manufacturer
)
SELECT
    * EXCLUDE(_total_ddays), -- Thanks DuckDB for EXCLUDE :) 
    row_number() OVER (
        PARTITION BY category
        ORDER BY
            soa DESC
    ) as position -- order then by soa on the category
FROM
    soa_per_category_per_manufacturer 
QUALIFY 
    position <= 3 -- top 3 competitors
    OR manufacturer = 57 -- and me please

-- ┌──────────┬──────────────┬────────┬──────────┐
-- │ category │ manufacturer │  soa   │ position │
-- │  int64   │    int64     │ double │  int64   │
-- ├──────────┼──────────────┼────────┼──────────┤
-- │        3 │           39 │  22.12 │        1 │
-- │        3 │            2 │  16.04 │        2 │
-- │        3 │           67 │  14.36 │        3 │
-- │        3 │           57 │   3.95 │        8 │< I am no longer 7th on the market
-- │        4 │           34 │   24.6 │        1 │
-- │        4 │            7 │  17.27 │        2 │
-- │        4 │            2 │   10.9 │        3 │
-- │        4 │           57 │  10.36 │        5 │< Still 5th
-- └──────────┴──────────────┴────────┴──────────┘
```

This query uses `CTE`s which are a way to organize your queries to make them readable 
and in some cases avoid recomputing the same results over and over 
if materialized and used in many places (check [Materialized CTEs](https://duckdb.org/docs/sql/query_syntax/with.html#materialized-ctes)).

#### 6) Temporal evolution!

Imagine now you want to compare SOA on your categories between the selected week `2024_1` with respect to previous week `2023_52`.

```sql
WITH soa_per_period_category_per_manufacturer AS (
    SELECT
        period,
        category,
        manufacturer,
        SUM(distributed_days) as _total_ddays,
        ROUND(
            100 * _total_ddays / SUM(_total_ddays) OVER (
                PARTITION BY 
                period, -- notice adding period here :)
                category
            ),
            2
        ) as soa,
    FROM
        product_details
    GROUP BY
        period -- and here too :),
        category,
        manufacturer
),
soa_variation AS (
    SELECT
        * EXCLUDE (_total_ddays),
        -- notice the use of a defined window to
        -- simplify the query especially if it's used in many places.
        soa - lag(soa, 1, 0) OVER period_window as variation_soa,
    FROM
        soa_per_period_category_per_manufacturer
        -- define the window and assign a name to it
        WINDOW period_window AS (
            PARTITION BY category,
            -- Notice that we need manufacturer here
            -- since the variation is by manufacturer by category
            manufacturer
            ORDER BY
                -- this is very important as variation is
                -- current soa - previous soa. The order
                -- of the values in the window should be like that
                -- to be able to use soa - lag(soa, 1, 0).
                -- notice also that '2023_52' < '2024_1'
                period 
        )
        -- take this week only after the window function
        -- as tha variation is linked to it
        QUALIFY period = '2024_1'
    ORDER BY
        period
)
SELECT
    -- thanks DuckDB for EXCLUDE/REPLACE :)
    * EXCLUDE (period) REPLACE (ROUND(variation_soa, 2) as variation_soa),
    row_number() OVER (
        PARTITION BY category
        ORDER BY
            variation_soa DESC
    ) as position -- order then by soa on the category
FROM
    -- take first from each category
    soa_variation QUALIFY position = 1
ORDER BY
    category
LIMIT
    10;

-- ┌──────────┬──────────────┬────────┬───────────────┬──────────┐
-- │ category │ manufacturer │  soa   │ variation_soa │ position │
-- │  int64   │    int64     │ double │    double     │  int64   │
-- ├──────────┼──────────────┼────────┼───────────────┼──────────┤
-- │        1 │           39 │  34.19 │          3.11 │        1 │< 39 is getting better
-- │        2 │           39 │  11.68 │          1.37 │        1 │< over multiple categories
-- │        3 │           39 │   24.8 │          5.64 │        1 │
-- │        4 │            6 │  11.84 │          2.41 │        1 │
-- │        5 │           39 │  40.07 │          6.52 │        1 │
-- │        6 │           39 │  42.08 │         10.42 │        1 │< That's a good improvement
-- │        7 │           34 │   6.29 │          3.65 │        1 │
-- │        8 │            6 │  51.29 │          0.86 │        1 │
-- │        9 │            3 │  52.69 │          9.33 │        1 │
-- │       10 │            3 │  42.24 │          6.81 │        1 │
-- ├──────────┴──────────────┴────────┴───────────────┴──────────┤
-- │ 10 rows                                           5 columns │
-- └─────────────────────────────────────────────────────────────┘
```

This query get's the manufacturer with the best variation over each category. 
Notice the multiple `CTE`s to organize the query. 

`lag(n)` is a window function to allow using the value of the previous `n` row inside the window.

#### 6) Not all retailers are equal!

Imagine you have SOA by retailer, and you want an average but not just a normal average,
because retailers don't have the same importance (number of shops, zones, contracts, ...) you would
like to compute the weighted average over the different retailers.

Imagine you have these weights (you can find the data [here](https://github.com/taki-mekhalfa/taki-mekhalfa.github.io/raw/main/assets/posts/1-olap/retailer_w.parquet)):

```sql
select * from retailer_w;
-- ┌──────────┬────────┐
-- │ retailer │   w    │
-- │  int32   │ double │
-- ├──────────┼────────┤
-- │       2  │    1.0 │
-- │       1  │    3.0 │< 3 times more important than retailer 2
-- └──────────┴────────┘
```

Average SOA would be then: 

```
avg_soa = SUM(retailer_i * w_i) / SUM(w_i) for all (retailer_i, w_i) 
```

```sql
WITH soa_per_retailer_per_manufacturer AS (
    SELECT
        retailer,
        manufacturer,
        SUM(distributed_days) as _total_ddays,
        -- total ddays per each manufacturer
        ROUND(
            100 * _total_ddays / SUM(_total_ddays) OVER (PARTITION BY retailer),
            2
        ) as soa -- this is exactly the SOA formula,
    FROM
        product_details
    GROUP BY
        retailer,
        manufacturer
)
SELECT
    manufacturer,
    ROUND(
        -- we are grouping my manufacturer so we can just
        -- do this to have the weighted average
        SUM(soa * w) / SUM(w),
        2
    ) as avg_soa,
FROM
    soa_per_retailer_per_manufacturer
    JOIN retailer_w USING(retailer)
GROUP BY
    manufacturer
ORDER BY avg_soa DESC
LIMIT 5;
-- ┌──────────────┬─────────┐
-- │ manufacturer │ avg_soa │
-- │    int64     │ double  │
-- ├──────────────┼─────────┤
-- │            7 │   22.52 │
-- │           34 │   16.03 │
-- │            2 │    12.8 │
-- │           26 │    7.56 │
-- │           39 │    6.72 │
-- └──────────────┴─────────┘
```

### That's it. Hope you had fun reading this :)
### Taki