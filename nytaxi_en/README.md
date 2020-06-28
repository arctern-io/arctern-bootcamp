# Analyzing New York City Taxi Dataset with Arctern

This tutorial uses the New York City taxi dataset as an example to describe how to use Arctern to process large geospatial data and use kepler.gl to visualize data.

## Prerequisite

#### [Install Arctern](https://arctern.io/docs/versions/v0.2.x/development-doc-en/html/quick_start/standalone_installation.html)

#### Install Jupyter Notebook

In the previous step, you have set up the `artern_env` environment. Enter this environment and run the following command to install Jupyter Notebook:

```bash
$ conda install -c conda-forge notebook
```

#### Install required libraries

In the `arctern_env` environment, run the following command to install required libraries:

```bash
$ pip install keplergl pyshp sridentify
```

## Data Preparation

Download the New York City taxi dataset, which includes 200,000 records of New York City taxi data and the New York City topographic map. Save the data in `/tmp` by default:

```bash
$ cd /tmp
# Download New York City Taxi data
$ wget https://raw.githubusercontent.com/arctern-io/arctern-bootcamp/master/nytaxi/file/0_2M_nyc_taxi_and_building.csv
# Download and unzip the topographic map of New York
$ wget https://github.com/arctern-io/arctern-bootcamp/raw/master/nytaxi/file/taxi_zones.zip
$ unzip -d taxi_zones taxi_zones.zip
# Download New York City road network data
$ wget https://raw.githubusercontent.com/arctern-io/arctern-bootcamp/master/nytaxi/file/nyc_road.csv
# Download kepler.gl config file
$ wget https://raw.githubusercontent.com/arctern-io/arctern-bootcamp/master/nytaxi/file/map_config.json
```

## Initialize Jupyter Notebook

Download [arctern_nytaxi_bootcamp.ipynb](./arctern_nytaxi_bootcamp.ipynb), and then start Jupyter Notebook in the `arctern_env` environment：

```bash
$ wget https://raw.githubusercontent.com/arctern-io/arctern-bootcamp/master/nytaxi_en/arctern_nytaxi_bootcamp.ipynb
# Start Jupyter Notebook
$ jupyter notebook
```

Open **arctern_nytaxi_bootcamp.ipynb** in Jupyter Notebook. Let's start to have fun with the example codes.

## Introduction of example codes

This example includes codes for data cleansing and data analysis.

### 1. Data cleansing

This tutorial uses 200,000 records extracted from the New York City taxi dataset. Noisy data is inevitable when dealing with data in such scale and they interfere with the results directly, so efficiently identifying and cleansing noisy data is key to the data analyzing procedure.

#### 1.1 Data loading

First, define the schema `nyc_schema` that describes column names and data types of the 200,000 records, and load these records into the dataframem `nyc_df`.

```python
import pandas as pd
nyc_schema={
    "VendorID":"string",
    "tpep_pickup_datetime":"string",
    "tpep_dropoff_datetime":"string",
    "passenger_count":"int64",
    "trip_distance":"double",
    "pickup_longitude":"double",
    "pickup_latitude":"double",
    "dropoff_longitude":"double",
    "dropoff_latitude":"double",
    "fare_amount":"double",
    "tip_amount":"double",
    "total_amount":"double",
    "buildingid_pickup":"int64",
    "buildingid_dropoff":"int64",
    "buildingtext_pickup":"string",
    "buildingtext_dropoff":"string",
}
nyc_df=pd.read_csv("/tmp/0_2M_nyc_taxi_and_building.csv",
               dtype=nyc_schema,
               date_parser=pd.to_datetime,
               parse_dates=["tpep_pickup_datetime","tpep_dropoff_datetime"])
```

#### 1.2 Data display

The data set includes the longitudes and latitudes of pick-up and drop-off locations for each taxi trip. Via Arctern and kepler.gl, we can visualize all these locations on a map to get a better understanding of these data. 

Load the pick-up locations:

