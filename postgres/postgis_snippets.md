- generate one bbox for entire census dataset and add index

```sql
DROP TABLE IF EXISTS census_bb;


CREATE TABLE census_bb AS
SELECT (row_number() OVER ()) AS oid,
       Box2D(ST_Collect(Box2D(geom))) AS geom
FROM census;


ALTER TABLE census_bb
ALTER COLUMN geom TYPE geometry(POLYGON, 4326) USING ST_SetSRID(geom,4326);


DROP INDEX IF EXISTS census_bb_geom_idx;


CREATE INDEX census_bb_geom_idx ON census USING GIST (geom);
```

- generate 1 million points

```sql
DROP TABLE IF EXISTS random_points;


CREATE TABLE random_points AS
WITH new_points AS
    (SELECT ((ST_Dump(ST_GeneratePoints(geom, 1000))).geom) AS geom
     FROM census_bb)
SELECT row_number() OVER () AS oid,
       geom
FROM new_points;


ALTER TABLE random_points
ALTER COLUMN geom TYPE geometry(POINT, 4326) USING ST_SetSRID(geom,4326);


DROP INDEX IF EXISTS random_points_geom_idx;


CREATE INDEX random_points_geom_idx ON random_points USING GIST (geom);
```

- generate distance table

```sql
CREATE TABLE random_points_distance AS
SELECT p.oid,
       p.geom,
       coalesce(c1_geoid10,
                c2_geoid10) geoid10,
               coalesce(c1_distance,
                        c2_distance) distance
FROM random_points p
LEFT OUTER JOIN
    (SELECT census.geoid10 c1_geoid10,
            0 c1_distance,
            census.geom c1_geom
     FROM census) c1 ON ST_Intersects(c1.c1_geom,
                                      p.geom)
LEFT  JOIN LATERAL
    (SELECT ST_DistanceSpheroid(p.geom, c2x.geom, 'SPHEROID["GRS 1980",6378137,298.257222101,
            AUTHORITY["EPSG","7019"]]') AS c2_distance,
            c2x.geoid10 c2_geoid10,
            c2x.geom c2_geom
     FROM random_points p2x,
          census c2x
     WHERE c1_geom IS NULL
         AND p2x.oid = p.oid
         AND ST_DWithin(p.geom, c2x.geom, 0.5)
     ORDER BY c2_distance LIMIT 1) c2 ON TRUE;
```

# reference links

[lateral join alternatives](http://illuminatedcomputing.com/posts/2015/02/lateral_join_alternatives/)

- lateral join
- distinct on
- window functions

[row_number explained](http://www.postgresqltutorial.com/postgresql-row_number/)
