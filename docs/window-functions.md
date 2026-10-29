# Window Functions

!!! info "History"
    - Added by [qlever#TODO](https://github.com/ad-freiburg/qlever/pull/TODO)

In SQL, **window functions** compute a value for each row from other rows of
the same *window*, that is, the rows with the same values in the `PARTITION
BY` columns, in the order given by the `ORDER BY` clause. For example,
`LAG(population) OVER (PARTITION BY city ORDER BY year)` returns, for each row,
the population of the previous row of the same city. Unlike `GROUP BY`, a
window function keeps all rows. SPARQL has no window functions.

QLever provides a `SERVICE` that adds a column with **row numbers** to the
result of a sorted subquery. Joining such results with each other on the row
number (and the partition variables) yields the equivalent of `LAG`, `LEAD`,
`FIRST_VALUE`, `LAST_VALUE`, `ROW_NUMBER`, and top-k per group, as shown in the
[examples](#sql-window-functions-by-example) below. Because the result is
sorted by the row number, these joins are particularly efficient.

A simple SPARQL function to be used with `BIND` would be unsuitable for this,
because the value of a `BIND` expression may only depend on the bindings of its
own row and not on the position of that row in the result.

## Row numbers

The row number service is invoked with a `SERVICE` to the IRI
`<https://qlever.cs.uni-freiburg.de/rowNumber/>`. Note that this address is not
contacted but only used to activate the feature locally. A row number query has
the following form:

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT * WHERE {
  SERVICE rn: {
    [] rn:bind ?n ;
       rn:offset 1 .
    {
      SELECT ?x ?y WHERE { ... }
      ORDER BY ?x ?y
    }
  }
}
```

The result contains all variables selected by the subquery and additionally the
variable given by `bind`, which is bound to the row numbers `offset, offset + 1,
offset + 2, ...` in the order of the subquery's `ORDER BY` clause.

The following configuration parameters are provided in the `SERVICE` as triples
with arbitrary subject. The predicate must be an IRI of the form `<parameter>`
or `rn:parameter`. The parameter `bind` is mandatory.

| Parameter | Domain | Description |
|--|--|--|
| `bind` | variable | An otherwise unbound variable, to which the row numbers are bound as `xsd:integer` values. |
| `offset` | non-negative integer | The row number of the first row. Defaults to `0`. |

The `SERVICE` must contain exactly one group graph pattern, which consists of a
single subquery with an `ORDER BY` or `INTERNAL SORT BY` clause. The subquery
may use all other features of SPARQL, including `GROUP BY`, `HAVING`, `DISTINCT`,
`VALUES`, and `ORDER BY` expressions like `DESC(YEAR(?date))`. Each variable of
the `ORDER BY` clause must be bound by the subquery, otherwise the query is
rejected with an error (an unbound variable has no effect on the order, so the
row numbers would silently not follow the requested order).

## Example data

The examples on this page use the following data, which contains the
(fictional) population of three cities in different years:

```turtle
@prefix ex: <http://example.org/> .
ex:m1 ex:city ex:Freiburg ; ex:year 2000 ; ex:population 205000 .
ex:m2 ex:city ex:Freiburg ; ex:year 2010 ; ex:population 224000 .
ex:m3 ex:city ex:Freiburg ; ex:year 2020 ; ex:population 231000 .
ex:m4 ex:city ex:Basel ; ex:year 2000 ; ex:population 166000 .
ex:m5 ex:city ex:Basel ; ex:year 2020 ; ex:population 173000 .
ex:m6 ex:city ex:Graz ; ex:year 2010 ; ex:population 262000 .
```

The following query numbers all measurements, sorted by city and year:

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?n ?city ?year ?population WHERE {
  SERVICE rn: {
    [] rn:bind ?n .
    {
      SELECT ?city ?year ?population WHERE {
        ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
      }
      ORDER BY ?city ?year
    }
  }
}
```

| `?n` | `?city` | `?year` | `?population` |
|--|--|--|--|
| 0 | `ex:Basel` | 2000 | 166000 |
| 1 | `ex:Basel` | 2020 | 173000 |
| 2 | `ex:Freiburg` | 2000 | 205000 |
| 3 | `ex:Freiburg` | 2010 | 224000 |
| 4 | `ex:Freiburg` | 2020 | 231000 |
| 5 | `ex:Graz` | 2010 | 262000 |

## SQL window functions by example

The row numbers are not restarted for each partition. Instead, the partition
variables (here `?city`) are the first variables of the `ORDER BY` clause, such
that the rows of each partition are consecutive. The window functions are then
expressed by joining two row number services with **different offsets** on the
row number variable **and** the partition variables. The following table gives
an overview; the details are explained in the subsequent sections.

| SQL | QLever |
|--|--|
| `LAG(?x, k) OVER (PARTITION BY ?p ORDER BY ?o)` | `OPTIONAL` join with a second service with `offset` k |
| `LEAD(?x, k) OVER (PARTITION BY ?p ORDER BY ?o)` | `offset` k on the outer service, `OPTIONAL` join with a second service |
| `LAST_VALUE`, argmax per group | `MINUS` of the rows that have a successor (`LEAD`) |
| `FIRST_VALUE`, argmin per group | `MINUS` of the rows that have a predecessor (`LAG`) |
| `ROW_NUMBER() OVER (PARTITION BY ?p ORDER BY ?o)`, top-k per group | Subtract the smallest row number of the partition |

### `LAG`: the previous row

The following query computes, for each measurement, the population of the
previous measurement of the same city and the growth since then. This is the
equivalent of `LAG(population) OVER (PARTITION BY city ORDER BY year)` in SQL.

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?city ?year ?population ?previous ?growth WHERE {
  SERVICE rn: {
    [] rn:bind ?n .
    {
      SELECT ?city ?year ?population WHERE {
        ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
      }
      ORDER BY ?city ?year
    }
  }
  OPTIONAL {
    SERVICE rn: {
      [] rn:bind ?n ; rn:offset 1 .
      {
        SELECT ?city ?previousYear ?previous WHERE {
          ?m ex:city ?city ; ex:year ?previousYear ; ex:population ?previous .
        }
        ORDER BY ?city ?previousYear
      }
    }
  }
  BIND(?population - ?previous AS ?growth)
}
```

| `?city` | `?year` | `?population` | `?previous` | `?growth` |
|--|--|--|--|--|
| `ex:Basel` | 2000 | 166000 | | |
| `ex:Basel` | 2020 | 173000 | 166000 | 7000 |
| `ex:Freiburg` | 2000 | 205000 | | |
| `ex:Freiburg` | 2010 | 224000 | 205000 | 19000 |
| `ex:Freiburg` | 2020 | 231000 | 224000 | 7000 |
| `ex:Graz` | 2010 | 262000 | | |

The second service numbers the rows starting at 1, so its row `i` has the same
row number as row `i + 1` of the first service. Joining on `?n` and `?city`
thus attaches to each row its predecessor, if the predecessor belongs to the
same city. The `OPTIONAL` keeps the first row of each city, which has no
predecessor. To get the row `k` rows before the current row, that is,
`LAG(population, k)`, use `rn:offset k` in the second service. A default value
like in `LAG(population, 1, 0)` can be added with `COALESCE(?previous, 0)`.

The second subquery selects the same variables as the first one, but under
different names, such that only `?n` and `?city` are shared between the two
services. See the [performance hints](#performance) for why the subqueries
should otherwise be identical.

### `LEAD`: the next row

For the next row instead of the previous one, the offset is moved to the
**outer** service. The following query computes, for each measurement, the year
of the next measurement of the same city. This is the equivalent of
`LEAD(year) OVER (PARTITION BY city ORDER BY year)` in SQL.

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?city ?year ?nextYear WHERE {
  SERVICE rn: {
    [] rn:bind ?n ; rn:offset 1 .
    {
      SELECT ?city ?year WHERE {
        ?m ex:city ?city ; ex:year ?year .
      }
      ORDER BY ?city ?year
    }
  }
  OPTIONAL {
    SERVICE rn: {
      [] rn:bind ?n .
      {
        SELECT ?city ?nextYear WHERE {
          ?m ex:city ?city ; ex:year ?nextYear .
        }
        ORDER BY ?city ?nextYear
      }
    }
  }
}
```

| `?city` | `?year` | `?nextYear` |
|--|--|--|
| `ex:Basel` | 2000 | 2020 |
| `ex:Basel` | 2020 | |
| `ex:Freiburg` | 2000 | 2010 |
| `ex:Freiburg` | 2010 | 2020 |
| `ex:Freiburg` | 2020 | |
| `ex:Graz` | 2010 | |

As a rule of thumb, the offset is the distance between the two rows and it is
put on the service whose row comes **first** in the order. Since the offset
cannot be negative, `LEAD` and `LAG` differ in which service gets the offset.

### `LAST_VALUE` and argmax: the last row of each partition

The last row of each partition is the row without a successor in the same
partition. The following query computes the most recent measurement of each
city, that is, the argmax of the year per city. Unlike the common approach
with `GROUP BY ?city` and `MAX(?year)`, this yields all variables of the row
without joining the pattern a second time, and exactly one row per city, even
if there are ties.

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?city ?year ?population WHERE {
  SERVICE rn: {
    [] rn:bind ?n ; rn:offset 1 .
    {
      SELECT ?city ?year ?population WHERE {
        ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
      }
      ORDER BY ?city ?year
    }
  }
  MINUS {
    SERVICE rn: {
      [] rn:bind ?n .
      {
        SELECT ?city ?nextYear ?nextPopulation WHERE {
          ?m ex:city ?city ; ex:year ?nextYear ; ex:population ?nextPopulation .
        }
        ORDER BY ?city ?nextYear
      }
    }
  }
}
```

| `?city` | `?year` | `?population` |
|--|--|--|
| `ex:Basel` | 2020 | 173000 |
| `ex:Freiburg` | 2020 | 231000 |
| `ex:Graz` | 2010 | 262000 |

For the argmin, that is, the first row of each partition, put the offset on the
service inside the `MINUS` instead, or sort in descending order, for example
with `ORDER BY ?city DESC(?year)`.

### `FIRST_VALUE`: the first row of each partition for every row

To attach a value of the first (or last) row of each partition to every row,
like `FIRST_VALUE(population) OVER (PARTITION BY city ORDER BY year)` in SQL,
join the result of the previous section with all rows. The following query
computes, for each measurement, the population of the first measurement of the
same city.

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?city ?year ?population ?firstPopulation WHERE {
  ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
  {
    SELECT ?city ?firstPopulation WHERE {
      SERVICE rn: {
        [] rn:bind ?n .
        {
          SELECT ?city ?year ?firstPopulation WHERE {
            ?m ex:city ?city ; ex:year ?year ; ex:population ?firstPopulation .
          }
          ORDER BY ?city ?year
        }
      }
      MINUS {
        SERVICE rn: {
          [] rn:bind ?n ; rn:offset 1 .
          {
            SELECT ?city ?prevYear ?prevPopulation WHERE {
              ?m ex:city ?city ; ex:year ?prevYear ; ex:population ?prevPopulation .
            }
            ORDER BY ?city ?prevYear
          }
        }
      }
    }
  }
}
```

| `?city` | `?year` | `?population` | `?firstPopulation` |
|--|--|--|--|
| `ex:Freiburg` | 2000 | 205000 | 205000 |
| `ex:Freiburg` | 2010 | 224000 | 205000 |
| `ex:Freiburg` | 2020 | 231000 | 205000 |
| `ex:Basel` | 2000 | 166000 | 166000 |
| `ex:Basel` | 2020 | 173000 | 166000 |
| `ex:Graz` | 2010 | 262000 | 262000 |

### `ROW_NUMBER` per partition and top-k per group

The row numbers of the service are global, but the rows of each partition are
consecutive. The row number within the partition is therefore the global row
number minus the smallest row number of the partition. The following query
computes the two measurements with the largest population for each city. This
is the equivalent of filtering by `ROW_NUMBER() OVER (PARTITION BY city ORDER
BY population DESC) <= 2` in SQL.

```sparql
PREFIX ex: <http://example.org/>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?city ?year ?population ?rank WHERE {
  SERVICE rn: {
    [] rn:bind ?n .
    {
      SELECT ?city ?year ?population WHERE {
        ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
      }
      ORDER BY ?city DESC(?population)
    }
  }
  {
    SELECT ?city (MIN(?n) AS ?first) WHERE {
      SERVICE rn: {
        [] rn:bind ?n .
        {
          SELECT ?city ?year ?population WHERE {
            ?m ex:city ?city ; ex:year ?year ; ex:population ?population .
          }
          ORDER BY ?city DESC(?population)
        }
      }
    }
    GROUP BY ?city
  }
  BIND(?n - ?first + 1 AS ?rank)
  FILTER(?rank <= 2)
}
```

| `?city` | `?year` | `?population` | `?rank` |
|--|--|--|--|
| `ex:Basel` | 2020 | 173000 | 1 |
| `ex:Basel` | 2000 | 166000 | 2 |
| `ex:Freiburg` | 2020 | 231000 | 1 |
| `ex:Freiburg` | 2010 | 224000 | 2 |
| `ex:Graz` | 2010 | 262000 | 1 |

## Performance

The result of the row number service is sorted by the row number. Since the
row numbers are unique, it is also sorted by the row number followed by any
other variables. The joins on the row number and the partition variables in the
examples above are therefore computed without sorting their inputs again.

Joining two row number services requires two subqueries, which have to be
sorted. If the two subqueries are identical up to the names of their
variables, that is, they have the same graph pattern, the same `ORDER BY`
clause and select the same variables (possibly under different names), QLever
computes and sorts the result only once and takes it from the cache for the
second service. This is why the examples above rename the variables of the
second subquery instead of selecting fewer variables.

`ORDER BY` sorts the values by their semantics as defined by SPARQL.
`INTERNAL SORT BY` instead uses QLever's internal order, which is cheaper and
can sometimes avoid sorting entirely, for example if the subquery is a single
triple pattern. The internal order coincides with the semantic order for many
values, but not for all (for example, negative numbers, or numbers of different
datatypes). It is a good choice if any consistent order suffices, for example
for the partition variables, or if the values are known to be compatible.

If several rows have the same values for all variables of the `ORDER BY`
clause, their order, and thus their row numbers, are arbitrary. For a
deterministic result, add further variables to the `ORDER BY` clause until it
is unique.

## Current limitations

1. The row numbers are not restarted for each partition. See the
   [example](#row_number-per-partition-and-top-k-per-group) above for how to
   compute row numbers per partition.
2. `RANK`, `DENSE_RANK`, `NTILE`, and aggregates over window frames (for
   example, running sums with `SUM(...) OVER (... ROWS UNBOUNDED PRECEDING)`)
   are not directly supported. Frames of a fixed size, like a moving average
   over the previous three rows, can be computed with one `LAG` join per row
   of the frame.
3. The `offset` must not be negative. Use the other service for the offset
   instead, as explained for [`LEAD`](#lead-the-next-row).