```python
import arctern
from arctern import GeoSeries
from keplergl import KeplerGl

pickup_points = GeoSeries.point(nyc_df.pickup_longitude,nyc_df.pickup_latitude)
KeplerGl(data={"pickup_points": pd.DataFrame(data={'pickup_points':pickup_points.to_wkt()})})
```

<img src="./pic/nyc_taxi_pickup_all.png">

With the visualized results on the map, we can identify the noisy data easily, as some of the pick-up locations are in the ocean. These noisy data need to be filtered.

#### 1.3 Data filter

To get rid of the noisy data, we can filter the data according to the topographic map of New York City. Specifically, if the pick-up or drop-off location of a record goes beyond the boundary of New York City, this record should be filtered out.

##### 1.3.1 Data conversion

Load the New York City topographic map from the GeoJSON file with Arctern:

```python
import shapefile
import json
# Read the topographic data map of New York City
nyc_shape = shapefile.Reader("/tmp/taxi_zones/taxi_zones.shp")
nyc_zone=[ shp.shape.__geo_interface__  for shp in nyc_shape.shapeRecords()]
nyc_zone=[json.dumps(shp) for shp in nyc_zone]
# Read the data with Arctern
nyc_zone_series=pd.Series(nyc_zone)
nyc_zone_arctern=GeoSeries.geom_from_geojson(nyc_zone_series)
nyc_zone_arctern.to_wkt()
```

Display the loaded map data:

```
0      POLYGON ((933100.91835271 192536.085697202,933...
1      MULTIPOLYGON (((1033269.24359129 172126.007812...
2      POLYGON ((1026308.76950666 256767.697540373,10...
3      POLYGON ((992073.46679686 203714.07598877,9920...
4      POLYGON ((935843.310493261 144283.335850656,93...
                             ...                        
258    POLYGON ((1025414.78196019 270986.139363825,10...
259    POLYGON ((1011466.96605045 216463.005203798,10...
260    POLYGON ((980555.204311222 196138.486258477,98...
261    MULTIPOLYGON (((999804.794550449 224498.527048...
262    POLYGON ((997493.322715312 220912.386162326,99...
Length: 263, dtype: object
```

Get the current coordinate system of the New York City topographic map, and use Arctern to convert its coordinate reference system to "EPSG: 4326":

```python
from sridentify import Sridentify
ident = Sridentify()
ident.from_file('/tmp/taxi_zones/taxi_zones.prj')
src_crs = ident.get_epsg()
nyc_zone_arctern.set_crs(f'EPSG:{src_crs}')
nyc_arctern_4326 = nyc_zone_arctern.to_crs(crs="EPSG:4326")
nyc_arctern_4326.to_wkt()
```

This is the results after coordinate system conversion:

```
0      POLYGON ((-74.184453 40.694996,-74.184489 40.6...
1      MULTIPOLYGON (((-73.8233759726066 40.638987047...
2      POLYGON ((-73.8479261409998 40.871342234,-73.8...
3      POLYGON ((-73.9717741096532 40.7258212813371,-...
4      POLYGON ((-74.1742173809999 40.5625680859999,-...
                             ...                        
258    POLYGON ((-73.851071161919 40.910371520111,-73...
259    POLYGON ((-73.9017537339999 40.760775475,-73.9...
260    POLYGON ((-74.0133261089999 40.7050307879999,-...
261    MULTIPOLYGON (((-73.9438325669999 40.782859089...
262    POLYGON ((-73.95218622 40.7730198449999,-73.95...
Length: 263, dtype: object
```

With the converted latitude and longitude coordinates, the topographic map of New York City is rendered as follows:

```python
KeplerGl(data={"nyc_zones": pd.DataFrame(data={'nyc_zones':nyc_arctern_4326.to_wkt()})})
```
<img src="./pic/nyc_shape_all.png">

##### 1.3.2 Data cleaning

In order to clean up the noisy data, we can filter out records with pick-up locations outside the boundaries of New York City.

