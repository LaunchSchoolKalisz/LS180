# Subqueries

## What are subqueries

A **subquery** is a `SELECT` clause that is nested within _another_ `SELECT` clause. Basically, we use the nested subquery to generate a result set that can then be used in a condition to filter the records generated by the outer select query.

We use a special set of conditional operators known as **subquery expressions** to compare the results of the nested subquery with the records of the outer `SELECT` query. The most common ones are `IN`, `NOT IN`, `ANY`, `SOME`, and `ALL`.

Often, subqueries generate the same results as a `JOIN`. A general use-case for utilizing subqueries is one in which data is being returned from one table that relies on some kind of conditional expression that _utilizes_ data from another table, but no data from that second table actually needs to be included in the result set. When data from two or more tables needs to be represented in the result set, a join is probably preferred.

With large databases, questions of performance optimization can arise. We can compare the potential performance of using either a `JOIN` or a subquery by utilizing the key words `EXPLAIN` or `EXPLAIN ANALYZE`.

## General Syntax

```sql
SELECT col_name_a FROM table_name
  WHERE SUBQUERY_EXPRESSION
  (SELECT col_name_b FROM table_name [WHERE (expression)]);
```

When using subqueries, we first write the _outer_ select statement. This is followed by a `WHERE` clause that utilizes one of the **subquery expressions**. Finally, we include the nested subquery in the conditional expression by putting it in parenthesis.

It's important to remember that when utilizing subqueries and subquery expressions, the nested subquery (the one inside parenthesis) must return _only values that exist in a single column_. Otherwise, we need to use a **row constructor**.

Something else to keep in mind when utilizing subquery expressions is how `NULL` values can affect the result. Recall that `NULL` when used with a conditional operator will always return `NULL`. Therefore, if the left-hand expression (the `WHERE` clause) yields `NULL`, if the subquery result set is empty, or at least one of the records in the subquery result set yields `NULL`, the result of the subquery expression will also be `NULL`, rather than `true` or `false`.

## Subquery Expression Examples

For the following discussion of subqueries, we'll utilize an example database that is an inventory for various items being stored in storage units. We have one table representing the storage units, and another table representing the items being stored. We'll use a foreign key in the items table to specify which storage unit any given item is stored in.

`items`

| id | description | storage_id | date_packed | weight | value |
|----|-------------|------------|-------------|--------|-------|
| 1  | box of dishes | 2 | 2021-03-13 | 25.2 | 100.00 |
| 2  | dinning room chair | 2 | 2021-03-13 | 18.4 | 150.00 |
| 3  | photo album | 3 | 2019-11-04 | 4.7 | 4.00 |
| 4  | wedding dress | 2 | 2016-09-22 | 8.1 | 2000.00 |
| 5  | skis | 1 | 2018-02-15 | 9.3 | 350.00 |

`storage_units`

| id | unit_number | cost_per_month |
|----|-------------|----------------|
| 1  | B307        | 200.00
| 2  | B515        | 275.00
| 3  | C203        | 150.00

### IN

`IN` utilizes the conditional expression in the `WHERE` clause and compares it to each row in the result set returned by the given subquery. `IN` will return `true` if _any_ row within the subquery result meets the condition. Otherwise, it will return `false`.

```sql
SELECT unit_number FROM storage_units WHERE id IN
  (SELECT storage_id FROM items WHERE weight < 10);
/*
 unit_number
-------------
 C203
 B515
 B307
(3 rows) */
```

First, the above query will execute the parenthetical subquery. This returns a single column of data, the storage unit id number for those items that have a weight less than 10 pounds. In this case, that is all the storage units. Next, we test the outer `SELECT` query against these rows: if the id number of the storage unit being checked is represented in the set returned by the subquery at all, the `unit_number` of that storage unit is added to the final result set. Since all the storage units have at least one item that is less than 10 pounds, all the storage units are selected.

```sql
SELECT unit_number FROM storage_units WHERE id IN
  (SELECT storage_id FROM items WHERE age(date_packed) > '3 years');
/* 
 unit_number
-------------
 B515
 B307
(2 rows) */
```

Here we are returning the unit number of those storage units which have items that were packed and stored over 3 years ago. This only returns 2 rows of data. This is because the nested subquery returns only the id for two units (`1` and `2`). The `id` value for storage unit `C203` is not "in" our nested subquery's result set, and therefore it is not selected by the outer query.

