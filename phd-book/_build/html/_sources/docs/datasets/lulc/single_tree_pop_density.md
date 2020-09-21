# Single Tree Population Density

The aim is produce the map to show the tree density in the residential area. Inside the residential areas, tree distrbution is higher so that they usually consider as single tree. Such a map can be supplementary to show which parts of the residential areas have more trees.


## Producing the Grid

To count the number of the trees inside the 1 km2 grid, it is required to produce the grid. For this purpose the following function was introduced to PostGIS:


```
CREATE OR REPLACE FUNCTION ST_CreateFishnet(
        nrow integer, ncol integer,
        xsize float8, ysize float8,
        x0 float8 DEFAULT 0, y0 float8 DEFAULT 0,
        OUT "row" integer, OUT col integer,
        OUT geom geometry)
    RETURNS SETOF record AS
$$
SELECT i + 1 AS row, j + 1 AS col, ST_Translate(cell, j * $3 + $5, i * $4 + $6) AS geom
FROM generate_series(0, $1 - 1) AS i,
     generate_series(0, $2 - 1) AS j,
(
SELECT ('POLYGON((0 0, 0 '||$4||', '||$3||' '||$4||', '||$3||' 0,0 0))')::geometry AS cell
) AS foo;
$$ LANGUAGE sql IMMUTABLE STRICT;
```

In the next step, the extension of the Switzerland was found as follow:

```sql
create table swiss_bbox as
select ST_Envelope(geom) from swiss_border;
```

Then, the extension of the produced area was used to create a grid as follow:

```sql
create table swiss_grid as
SELECT *
FROM ST_CreateFishnet(221, 350, 1000, 1000, 485373.900,75186.047) AS cells;

select row, col,gid, ST_SetSRID(swiss_grid.geom, 21781) as geom into swiss_grid_proj from swiss_grid
```

In the final step the number of the trees was count inside the each polygon:

```sql
-- Counting the tree in each polygon
create view lulc.single_tree_count as
SELECT grid.gid, count(single_tree_shrubbery.wkb_geometry) AS totale
FROM swiss_grid_proj grid
LEFT JOIN lulc.single_tree_shrubbery
    ON st_contains(grid.geom,single_tree_shrubbery.wkb_geometry)
GROUP BY grid.gid;

-- joining the produced table to grid
select g.id, g.row,g.col,stc.totale,g.geom
into lulc.single_tree_count_grid
from swiss_grid_proj g
join single_tree_count stc on g.gid = stc.gid;

ALTER TABLE single_tree_count_grid
ADD PRIMARY KEY (id);
```

```{note}
In total, there are **4244828** single tree which mapped.
```

```bash
gdal_rasterize -l lulc.single_tree_count_grid -a totale -ts 1000.0 1000.0 -a_nodata -9999 -te 485422.117343434 75159.3856499034 834069.939666441 298248.830831526 -ot Float32 -of GTiff "PG:dbname='pochasdb' host=127.0.0.2 port=5432 user='gis' password='gis'" single_tree_density.tif
```



```{image} /pics/single_tree.PNG
:alt: fishy
:class: bg-primary
:width: 500px
:align: center
```

