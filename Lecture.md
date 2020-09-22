# Lecture

Sept 22, 2020 -- Spatial Joins and Indexes

Spatial Joins are a powerful feature of PostGIS. They're made much more so because of spatial indexes.

## Data

Let's take a look at SEPTA's GTFS data:
* Septa data feed: http://www3.septa.org/developer/
* Easily download from the repo: `https://raw.githubusercontent.com/MUSA-509/week-4-spatial-joins-and-indexes/master/data/septa_bus_stops.txt`

```SQL
UPDATE andyepenn.septa_bus_stops
SET the_geom = ST_SetSRID(ST_MakePoint(stop_lon, stop_lat), 4326)
```

Filling `the_geom` with a geometry. Carto has [database triggers](https://www.postgresql.org/docs/12/sql-createtrigger.html).

## Outline

1. Spatial Joins
  a. Walk through an example explicitly (looping through all of the matches)
  b. `ST_DWithin`, `ST_Intersects`
  c. INNER JOIN vs LEFT JOIN
  d. LATERAL keyword example
2. Spatial Indexes
  a. Theory / build intuition
  b. Carto builds them automatically, has triggers for table updates, etc. All to improve performance of rendering of data and fetching of data
  c. Query to find the indexes
  d. Index syntax -- no need in carto generally, but once we have our own databases we'll need to manage them more
3. UPDATE statements
  a. If you have tables without explicit geometries

### JOIN Use cases

* Given a point, find the State, County, Zip code, census tract, etc. etc. from a polygon dataset. E.g., "What zip code is this user from?"
* Given a point, get the population density, average retail spend, biodiversity measures, last canopy cover, etc. etc.
* Given a set of points, find all boundaries they intersect with and GROUP BY to give counts of number of visits or other aggregate statistics
* What census tracts are within another boundary?

## Equivalent syntax

```SQL
SELECT s.*
  FROM safegraph_neigbhorhoods as s
  JOIN cbgs_boundaries as c
    ON s.area = c.geoid
```

```SQL
SELECT s.*
  FROM safegraph_neigbhorhoods as s,
       cbgs_boundaries as c
    WHERE s.area = c.geoid
```

## Count bus stations in census block groups

```SQL
SELECT
  c.geom, count(ind.*) AS num_stations
FROM andyepenn.philadelphia_cbgs_w_population AS c
LEFT JOIN andyepenn.septa_bus_stops AS ind
ON ST_Contains(c.the_geom, ind.the_geom)
GROUP BY 1
```

## Other types of JOINs

**(INNER) JOIN**

This is the JOIN that we've been doing by default. It drops any rows in either table where there is not a match on the join condition.

Show JOIN results with holes in the map where points were not present.

```SQL
SELECT
  c.geom,
  c.geoid,
  coalesce(count(bus.*), 0) AS num_stations
FROM census_block_groups AS c
JOIN septa_bus_stops AS bus 
  ON ST_Contains(c.geom, ind.geom)
GROUP BY 1, 2
```

**LEFT JOIN**

Used when you want to keep all entries from the _left_ table. Joined columns with no corresponding entry in the left table will be filled with null values. Equivalent to `RIGHT JOIN` if tables names switched places.

Great for joining polygons to points when you might not always have a point intersect the polygon. Non-overlapping polygons will get a null value when counting overlapping points. This can be turned into zeros with the `coalesce` function.

Note: LEFT JOINs lack some of the performance benefits of an INNER JOIN.


Count Indego Bike stations by census block group

```SQL
SELECT
  c.geom,
  c.geoid,
  coalesce(count(bus.*), 0) as num_stations
FROM census_block_groups AS c
LEFT JOIN septa_bus_stops AS bus
  ON ST_Contains(c.geom, ind.geom)
GROUP BY 1, 2
```

**OUTER JOIN**

Used when you want to keep rows from each dataset. E.g., show all customers from a customers table and all purchases from a products table and show all things bought or not.

AFAIK, doesn't work with spatial operators.


**LATERAL JOIN**

A special PostgreSQL keywork `LATERAL` lets your subquery share information with the outer query. It can be powerful depending on the application.

Read more here: [Lateral Join blog post](https://carto.com/blog/lateral-joins/)



## Spatial Indexes

Spatial indexes are database indexes built on the bounding boxes of geomeries in a dataset. Bounding boxes have a small memory footprint (four vertices) so are good for building indexes on.

**What is an index?**

A separate database object that allows for fast and efficient lookup of values. Different indexes exist for different purposes. Some are great for range searches, others for geographic intersects.

**Why are they great?**

They can speed up database query performance by many orders of magnitude by letting the database to use the index in a way to avoid sequentially reading every row of the dataset, an operation that slows with an increasing size of the dataset. Indexes come with downsides (e.g., they can be large) and can be complicated to manage. They also do not work for all queries, can have unexpected behavior (e.g., not be used).

Benefits:

* Indexes can also benefit UPDATE and DELETE commands with search conditions
* Indexes can moreover be used in join searches. Thus, an index defined on a column that is part of a join condition can also significantly speed up queries with joins.

**Types of Indexes**

* B-tree:
  * best for equality and range queries (e.g., `start_hour < 10 and start_hour > 8`) that use <,<=, =, =, >, IS (NOT) NULL, BETWEEN, IN
  * Can be used with string matching using the `~` with LIKE for queries like `col ~ 'and%'` but not `col ~ '%ndy'`.
* Hash
  * Only used for simple equality
* GiST
  * General indexing framework that can be adapted to many situations
  * Great for indexing geometries

### Index Experiment

Let's DROP (Postgres notation for delete) indexes on tables in a Spatial Join to compare the performance before and after.

**Get indexes on a table**

```SQL
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'andyepenn'
AND tablename IN ('philadelphia_cbgs_w_population', 'septa_bus_stops')
-- AND indexname like '%the_geom_idx'
ORDER BY tablename, indexname
```


Drop the ones that reference the `the_geom` columns.

```SQL
DROP INDEX philadelphia_cbgs_w_population_the_geom_idx;
DROP INDEX stops_the_geom_idx;
```


Now run the JOIN query:

```SQL
SELECT c.the_geom, c.geoid, coalesce(count(ind.*), 0) AS num_stations
FROM andyepenn.philadelphia_cbgs_w_population AS c
JOIN andyepenn.septa_bus_stops AS bus 
ON ST_Intersects(c.the_geom, bus.the_geom)
GROUP BY 1, 2
ORDER BY geoid
```

TIMEOUT?? Why did that happen?

Let's run it in [a tool that won't timeout](https://cartodb.github.io/customer_success/batch/).

* Time it without the index
* Add back the indexes
  ```SQL
  CREATE INDEX philadelphia_cbgs_w_population_the_geom_idx ON andyepenn.philadelphia_cbgs_w_population USING gist (the_geom);
  CREATE INDEX stops_the_geom_idx ON andyepenn.septa_bus_stops USING gist (the_geom);
  ```
* Time it again. Was it faster?


## How would you index the first lab question from last week?


```SQL
SELECT c.*, p.raw_stop_counts, p.raw_device_counts, p.distance_from_home
FROM andyepenn.philadelphia_cbgs_w_population as c
JOIN andyepenn.neighborhood_patterns_philadelphia as p
ON p.area = c.geoid
```
