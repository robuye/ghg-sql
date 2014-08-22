# Common Table Expressions (CTE) (also known as WITH queries)

It's almost syntatic sugar for making big queries with nested subqueries easier to read.

Given following (small) query:
```
SELECT * FROM (
  SELECT * FROM country
    INNER JOIN city ON city.id = country.capital
    WHERE city.population > 500000
) countries_with_capital
WHERE indepyear > 1960;
```

it can be rewritten using CTE:

```
WITH countries_with_capital AS (
  SELECT * FROM country
    INNER JOIN city ON city.id = country.capital
    WHERE city.population > 500000
)

SELECT * FROM countries_with_capital
WHERE indepyear > 1960;
```

it's longer but easier to read.

We can use as many CTEs as we want.

Downside: it forces Query Planner to materialize CTE tables, as
a result performance may not be optimal. Usually it's not an issue.

# WINDOW FUNCTIONS

It allows to aggregate data across multiple rows into single column.
It's super-duper useful and fast.

Imagine we want to rank countries based on popularity of official language.

```

WITH countries_with_language as (
SELECT * FROM country c
  INNER JOIN countrylanguage cl ON c.code = cl.countrycode
)
SELECT name, language, percentage, rank() OVER (ORDER BY percentage DESC)
FROM countries_with_language;
```

That is good, but we could do better.
Imagine we want to rank countries based on popularity of official language but
have separate ranking for each language.

```
WITH countries_with_language as (
SELECT * FROM country c
  INNER JOIN countrylanguage cl ON c.code = cl.countrycode
)
SELECT name, language, percentage, rank() OVER (PARTITION BY language ORDER BY percentage DESC)
FROM countries_with_language
WHERE percentage > 50 --filter silly results
ORDER BY language, rank;
```


many many Window Functions are available, for details see documentation:
http://www.postgresql.org/docs/9.1/static/functions-aggregate.html
(there is great explanation how it works with simple examples)

# PERFORMANCE

Any query that takes > 50ms is slow (unless it's fast).

Index everything.

Don't ever use `like` queries. If you need to remember to add `xxx_patterns_ops`, i.e.

```
CREATE INDEX test_index ON test_table (column text_pattern_ops);
```

Never use `like %text%` queries, those are impossible to index. Better use FullTextSearch.

Don't do multicolumn indexes and if you do ALWAYS define scope in ActiveRecord. Multicolumn
indexes need correct order of where conditions.

Always check queries if they use indexes by doing `explain analyze`. If query plan includes `Seq Scan` it
always can be optimized.

Have good test data (more is better). With 5 rows it will be more optimal to scan table than use index.

If you have HUGE datasets don't do offset based pagination, it's getting slower over time.


# EXPRESSIONS
Given that there are records with NULL in "gnpold" what will be result of this query?

(protip: `coalesce` will replace `null` with 0)

```
SELECT * FROM country
WHERE gnp / coalesce(gnpold, 0) > 1
  AND gnpold IS NOT NULL;
```

and this one?

```
SELECT * FROM country
WHERE gnpold IS NOT NULL
  AND gnp / coalesce(gnpold, 0) > 1;
```

and this one?

```
SELECT * FROM country
WHERE gnp / coalesce(gnpold, 0) > 1;
```

Answer:

¯\_(ツ)_/¯


Expression conditions are not evaluated from-left-to-right. Query Planner will reorder as it thinks.
Relying on order of expression is bad idea.
Having functions with side-effects is terrible idea.

If above examples work (they should) it's only because we are lucky.

To force expression order we need to use CASE statement, i.e.

```
SELECT * FROM country
WHERE CASE WHEN gnpold IS NULL THEN false ELSE gnp / gnpold > 1 END;
```

Query Planner won't try to optimise this so it's usually not good idea.



## USEFUL QUERIES FOR FUN AND PROFIT

#Find all duplicate records (using Window Functions)

Selects all duplicate cities (based on ID) in each country:

```
SELECT * FROM (
  SELECT *, row_number() over(PARTITION BY countrycode ORDER BY id) AS row_number
  FROM city
) cities_with_row_number
WHERE row_number > 1
ORDER BY countrycode, id;
```


Can be rewritten with CTE as:

```
WITH cities_with_row_number AS (
  SELECT *, row_number() over(PARTITION BY countrycode ORDER BY id) AS row_number
  FROM city
)
SELECT * FROM cities_with_row_number
WHERE row_number > 1
ORDER BY countrycode, id;
```

#Find records with missing (invalid) association

(lets invalidate association first)
```
UPADTE city SET countrycode = 'ZZZ' WHERE id = 1;
```

Now select all cities that have no matching country.

Using CTE:

```
WITH cities_with_countries as (
SELECT id FROM city
INNER JOIN country ON city.countrycode = country.code
)
SELECT * FROM city
WHERE id NOT IN (SELECT * FROM cities_with_countries);
```

Without CTE:
```
SELECT * FROM city
WHERE NOT EXISTS (SELECT 1 FROM country WHERE city.countrycode = country.code LIMIT 1);
```

`1` in SQL evaluates to `true`, it saves SQL from reading real data from disk.
We also `LIMIT 1` because we don't care how many results there are, aborts the query.


// from Community (funny usecase)

Find users who are likely abusing PrivateMessages system by sending over 20 pms per hour

```
WITH messages_per_hour AS (
  SELECT date_trunc('hour', created_at) AS hour,
         sender_id
  FROM messages
  WHERE created_at >= '#{@previous_run}' AND created_at < '#{@current_run}'
)
SELECT count(*) AS messages_count,
       sender_id,
       hour
FROM messages_per_hour
GROUP BY sender_id, hour
HAVING count(*) > 20
ORDER BY count(*) DESC;
```
