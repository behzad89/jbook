# Data Prepration

the datasets were imported and codes was prepared based on the metadata. in the next steps the following script was used to join metada with tables and get some intial information:


```{admonition} Classes
:class: tip
From swissTLM3D, **9** groups from Bodenbedeckung (ground cover) layer as follow, plus Siedlungsname (Settlement name) which includes information regarding the residential areas were extracted:

- water course
- wetlands
- spare forests
- brushland/scrubland
- glacier
- stagnant waters
- rock
- loose rocks
- dense forest
```
## Ground Cover Layer

```sql
CREATE VIEW lulc.ground_cover_with_name AS
select *,st_area(wkb_geometry) as area
from ground_cover join ground_cover_codes on objektart = code;
```

## Settlement Layer


Since this layer does not include any code and number, so at first step they were determined:

```sql
ALTER TABLE settlement ADD code int default 'settlement'
ALTER TABLE settlement ADD number int default 2;
```

## Agricultural Layer

These two layers comes from Vector25 datasets. **4** layers were selected as agricultures as follow:

|    ObjectVal          |   Description  |  Count            |
| :-------------------- | -------------  |-----------------: |
|   Z_BaumS             |   Baumschule   |  Plantation tree  |
|   Z_GsPist            |   Graspiste    |  Grass            |
|   Z_ObstAn            |   Graspiste    |  Fruit plantation |
|   Z_Reben             |   Reben        |  Wine             | 

The rest belongs to the class called others.

```sql
create table lulc.vector25_farm as
select objectid,objectval,geom
    from vector25 where objectval in ('Z_BaumS','Z_ObstAn','Z_Reben','Z_GsPist');

ALTER TABLE vector25_farm ADD code varchar(50) default 'agricultural area';


ALTER TABLE vector25_farm ADD number int default 18;
ALTER TABLE vector25_farm ADD ID SERIAL PRIMARY KEY;


select distinct objectval,round(((sum(st_area(geom)))/10000)::numeric,2) as area_ha, count(*)
from vector25_farm
group by objectval
order by area_ha;
```

```{note}
:class: dropdown
the results is as follow:


|    ObjectVal          |   Area(ha)     |  Count    |
| :-------------------- | -------------  |---------: |
|   Z_GsPist            |   105.14       |    36     |
|   Z_BaumS             |   1151.55      |    998    |
|   Z_Reben             |   17118.76     |    5331   |
|   Z_ObstAn            |   27930.81     |    37324  |
```

## Others Layer

```sql
create table lulc.vector25_others as
select objectid,objectval,geom
    from vector25 where objectval in ('Z_Uebrig');

ALTER TABLE vector25_others ADD code varchar(50) default 'others';
ALTER TABLE vector25_others ADD number int default 25;
ALTER TABLE vector25_others ADD ID SERIAL PRIMARY KEY;
```

## Ceating LULC Map and Exploration
```sql

-- Creating the LULC Map

with lulc_map as (
    select objectid, code, name, wkb_geometry
    from ground_cover_with_name

    union all

    select objectid, number, code, wkb_geometry
    from settlement

    union all

    select objectid, number, code, geom
    from vector25_farm
    union all

    select objectid, number, code, geom
    from vector25_others
)select * into lulc.lulc_map
from lulc_map;

ALTER TABLE lulc_map ADD ID SERIAL PRIMARY KEY;
```


```sql
-- Exploration the dataset
select distinct name,code,round(((sum(st_area(wkb_geometry)))/10000)::numeric,2) as area_ha, count(*)
from lulc_map
group by name,code
order by area_ha;
```

```{note}
:class: dropdown
the results is as follow:

|    Name               |   Area(ha)     |  Count        |
| :-------------------- | -------------  |-------------: |
|  water course         |   13989.7      |  1088         |
|   wetlands            |   18385.67     |  12080        |
|   spare forests       |   35111.75     |  13286        |
|   brushland/scrubland |   62442.2      |  27347        | 
|   glacier             |   106742       |  5349         |
|   stagnant waters     |   18385.67     |  16537        | 
|   rock                |   305289.32    |  145952       |
|   loose rocks         |   324763.28    |  12080        |
|   dense forest        |   1090630.03   |  102296       |
|   settlement          |   360576.44    |  57145        |
|   agricultural area   |   46306.26     |  43689        |
|   others              |   2230695.54   |  143949       |
```


## Converting the Vectot to Raster
```bash
gdal_rasterize -l lulc.lulc_map -a code -ts 1000.0 1000.0 -a_nodata 0.0 -te 485422.117343434 75159.3856499034 834069.939666441 298248.830831526 -ot Float32 -of GTiff "PG:dbname='pochasdb' host=127.0.0.2 port=5432 user='gis' password='gis'" swiss_lulc_map.tif
```


```{image} /pics/lulc_map.PNG
:alt: fishy
:class: bg-primary
:width: 500px
:align: center
```


