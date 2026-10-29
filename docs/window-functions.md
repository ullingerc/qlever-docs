# Window Functions

!!! info "History"
    - Added by [qlever#3848](https://github.com/ad-freiburg/qlever/pull/3848)

In SQL, **window functions** compute a value for each row from other rows of
the same *window*, that is, the rows with the same values in the `PARTITION
BY` columns, in the order given by the `ORDER BY` clause. For example,
`LEAD(?d) OVER (PARTITION BY ?e ORDER BY ?d)` returns, for each row, the value
of `?d` in the next row with the same `?e`. Unlike `GROUP BY`, a window
function keeps all rows. SPARQL has no window functions.

QLever provides a `SERVICE` that adds a column with **row numbers** to the
result of a sorted subquery. Joining such results with each other on the row
number (and the partition variables) yields the equivalent of `LAG`, `LEAD`,
`FIRST_VALUE`, `LAST_VALUE`, `ROW_NUMBER`, and top-k per group, as shown in the
[examples](#sql-window-functions-by-example) below. Because the result is
sorted by the row number, these joins are particularly efficient. See the
[Wikidata example](#example-the-most-famous-people-of-each-country) for
a query that is practically impossible to evaluate without this feature.

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
SELECT ?n ?e ?d WHERE {
  SERVICE rn: {
    [] rn:bind ?n ;
       rn:offset 0 .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
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

## SQL window functions by example

The examples in this section use the following data, where `?e` is an entity
and `?d` is one of its values:

```turtle
<a> <value> 1, 2, 4 .
<b> <value> 3, 5 .
<c> <value> 6 .
```

The query from the [previous section](#row-numbers) numbers the rows as
follows:

| `?n` | `?e` | `?d` |
|--|--|--|
| 0 | `<a>` | 1 |
| 1 | `<a>` | 2 |
| 2 | `<a>` | 4 |
| 3 | `<b>` | 3 |
| 4 | `<b>` | 5 |
| 5 | `<c>` | 6 |

The row numbers are not restarted for each partition. Instead, the partition
variables (here `?e`) are the first variables of the `ORDER BY` clause, such
that the rows of each partition are consecutive. The window functions are then
expressed by joining two row number services with **different offsets** on the
row number variable **and** the partition variables. The following table gives
an overview; the details are explained in the subsequent sections.

| SQL | QLever |
|--|--|
| `LAG(?d, k) OVER (PARTITION BY ?e ORDER BY ?d)` | `OPTIONAL` join with a second service with `offset` k |
| `LEAD(?d, k) OVER (PARTITION BY ?e ORDER BY ?d)` | `offset` k on the outer service, `OPTIONAL` join with a second service |
| `LAST_VALUE`, argmax per group | `MINUS` of the rows that have a successor |
| `FIRST_VALUE`, argmin per group | `MINUS` of the rows that have a predecessor |
| Top-k per group | `MINUS` of the rows that have a predecessor k rows before them |
| `ROW_NUMBER() OVER (PARTITION BY ?e ORDER BY ?d)` | Subtract the smallest row number of the partition |

### `LAG`: the previous row

The following query computes, for each row, the value of the previous row of
the same entity. This is the equivalent of `LAG(?d) OVER (PARTITION BY ?e
ORDER BY ?d)` in SQL.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d ?previous WHERE {
  # All rows, numbered from 0.
  SERVICE rn: {
    [] rn:bind ?n .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
  }
  # All rows, numbered from 1, such that row `i` gets the number of row `i + 1`
  # above. Since `?e` is also a join variable, only rows of the same entity
  # are joined.
  OPTIONAL {
    SERVICE rn: {
      [] rn:bind ?n ;
         rn:offset 1 .
      { SELECT ?e ?previous WHERE { ?e <value> ?previous } ORDER BY ?e ?previous }
    }
  }
}
```

| `?e` | `?d` | `?previous` |
|--|--|--|
| `<a>` | 1 | |
| `<a>` | 2 | 1 |
| `<a>` | 4 | 2 |
| `<b>` | 3 | |
| `<b>` | 5 | 3 |
| `<c>` | 6 | |

The `OPTIONAL` keeps the first row of each entity, which has no predecessor. To
get the row `k` rows before the current row, that is, `LAG(?d, k)`, use
`rn:offset k` in the second service. A default value like in `LAG(?d, 1, 0)`
can be added with `COALESCE(?previous, 0)`.

The second subquery selects the same variables as the first one, but under
different names, such that only `?n` and `?e` are shared between the two
services. See the [performance hints](#performance) for why the subqueries
should otherwise be identical.

### `LEAD`: the next row

For the next row instead of the previous one, the offset is moved to the
**outer** service. This is the equivalent of `LEAD(?d) OVER (PARTITION BY ?e
ORDER BY ?d)` in SQL.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d ?next WHERE {
  # All rows, numbered from 1.
  SERVICE rn: {
    [] rn:bind ?n ;
       rn:offset 1 .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
  }
  # All rows, numbered from 0, such that row `i + 1` gets the number of row
  # `i` above.
  OPTIONAL {
    SERVICE rn: {
      [] rn:bind ?n .
      { SELECT ?e ?next WHERE { ?e <value> ?next } ORDER BY ?e ?next }
    }
  }
}
```

| `?e` | `?d` | `?next` |
|--|--|--|
| `<a>` | 1 | 2 |
| `<a>` | 2 | 4 |
| `<a>` | 4 | |
| `<b>` | 3 | 5 |
| `<b>` | 5 | |
| `<c>` | 6 | |

As a rule of thumb, the offset is the distance between the two rows and it is
put on the service whose row comes **first** in the order. Since the offset
cannot be negative, `LEAD` and `LAG` differ in which service gets the offset.

### `LAST_VALUE` and argmax: the last row of each partition

The last row of each partition is the row without a successor in the same
partition. Unlike the common approach with `GROUP BY ?e` and `MAX(?d)`, this
yields all variables of the row (the subqueries may select further variables)
without evaluating the graph pattern a second time, and exactly one row per
entity, even if there are ties.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d WHERE {
  # All rows, numbered from 1.
  SERVICE rn: {
    [] rn:bind ?n ;
       rn:offset 1 .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
  }
  # Remove the rows that have a successor with the same entity.
  MINUS {
    SERVICE rn: {
      [] rn:bind ?n .
      { SELECT ?e ?next WHERE { ?e <value> ?next } ORDER BY ?e ?next }
    }
  }
}
```

| `?e` | `?d` |
|--|--|
| `<a>` | 4 |
| `<b>` | 5 |
| `<c>` | 6 |

For the argmin, that is, the first row of each partition, put the offset on the
service inside the `MINUS` instead, or sort in descending order with `ORDER BY
?e DESC(?d)`.

### `FIRST_VALUE`: the first row of each partition for every row

To attach a value of the first (or last) row of each partition to every row,
like `FIRST_VALUE(?d) OVER (PARTITION BY ?e ORDER BY ?d)` in SQL, join the
first rows with all rows.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d ?first WHERE {
  ?e <value> ?d .
  # The first row of each entity, that is, the rows without a predecessor with
  # the same entity.
  {
    SELECT ?e ?first WHERE {
      SERVICE rn: {
        [] rn:bind ?n .
        { SELECT ?e ?first WHERE { ?e <value> ?first } ORDER BY ?e ?first }
      }
      MINUS {
        SERVICE rn: {
          [] rn:bind ?n ;
             rn:offset 1 .
          { SELECT ?e ?previous WHERE { ?e <value> ?previous } ORDER BY ?e ?previous }
        }
      }
    }
  }
}
```

| `?e` | `?d` | `?first` |
|--|--|--|
| `<a>` | 1 | 1 |
| `<a>` | 2 | 1 |
| `<a>` | 4 | 1 |
| `<b>` | 3 | 3 |
| `<b>` | 5 | 3 |
| `<c>` | 6 | 6 |

### Top-k per group

A row is among the first `k` rows of its partition if and only if the row `k`
rows before it belongs to a different partition (or does not exist). The
following query computes the two largest values of each entity.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d WHERE {
  # All rows, sorted by descending value per entity, numbered from 0.
  SERVICE rn: {
    [] rn:bind ?n .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e DESC(?d) }
  }
  # Remove the rows that have a row of the same entity two rows before them.
  MINUS {
    SERVICE rn: {
      [] rn:bind ?n ;
         rn:offset 2 .
      { SELECT ?e ?other WHERE { ?e <value> ?other } ORDER BY ?e DESC(?other) }
    }
  }
}
```

| `?e` | `?d` |
|--|--|
| `<a>` | 4 |
| `<a>` | 2 |
| `<b>` | 5 |
| `<b>` | 3 |
| `<c>` | 6 |

### `ROW_NUMBER` per partition

The row numbers of the service are global, but the rows of each partition are
consecutive. The row number within the partition is therefore the global row
number minus the smallest row number of the partition. This is the equivalent
of `ROW_NUMBER() OVER (PARTITION BY ?e ORDER BY ?d)` in SQL.

```sparql
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?e ?d ?rank WHERE {
  # All rows, numbered from 0.
  SERVICE rn: {
    [] rn:bind ?n .
    { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
  }
  # The smallest row number of each entity.
  {
    SELECT ?e (MIN(?n) AS ?first) WHERE {
      SERVICE rn: {
        [] rn:bind ?n .
        { SELECT ?e ?d WHERE { ?e <value> ?d } ORDER BY ?e ?d }
      }
    }
    GROUP BY ?e
  }
  BIND(?n - ?first + 1 AS ?rank)
}
```

| `?e` | `?d` | `?rank` |
|--|--|--|
| `<a>` | 1 | 1 |
| `<a>` | 2 | 2 |
| `<a>` | 4 | 3 |
| `<b>` | 3 | 1 |
| `<b>` | 5 | 2 |
| `<c>` | 6 | 1 |

## Example: the most famous people of each country

The following query computes, for each sovereign state, the three citizens
with the most *sitelinks* on Wikidata, that is, the number of articles about
the person in the different language editions of Wikipedia and other Wikimedia
projects. In SQL, this would be a filter on `ROW_NUMBER() OVER (PARTITION BY
country ORDER BY sitelinks DESC) <= 3`. The query uses the [top-k
pattern](#top-k-per-group) from above.

```sparql
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX wikibase: <http://wikiba.se/ontology#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rn: <https://qlever.cs.uni-freiburg.de/rowNumber/>
SELECT ?country ?countryLabel ?person ?personLabel ?sitelinks WHERE {
  # All citizens of all sovereign states, sorted by country and by descending
  # number of sitelinks, numbered from 0.
  SERVICE rn: {
    [] rn:bind ?n .
    {
      SELECT ?country ?person ?sitelinks WHERE {
        ?country wdt:P31 wd:Q3624078 .
        ?person wdt:P27 ?country ; wdt:P31 wd:Q5 ; wikibase:sitelinks ?sitelinks .
      }
      ORDER BY ?country DESC(?sitelinks) ?person
    }
  }
  # Remove the people that have a person of the same country three rows before
  # them. The subquery is the same as above with renamed variables.
  MINUS {
    SERVICE rn: {
      [] rn:bind ?n ;
         rn:offset 3 .
      {
        SELECT ?country ?other ?otherSitelinks WHERE {
          ?country wdt:P31 wd:Q3624078 .
          ?other wdt:P27 ?country ; wdt:P31 wd:Q5 ; wikibase:sitelinks ?otherSitelinks .
        }
        ORDER BY ?country DESC(?otherSitelinks) ?other
      }
    }
  }
  OPTIONAL { ?country rdfs:label ?countryLabel . FILTER(LANG(?countryLabel) = "en") }
  OPTIONAL { ?person rdfs:label ?personLabel . FILTER(LANG(?personLabel) = "en") }
}
ORDER BY ?countryLabel DESC(?sitelinks)
```

An excerpt of the result:

| `?countryLabel` | `?personLabel` | `?sitelinks` |
|--|--|--|
| Germany | Albert Einstein | 322 |
| Germany | Angela Merkel | 220 |
| Germany | Benedict XVI | 200 |
| India | Mahatma Gandhi | 287 |
| India | Mother Teresa | 200 |
| India | Jawaharlal Nehru | 193 |
| Japan | Matsuo Bashō | 202 |
| Japan | Naruhito | 189 |
| Japan | Katsushika Hokusai | 177 |

Wikidata contains about 5.3 million citizens of 200 sovereign states, with up
to about 690 thousand citizens per state. The query above sorts these rows once
(the second service takes the sorted result from the cache) and then computes
the `MINUS` in a single pass over both sorted results, without sorting again.
On an index with the relevant part of Wikidata, it takes about 1.5 seconds.

??? note "The same query in standard SPARQL"

    Without row numbers, the usual way to compute the top-k per group is to
    count, for each person, the people of the same country with at least as many
    sitelinks:

    ```sparql
    PREFIX wd: <http://www.wikidata.org/entity/>
    PREFIX wdt: <http://www.wikidata.org/prop/direct/>
    PREFIX wikibase: <http://wikiba.se/ontology#>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    SELECT ?country ?countryLabel ?person ?personLabel ?sitelinks WHERE {
      {
        # For each person, count the people of the same country with at least as
        # many sitelinks (including the person itself).
        SELECT ?country ?person ?sitelinks WHERE {
          ?country wdt:P31 wd:Q3624078 .
          ?person wdt:P27 ?country ; wdt:P31 wd:Q5 ; wikibase:sitelinks ?sitelinks .
          ?other wdt:P27 ?country ; wdt:P31 wd:Q5 ; wikibase:sitelinks ?otherSitelinks .
          FILTER(?otherSitelinks >= ?sitelinks)
        }
        GROUP BY ?country ?person ?sitelinks
        HAVING (COUNT(*) <= 3)
      }
      OPTIONAL { ?country rdfs:label ?countryLabel . FILTER(LANG(?countryLabel) = "en") }
      OPTIONAL { ?person rdfs:label ?personLabel . FILTER(LANG(?personLabel) = "en") }
    }
    ORDER BY ?countryLabel DESC(?sitelinks)
    ```

    This query joins all pairs of citizens of the same country, which are about
    1.2 trillion pairs. On the same index as above, it fails after about 20
    seconds because it exceeds a memory limit of 20 GB, and no amount of memory
    would make it practical. For the top-1 per group, `GROUP BY` with `MAX` is a cheaper
    alternative, but it requires joining the graph pattern a second time and
    returns several rows per group in case of ties.

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
   [example](#row_number-per-partition) above for how to compute row numbers
   per partition.
2. `RANK`, `DENSE_RANK`, `NTILE`, and aggregates over window frames (for
   example, running sums with `SUM(...) OVER (... ROWS UNBOUNDED PRECEDING)`)
   are not directly supported. Frames of a fixed size, like a moving average
   over the previous three rows, can be computed with one `LAG` join per row
   of the frame.
3. The `offset` must not be negative. Use the other service for the offset
   instead, as explained for [`LEAD`](#lead-the-next-row).
