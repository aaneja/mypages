# Plan constraints grammar

Pre read :  [This blog post](https://prestodb.io/blog/2024/03/21/elevating-presto-query-optimization/) and [overview doc](https://prestodb.io/wp-content/uploads/Search-Space-Improvements-Plan-Constraints.pdf) on plan constraints

## Grammar

Grammar for plan constraints in Presto

```
planConstraintString : /*! planConstraint [, ...] */

planConstraint : joinConstraint
| cardinalityConstraint

joinConstraint : joinType (joinNode) [distributionType]

cardinalityConstraint : CARD (joinNode cardinality)

distributionType : [P]
   | [R]

joinType : JOIN	(defaults to inner join)
| IJ
| LOJ
| ROJ

cardinality : integer constant (positive)

joinNode : (relationName relationName [, ...])

| joinConstraint

| (relationName joinNode)

| (joinNode relationName)

| (joinNode joinNode [, ...])

```

The plan constraint string gets parsed according to the grammar above and a list of corresponding constraint objects are created and populated into the session for the optimizer to pick up. These objects then influence the join planner to build a “constrained” join when possible.

## Notes on how constraints will work

1. Constraints are specified in a SQL comment using a start and end marker. Anything between the text `/*!` and `/` is considered as a constraint
2. Only two types of constraints are initially supported -
    1. Inner Join constraints -
        1. `join (a (b c))` - Join the relations a,b,c in the tree shape denoted by the brackets. Use regular rules for determining join distribution (PARTITIONED or REPLICATED)
        2. `join (((a c) [R] b) [P])` - In addition to the join order, you can also specify if the join distribution to use is PARTITIONED `[P]` or REPLICATED `[R]`
        3. If an inner join condition does not exist between nodes a CrossJoin is automatically inferred
    2. Cardinality constraints
        1. `card (c 10)` - Set the output row count estimate of `c` to `10`
        2. `card ((c o) 10)` - When considering a join node of shape `(c o)` set the output row count estimate to `10`
3. Use sub-query, CTE and table aliases to refer to plan nodes that can be constrained, not the base table names

## Examples

### Cardinality constraint on TPCH

```sql
/*! card ((c o) 10) */

SELECT COUNT(*)
FROM nation n,
    customer c,
    orders o
WHERE c.custkey = o.custkey
    AND c.nationkey = n.nationkey
```

- Forces the cardinality of the `c IJ o` join to be considered as 10
- Resulting join order is `n (c o)`

### Join order benchmark q1a

```sql
/*! join ((ct ((t it) mi_idx)) mc) */

SELECT MIN(mc.note) AS production_note,
MIN(t.title) AS movie_title,
MIN(t.production_year) AS movie_year
FROM company_type AS ct,
info_type AS it,
movie_companies AS mc,
movie_info_idx AS mi_idx,
title AS t
WHERE ct.kind = 'production companies'
AND it.info = 'top 250 rank'
AND mc.note NOT LIKE '%(as Metro-Goldwyn-Mayer Pictures)%'
AND (mc.note LIKE '%(co-production)%'
OR mc.note LIKE '%(presents)%')
AND ct.id = mc.company_type_id
AND t.id = mc.movie_id
AND t.id = mi_idx.movie_id
AND mc.movie_id = mi_idx.movie_id
AND it.id = mi_idx.info_type_id;

```

- Forces a Cross Join
- Has a lower latency than the defaut plan

### TPCDS Q1

```sql
/*! join (date_dim store_returns) join (customer store) */

with customer_total_return as
(select sr_customer_sk as ctr_customer_sk
,sr_store_sk as ctr_store_sk
,sum(SR_RETURN_AMT_INC_TAX) as ctr_total_return
from store_returns
,date_dim
where sr_returned_date_sk = d_date_sk
and d_year =1999
group by sr_customer_sk
,sr_store_sk)
select  c_customer_id
from customer_total_return ctr1
,store
,customer
where ctr1.ctr_total_return > (select avg(ctr_total_return)*1.2
from customer_total_return ctr2
where ctr1.ctr_store_sk = ctr2.ctr_store_sk)
and s_store_sk = ctr1.ctr_store_sk
and s_state = 'TN'
and ctr1.ctr_customer_sk = c_customer_sk
order by c_customer_id
limit 100;

```

- Because of how the plan tree is initially built, Presto uses the same SourceLocation for the customer_total_return and ctr2 aliases. Due to this, we cannot build a constraint which refers to these CTE’s directly
- We can however control the join orders of the queries under & above the aggregation using 2 distinct join constraints