### NOT IN

`NOT IN` utilizes the conditional expression in the `WHERE` clause and compares it to each row in the result set returned by the given subquery. `NOT IN` will return `true` if a row that meets the given condition is _not_ found in the subquery's results, `false` otherwise. It is essentially the opposite of `IN`.

```sql
SELECT unit_number FROM storage_units WHERE id NOT IN
  (SELECT storage_id FROM items WHERE age(date_packed) > '3 years');
/*
 unit_number
-------------
 C203
(1 row) */
```

In the example above, we use the parenthetical subquery to return a list of `storage_id` values from `items` from those rows who's `date_packed` date indicates a time more than 3 years ago. Then those id values are compared to the primary key column of `storage_units`. We select the `unit_number` value from `storage_units` who's id value is _not_ represented in the list returned by the subquery, in this case, unit number `C203`. Looking at the items in this unit, we can see that there is a single item which was packed on `2019-11-04`, less than 3 years ago.

### ANY and SOME

Both `ANY` and `SOME` work the same way. Unlike `IN` and `NOT IN` they require the use of a comparison operator (i.e. `>`, `<` `=`). It then takes the expression on the left of that operator in the `WHERE` clause and compares it with each record returned by the given parenthetical subquery. If the result of this evaluation returns true for _any_ of the records returned by the subquery, the whole `WHERE` clause returns `true`.

```sql
SELECT unit_number FROM storage_units WHERE cost_per_month > ANY
  (SELECT sum(value) FROM items GROUP BY storage_id);
/* 
 unit_number
-------------
 B515
 C203
 B307
(3 rows) */
```

Above, we return all three storage units. This is because `ANY` will return `true` if _any_ true result is ever returned by the conditional operator. In this case, both `B515` and `C203` have items whose value is less than the unit's `cost_per_month`. This causes `ANY` to return true for _all_ the units, even `B307` which does not meet the condition.

When we use `=` with `ANY`, this is basically the same thing as `NOT IN`.

### ALL

Like `ANY`, `ALL` is also used with a comparison operator. It takes the expression on the left of that operator in the `WHERE` clause and compares it with each record returned by the given parenthetical subquery. If the reuslt of this evaluation returns true for _all_ the records returned by the subquery, the whole `WHERE` clause returns `true`, otherwise, it will return `false` and no records will be selected.

```sql
SELECT unit_number FROM storage_units WHERE cost_per_month > ALL
  (SELECT sum(value) FROM items GROUP BY storage_id);
/*
 unit_number
-------------
(0 rows) */
```

Above, we return _none_ of the storage unit records, because not all of them have met the condition specified in the `WHERE` clause. Although `C203` does meet this condition, the other units do not and so no units are selected (`WHERE` returns `false`).

When we use the `<>` or `!=` operator with `ALL`, this is basically the same thing as using `NOT IN`.

### EXISTS

`EXISTS` checks to see if any records at all are returned by the given parenthetical subquery. If at least one row is returned, the result of `EXISTS` will be `true`. If no rows at all are returned, the result will be `false`.

Typically, we use an arbitrary expression with `EXISTS` (such as `1`) rather than an identifier like we might be used to seeing with `SELECT`.

For example, what if we are looking for our winter coats in storage, and we want to see if they are in a storage unit. We can do so with `EXISTS`:

```sql
SELECT 'yes' WHERE EXISTS
  (SELECT id FROM items WHERE description = 'winter coats');
/*
 ?column?
----------
(0 rows) */
```

Above, we do not have an item with the description `'winter coats'`, and so it is not considered to "exist" within the database. `EXISTS` returns `false`, the arbitrary expression is not selected, and `NULL` is returned.

```sql
SELECT 'yes' WHERE EXISTS
  (SELECT id FROM items WHERE description = 'skis');
/*
 ?column?
----------
 yes
(1 row) */
```

Above, we _do_ have an item in the database with the description `'skis'`, and so it _is_ considered to "exist". At least one record (the `id` number for the item with the description `'skis'`) is returned by the subquery and so ``EXISTS` returns `true` and our arbitrary expression `'yes'` is selected and returned.