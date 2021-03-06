---
title: "GeoSpark: Bring sf to spark"
output:
  github_document:
    fig_width: 9
    fig_height: 5
---

## Introduction

`geospark` R package aims at bringing local `sf` functions to distributed spark mode with [GeoSpark](https://github.com/DataSystemsLab/GeoSpark) scala package.

Currently, geospark support most of important `sf` functions in spark, here is a [summary comparison](https://github.com/harryprince/geospark/blob/master/Reference.md).

## Performance Comparison

the performance comparison example comes from [geospark paper](https://pdfs.semanticscholar.org/347d/992ceec645a28f4e7e45e9ab902cd75ecd92.pdf):

No. |test case | the number of records
---|---|---
1|SELECT IDCODE FROM zhenlongxiang WHERE ST_Disjoint(geom,ST_GeomFromText(‘POLYGON((517000 1520000,619000 1520000,619000 2530000,517000 2530000,517000 1520000))’));|85,236 rows
2| SELECT fid FROM cyclonepoint WHERE ST_Disjoint(geom,ST_GeomFromText(‘POLYGON((90 3,170 3,170 55,90 55,90 3))’,4326)) | 60,591 rows

query performance(ms)

No. | PostGIS/PostgreSQL |GeoSpark SQL| ESRI Spatial Framework for Hadoop
---|---|---|---
1 | 9631 | 480 |40,784
2 | 110872 |394| 64,217

appeartly, the Geospark SQL definitely outperforms PG and ESRI UDF under a very large data set.


## Prerequisites

* Apache Spark 2.X

## Getting Started

here is a mini example about optimized spatial join query with quadrad tree indexing: 

firstly, initialize envs

```{r}

devtools::install_github("harryprince/geospark")

library(sparklyr)
library(dplyr)
library(geospark)
conf = spark_config()
conf$spark.serializer <- "org.apache.spark.serializer.KryoSerializer"
conf$spark.kryo.registrator <- "org.datasyslab.geospark.serde.GeoSparkKryoRegistrator"
sc <- spark_connect(master = "local",config = conf)
```

register gis udf for spark connection

```
register_gis(sc)
```

prepare mock data

```{r}
polygons <- read.table(text="california area|POLYGON ((-126.4746 32.99024, -126.4746 42.55308, -115.4004 42.55308, -115.4004 32.99024, -126.4746 32.99024))
new york area|POLYGON ((-80.50781 36.24427, -80.50781 41.96766, -70.75195 41.96766, -70.75195 36.24427, -80.50781 36.24427))
texas area |POLYGON ((-106.5234 25.40358, -106.5234 36.66842, -91.14258 36.66842, -91.14258 25.40358, -106.5234 25.40358))
dakota area|POLYGON ((-106.084 44.21371, -106.084 49.66763, -95.71289 49.66763, -95.71289 44.21371, -106.084 44.21371))
", sep="|",col.names=c("area","geom"))
points <- read.table(text="New York|NY|POINT (-73.97759 40.74618)
New York|NY|POINT (-73.97231 40.75216)
New York|NY|POINT (-73.99337 40.7551)
West Nyack|NY|POINT (-74.06083 41.16094)
West Point|NY|POINT (-73.9788 41.37611)
West Point|NY|POINT (-74.3547 41.38782)
Westtown|NY|POINT (-74.54593 41.33403)
Floral Park|NY|POINT (-73.70475 40.7232)
Floral Park|NY|POINT (-73.60177 40.75476)
Elmira|NY|POINT (-76.79217 42.09192)
Elmira|NY|POINT (-76.75089 42.14728)
Elmira|NY|POINT (-76.84497 42.12927)
Elmira|NY|POINT (-76.80393 42.07202)
Elmira|NY|POINT (-76.83686 42.08782)
Elmira|NY|POINT (-76.75089 42.14728)
Alcester|SD|POINT (-96.63848 42.97422)
Aurora|SD|POINT (-96.67784 44.28706)
Baltic|SD|POINT (-96.74702 43.72627)
Beresford|SD|POINT (-96.79091 43.06999)
Brandon|SD|POINT (-96.58362 43.59001)
Minot|ND|POINT (-101.2744 48.22642)
Abercrombie|ND|POINT (-96.73165 46.44846)
Absaraka|ND|POINT (-97.21459 46.85969)
Amenia|ND|POINT (-97.25029 47.02829)
Argusville|ND|POINT (-96.95043 47.0571)
Arthur|ND|POINT (-97.2147 47.10167)
Ayr|ND|POINT (-97.45571 47.02031)
Barney|ND|POINT (-96.99819 46.30418)
Blanchard|ND|POINT (-97.25077 47.3312)
Buffalo|ND|POINT (-97.54484 46.92017)
Austin|TX|POINT (-97.77126 30.32637)
Austin|TX|POINT (-97.77126 30.32637)
Addison|TX|POINT (-96.83751 32.96129)
Allen|TX|POINT (-96.62447 33.09285)
Carrollton|TX|POINT (-96.89163 32.96037)
Carrollton|TX|POINT (-96.89773 33.00542)
Carrollton|TX|POINT (-97.11628 33.20743)
Celina|TX|POINT (-96.76129 33.32793)
Carrollton|TX|POINT (-96.89328 33.03056)
Carrollton|TX|POINT (-96.77763 32.76727)
Los Angeles|CA|POINT (-118.2488 33.97291)
Los Angeles|CA|POINT (-118.2485 33.94832)
Los Angeles|CA|POINT (-118.276 33.96271)
Los Angeles|CA|POINT (-118.3076 34.07711)
Los Angeles|CA|POINT (-118.3085 34.05891)
Los Angeles|CA|POINT (-118.2943 34.04835)
Los Angeles|CA|POINT (-118.2829 34.02645)
Los Angeles|CA|POINT (-118.3371 34.00975)
Los Angeles|CA|POINT (-118.2987 33.78659)
Los Angeles|CA|POINT (-118.3148 34.06271)", sep="|",col.names=c("city","state","geom"))

polygons_tbl <- copy_to(sc, polygons)
points_tbl <- copy_to(sc, points)

```

inner join query by `st_contains` function

> convert wkt into geometry object with 4326 crs which means wgs84 projection

```{r}

ex2 <- copy_to(sc,tbl(sc, sql("  SELECT  area,state,count(*) cnt from
                            (select area,ST_GeomFromWKT(polygons.geom ,'4326') as y  from polygons)  polygons,
                            (SELECT ST_GeomFromWKT (points.geom,'4326') as x,state,city from points) points
                            where  ST_Contains(polygons.y,points.x) group by area,state")),"test2")

collect(ex2)
```

close connection

```{r}
spark_disconnect_all()
```
