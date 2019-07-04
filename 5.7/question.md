index dive
Queries with nested outer joins are executed in the same pipeline manner as queries with inner joins.
More exactly, a variation of the nested-loop join algorithm is exploited. Recall the algorithm by which
the nested-loop join executes a query (see Section 8.2.1.6, “Nested-Loop Join Algorithms”). Suppose
that a join query over 3 tables T1,T2,T3 has this form:之后的暂停掉

### 1.4.1 优化INSERT语句 P1275

chapter8.4
    8.4.1 Optimizing Data Size
        joins(第一段)
    8.4.4   EXPLAIN will not necessarily say Using temporary for derived or materialized temporary tables.    
        