```python
index_nyc = arctern.within_which(pickup_points, nyc_arctern_4326)
is_in_nyc = index_nyc.notna()
pickup_in_nyc = pickup_points[pd.Series(is_in_nyc)]
```

Display the pick-up locations after filtering.

```python
KeplerGl(data={"pickup_points": pd.DataFrame(data={'pickup_points':pickup_in_nyc.to_wkt()})})
```
<img src="./pic/nyc_taxi_pickup_filted.png">

Filter these data by the drop-off locations.

```python
dropoff_points = GeoSeries.point(nyc_df.dropoff_longitude,nyc_df.dropoff_latitude)
index_nyc = arctern.within_which(dropoff_points, nyc_arctern_4326)
is_dorpoff_in_nyc = index_nyc.notna()
dropoff_in_nyc=dropoff_points[is_dorpoff_in_nyc]
KeplerGl(data={"drop_points": pd.DataFrame(data={'drop_points':dropoff_in_nyc.to_wkt()})})
```
<img src="./pic/nyc_taxi_dropoff_filted.png">


To clean all noisy data, we filter data with the pick-up or drop-off locations outside the boundaries of New York City:


```python
in_nyc_df=nyc_df[is_in_nyc & is_dorpoff_in_nyc]
in_nyc_df.fare_amount.describe()
```

The summarized travel cost information for the filtered data is as follows:


    count    195479.000000
    mean          9.791914
    std           7.266372
    min           2.500000
    25%           5.700000
    50%           7.700000
    75%          11.300000
    max         175.000000
    Name: fare_amount, dtype: float64

After filtering the data according to the map of New York City, we find that some locations are far from roads, and even plotted on certain buildings:

```python
import json
with open("/tmp/map_config.json", "r") as f:
    config = json.load(f)
KeplerGl(data={"projectioned_point": pd.DataFrame(data={'projectioned_point':pickup_in_nyc.to_wkt()})},config=config)
```
<img src="./pic/nyc_taxi_pickup_filted_small_zone.png">

We think that the data farther away from roads are noisy data, which are more than 100 meters from the road. So we have to filter the noisy data according to the New York City road network.

First, load the New York City road network:

```python
import arctern
nyc_road=pd.read_csv("/tmp/nyc_road.csv", dtype={"roads":"string"}, delimiter='|')
roads=GeoSeries(nyc_road.roads)
```

Filter data based on both the pick-up and drop-off locations:

```python
pickup_points = GeoSeries.point(in_nyc_df.pickup_longitude,in_nyc_df.pickup_latitude)
pickup_points.set_axis(in_nyc_df.index,inplace=True)
dropoff_points = GeoSeries.point(in_nyc_df.dropoff_longitude,in_nyc_df.dropoff_latitude)
dropoff_points.set_axis(in_nyc_df.index,inplace=True)

is_pickup_near_road = arctern.near_road(roads, pickup_points)
is_dropoff_near_road = arctern.near_road(roads, dropoff_points)

is_near_road = is_pickup_near_road & is_dropoff_near_road

on_road_nyc_df = in_nyc_df[is_near_road]
```

After filtering out the data far away from the road, we bind the pick-up locations to the nearest roads to generate new pick-up locations:

```python
pickup_points = GeoSeries.point(on_road_nyc_df.pickup_longitude,on_road_nyc_df.pickup_latitude)
pickup_points.set_axis(on_road_nyc_df.index,inplace=True)
projectioned_pickup = arctern.nearest_location_on_road(roads, pickup_points)
projectioned_pickup = GeoSeries(projectioned_pickup)
```

Plot the new pick-up locations:

```python
KeplerGl(data={"projectioned_point": pd.DataFrame(data={'projectioned_point':projectioned_pickup.to_wkt()})},config=config)
```
<img src="./pic/nyc_taxi_pickup_on_road.png">

Bind the drop-off locations to the nearest roads to generate new drop-off locations:

