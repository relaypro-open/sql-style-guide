# SQL Style Guide

This style guide focuses on Postgres-based SQL dialects for the purpose of standardizing analytic queries to improve the consistency and ease of peer-reviewing and maintaining code.

## Naming Things

* Use lower case and underscores to name tables, views, CTEs, columns, etc. No spaces.
* Don't use [SQL keywords](https://www.postgresql.org/docs/current/sql-keywords-appendix.html) for naming things.
* If, for some reason (e.g., mirroring an external data source), you absolutely must use a SQL keyword as a name, always enclose it in double quotes (e.g., `"timestamp"`).
* When aliasing tables or subqueries, use short (1-5 character) aliases or don't alias. If using a single character, try to use a character that can be easily associated with the table/subquery in question. For example, use `c` for `customers`.

## White Space
### Spaces
* Before and after operators: `first_name = 'John'`
* After commas in function calls: `dateadd(day, 23, getdate())`

### New Lines
* Use after the SQL keyword introducing a SQL clause.
* Use after each individual column or equivalent in a SQL clause.
* Use _before_ logical operators in `where`, `having`, or `on` clauses.
* Use after the opening and closing parentheses for a CTE or subquery.

For example...
```sql
with the_cte as ( -- New line due to opening CTE parenthesis
    select -- New line due to SQL keyword introducing a new clause
        c.id as customer_id, -- New line after a column
        e.email
    from
        customers as c
    left join
        emails as e
    on
        c.id = e.customer -- New line due to logical operator in on clause
        and e.current = true
)
select
    email,
    count(*)
from
    the_cte
group by
    email
```

### Indentation
* Indent SQL code using four (4) spaces, not tabs
* Indent lists of columns or equivalents within each clause.
* Indent code for CTEs or subqueries within their parentheses.

For example...
```sql
with the_cte as (
    select -- Start indenting due to being within CTE parentheses
        c.id as customer_id, -- Indent further due to being a list of columns within select clause
        e.email
    from
        customers as c
    left join
        emails as e
    on
        c.id = e.customer
        and e.current = true
)
select
    email,
    count(*)
from
    the_cte
group by
    email
```

### White Space Discretion
* Use your discretion in dividing long case statements across multiple lines. If you do divide long case statements, it's preferable to start a new line for each `when` or `else`.
* If you have nested logic in a `where` clause (e.g., `foo = 1 and (bar = 2 or baz = 3)`), use discretion in splitting the inner clause over multiple lines. If it's short, one line is fine. If it's long, insert a line break and indent after the opening parenthesis, as you would format a subquery or CTE.
* Feel free to write very simple queries on a single line.

## CTEs
* For readability purposes, use [Common Table Expressions (CTEs)](https://www.postgresql.org/docs/current/queries-with.html) rather than subqueries to build complicated queries.
* Subqueries are generally unavoidable when filtering rows based on a particular column's presence in another table (e.g., `when email in (select email from blacklist)`). If the subquery in question starts getting too large, move it to a CTE and reference the CTE in the subquery rather than the original table. Use your discretion here.
* Never nest CTEs. This has all the disadvantages of both CTEs and subqueries with none of the advantages of either.
* CTEs are optimization fences in Postgres. If a CTE in Postgres is significantly hurting query performance, convert it to a subquery. Discretion.

For example...
```sql
with the_cte as ( -- CTE used instead of a subquery for readability.
    select
        left(c.last_name, 3) as first_three_letters
    from
        customers as c
    left join
        emails as e
    on
        c.id = e.customer
        and e.current = true
    where
        e.email in ( -- This subquery is unavoidable.
            select
                email
            from
                blacklist
        )
)
select
    first_three_letters, -- We use this twice, so it's calculated in the CTE to prevent repeating code (see below).
    count(*)
from
    the_cte
group by
    first_three_letters
```

## Staying DRY
* If you need to use the result of a particular function (e.g., `dateadd(day, 14, date_column)`) multiple times in a query, consider doing the calculation in a CTE before referencing it in your main query.
* If you need to use the result of a particular subquery in multiple queries, consider creating a temporary table containing the results.

## Referencing Columns
* In `group by` and `order by` clauses, reference the actual names of selected columns (e.g., `group by customer_id`) rather than their position (e.g., `group by 1`). Grouping/ordering by position is very brittle and can fail silently.
* When joining two tables, always specify which table a column comes from, even when it's only present in one table. For example `customers.id` or `c.id`, not just `id`.
* In general, it's preferable to list individual columns to be selected rather than using `select *`. This is especially true for the final select list in a query that is used by an application (e.g., a Python script), because applications could break in unexpected ways if the number, contents, or order of the columns changes.


## Miscellaneous
* Use lower case for SQL keywords.
* `cast()` is preferable to `::` for compatibility with other flavors of SQL.
* For performance reasons, use `like` or `ilike` instead of `~` unless `~` is needed.
* For compatibility, `coalesce()` is preferable to `nvl()`.
* When aliasing, include the `as` for clarity, even though it's not necessary.
* Use `between` (inclusive) where possible instead of combining multiple statements with `and`.
* Rather than using multiple `or` clauses, use `in ()`.
* Write booleans using all lowercase (i.e. `true`|`false`), rather than mixed case, strings (i.e. `'true'`), or integer equivalents (i.e. `0`|`1`).