```python
dropoff_points = GeoSeries.point(on_road_nyc_df.dropoff_longitude,on_road_nyc_df.dropoff_latitude)
dropoff_points.set_axis(on_road_nyc_df.index,inplace=True)
projectioned_dropoff = arctern.nearest_location_on_road(roads, dropoff_points)
projectioned_dropoff = GeoSeries(projectioned_dropoff)
KeplerGl(data={"projectioned_point": pd.DataFrame(data={'projectioned_point':projectioned_dropoff.to_wkt()})},config=config)
```
<img src="./pic/nyc_taxi_dropoff_on_road.png">

After binding the pick-up and drop-off loactions to the roads, add the information to  `on_road_nyc_df`:

```python
on_road_nyc_df.insert(16,'pickup_on_road',projectioned_pickup)
on_road_nyc_df.insert(17,'dropoff_on_road',projectioned_dropoff)
on_road_nyc_df.fare_amount.describe()
```

The summarized travel cost information for the filtered data are as follows:

    count    194786.000000
    mean          9.692384
    std           6.976573
    min           2.500000
    25%           5.700000
    50%           7.700000
    75%          11.000000
    max         175.000000
    Name: fare_amount, dtype: float64

Now, we have all data cleaned, so we can continue to analyze these data. 

### 2. Data analysis

It is very important to clean up data because it ensures valid analysis results. Next, we will analyze the New York City taxi dataset based on the transaction amounts and straight-line distances.

#### 2.1 About amount

Plot pick-up and drop-off locations with transaction amount greater than $50:

```python
fare_amount_gt_50 = on_road_nyc_df[on_road_nyc_df.fare_amount > 50]
KeplerGl(data={"pickup": pd.DataFrame(data={'pickup':fare_amount_gt_50.pickup_on_road.to_wkt()}),
               "dropoff":pd.DataFrame(data={'dropoff':fare_amount_gt_50.dropoff_on_road.to_wkt()})
              })
```

<img src="./pic/nyc_taxi_fare_gt_50.png">

You can interact with the map by expanding the small triangle in the upper left corner, such as hiding the pick-up or drop-off locations. We can see that most trips with the transaction amount greater than $50 are from the city center to faraway places.

#### 2.2 About distance

Calculate the straight-line distance between the pick-up and the drop-off locations:

```python
on_road_nyc_df.pickup_on_road.set_crs("EPSG:4326")
on_road_nyc_df.dropoff_on_road.set_crs("EPSG:4326")
nyc_distance=on_road_nyc_df.pickup_on_road.distance_sphere(on_road_nyc_df.dropoff_on_road)
nyc_distance.describe()
```

The straight-line distance summary for all the trips are as follows:

```
count    194786.000000
mean       3113.344497
std        3232.008220
min           0.000000
25%        1224.650347
50%        2087.753029
75%        3730.790193
max       35418.698339
dtype: float64
```

Get the pick-up and the drop-off locations for trips with a straight-line distance greater than 20 kilometers, and plot them.

```python
nyc_with_distance=pd.DataFrame({"pickup":on_road_nyc_df.pickup_on_road,
                                "dropoff":on_road_nyc_df.dropoff_on_road,
                                "sphere_distance":nyc_distance
                               })

nyc_dist_gt = nyc_with_distance[nyc_with_distance.sphere_distance > 20e3]
KeplerGl(data={"pickup": pd.DataFrame(data={'pickup':nyc_dist_gt.pickup.to_wkt()}),
               "dropoff":pd.DataFrame(data={'dropoff':nyc_dist_gt.dropoff.to_wkt()})
              })
```

<img src="./pic/nyc_taxi_distance_gt_20km.png">

We can see that trips with straight-line distances greater than 20 kilometers are also from the city center to faraway places. 

Now you have completed the analysis of New York City Taxi dataset on the transaction amounts and straight-line distances. Please refer to **[Arctern API](https://arctern.io/docs/versions/v0.2.x/development-doc-en/html/api_reference/api_reference.html)** for more function descriptions.
