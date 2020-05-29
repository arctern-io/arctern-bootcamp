# Analyzing NYC Taxi Dataset with Arctern

## Environment requirements

- #### [Install Arctern](https://arctern.io/docs/versions/v0.2.x/development-doc-cn/html/quick_start/standalone_installation.html)

- #### Install Jupyter

  Run the following command in the `artern_env` environment of the previous step to install Jupyter:

  ```bash
  $ conda install -c conda-forge jupyterlab
  ```
  
- #### Install required libraries

  Run the following command in the `arctern_env` environment to install required libraries:

  ```bash
  $ pip install keplergl pyshp sridentify
  ```



## Download data

We will introduce how to analyze NYC Taxi dataset with Arctern, and use keplergl to display the data. You need to download 200,000 New York taxi data and New York City topographic map. We are downloaded it to `/tmp` default:

```bash
$ cd /tmp
# Download NYC Taxi data
$ wget https://raw.githubusercontent.com/zilliztech/arctern-bootcamp/master/nytaxi/file/0_2M_nyc_taxi_and_building.csv
# Download and unzip the topographic map of New York
$ wget https://github.com/zilliztech/arctern-bootcamp/raw/master/nytaxi/file/taxi_zones.zip
$ unzip -d taxi_zones taxi_zones.zip
```



## Starting jupyter-notebook

Download [arctern_nytaxi_bootcamp.ipynb](./arctern_nytaxi_bootcamp.ipynb) ，running jupyter notebook in `arctern_env` environment：

```bash
$ wget https://raw.githubusercontent.com/zilliztech/arctern-bootcamp/master/nytaxi_en/arctern_nytaxi_bootcamp.ipynb
# starting jupyter notebook
$ jupyter notebook
```

Open arctern_nytaxi_bootcamp.ipynb in jupyter web, then you can run it.



## New York taxi data analysis

Next, we will introduce how to use Arctern to analyze large-scale gis data, and combine Keplergl to display data, you can also run the [arctern_nytaxi_bootcamp.ipynb](./arctern_nytaxi_bootcamp.ipynb) directly with jupyter.

### 1. Data preprocessing

The sample data we used is extracted 200,000 from New York taxi record data. When we deal with large-scale data, there is usually some noisy data, and these noisy data are not easy to find but affect the results directly. So how to quickly find noisy data and data preprocessing is a key point in analysis.

#### 1.1 Data loading

First of all, you can define a table's  `schema` to describe all column names and their types . Of course, you can modify the ` schema` to load your own data.

```python
import pandas as pd
nyc_schame={
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
               dtype=nyc_schame,
               date_parser=pd.to_datetime,
               parse_dates=["tpep_pickup_datetime","tpep_dropoff_datetime"])
```

#### 1.2 Data display

According above  table's `schema` , we can see this data mainly include longitude and latitude of up and down taxi. And we can use Arctern and keplergl to display all the gis points on the map, it can show us some issue. First loading the pick-up point:

```python
import arctern
from keplergl import KeplerGl

pickup_points = arctern.ST_Point(nyc_df.pickup_longitude,nyc_df.pickup_latitude)
KeplerGl(data={"pickup_points": pd.DataFrame(data={'pickup_points':arctern.ST_AsText(pickup_points)})})
```

<img src="./pic/nyc_taxi_pickup_all.png">

The map results support select and drag, we can see it has noisy data, because the pick-up point are painted on the sea. In fact, all the data should be concentrated on land. These noisy data need to clean and filter.

#### 1.3 Data filter

In order to correctly analyze the NYC taxi data, we will filter the data according to the topographic map of New York City. The data that is not in the New York City map is regarded as noisy and needed filtered. This step will also introduce how to loaded GeoJSON data and converted it to "EPSG: 4326", which is the latitude and longitude coordinates.

##### 1.3.1 Data convert

Read the topographic data map of New York City. The topographic data is stored in GeoJSON format. First, use Arctern to load the GeoJSON data:

```python
import shapefile
import json
nyc_shape = shapefile.Reader("/tmp/taxi_zones/taxi_zones.shp")
nyc_zone=[ shp.shape.__geo_interface__  for shp in nyc_shape.shapeRecords()]
nyc_zone=[json.dumps(shp) for shp in nyc_zone]
nyc_zone_series=pd.Series(nyc_zone)
nyc_zone_arctern=arctern.ST_GeomFromGeoJSON(nyc_zone_series)
arctern.ST_AsText(nyc_zone_arctern)
```

The results of the data read by Arctern are as follows:

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

Get the coordinate system of the current New York City topographic map, and use Arctern to convert the coordinate system to a latitude and longitude coordinate system, which is "EPSG: 4326":

```python
from sridentify import Sridentify
ident = Sridentify()
ident.from_file('/tmp/taxi_zones/taxi_zones.prj')
src_crs = ident.get_epsg()
nyc_arctern_4326 = arctern.ST_Transform(nyc_zone_arctern,f'EPSG:{src_crs}','EPSG:4326')
arctern.ST_AsText(nyc_arctern_4326)
```

The is the results after coordinate conversion:

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

According to the converted latitude and longitude coordinates, the topographic map of New York City is drawn as follows:

```python
KeplerGl(data={"nyc_zones": pd.DataFrame(data={'nyc_zones':arctern.ST_AsText(nyc_arctern_4326)})})
```
<img src="./pic/nyc_shape_all.png">

##### 1.3.2 Data cleaning

In order to analyze the taxi data in the New York City area, according to the topographic map of New York City, we see that the points that are not in the map are noise. In order to filter the noisy data, first we filter the pick-up points based on the skeleton map of New York City:


```python
# this step will cost some time
index_nyc = arctern.sjoin(pickup_points, nyc_arctern_4326, 'within')
is_in_nyc = index_nyc.map(lambda x: x >= 0)
pickup_in_nyc = pickup_points[pd.Series(is_in_nyc)]
```

Display the pick-up point after data filtering:

```python
KeplerGl(data={"pickup_points": pd.DataFrame(data={'pickup_points':arctern.ST_AsText(pickup_in_nyc)})})
```
<img src="./pic/nyc_taxi_pickup_filted.png">

We know that there are latitude and longitude data of the pick-up point and the drop-off point in the New York taxi data, then according to the same method, filter the drop-off point:

```python
# this step will cost some time
dropoff_points = arctern.ST_Point(nyc_df.dropoff_longitude,nyc_df.dropoff_latitude)
index_nyc = arctern.sjoin(dropoff_points, nyc_arctern_4326, 'within')
is_dorpoff_in_nyc = index_nyc.map(lambda x: x >= 0)
dropoff_in_nyc=dropoff_points[is_dorpoff_in_nyc]
KeplerGl(data={"drop_points": pd.DataFrame(data={'drop_points':arctern.ST_AsText(dropoff_in_nyc)})})
```
<img src="./pic/nyc_taxi_dropoff_filted.png">


According to the latitude and longitude data of the pick-up point and the drop-off point, filter all noisy data on the dataset:


```python
is_resonable = [is_dorpoff_in_nyc[idx] & is_in_nyc[idx] for idx in range(0,len(is_in_nyc)) ]
in_nyc_df=nyc_df[pd.Series(is_resonable)]
in_nyc_df.fare_amount.describe()
```

The descriptive information about the travel cost of the filtered data is:


    count    195479.000000
    mean          9.791914
    std           7.266372
    min           2.500000
    25%           5.700000
    50%           7.700000
    75%          11.300000
    max         175.000000
    Name: fare_amount, dtype: float64
In summary, we have completed data filtering. Based on the preprocessed data, we will analyze the NYC  taxi data.

### 2. Data analysis

We cleaned and filtered the data, this step is very important, it can ensure that the analysis results are valid. Next, we will analyze the NYC taxi dataset based on the transaction amount and mileage distance.

#### 2.1 About amount

We extract data with fees greater than $50 according to the transaction amount, and plot taxi pick-up and drop-off points:


```python
fare_amount_gt_50 = in_nyc_df[in_nyc_df.fare_amount > 50]
pickup_50 = arctern.ST_Point(fare_amount_gt_50.pickup_longitude,fare_amount_gt_50.pickup_latitude)
dropoff_50 = arctern.ST_Point(fare_amount_gt_50.dropoff_longitude,fare_amount_gt_50.dropoff_latitude)
KeplerGl(data={"pickup": pd.DataFrame(data={'pickup':arctern.ST_AsText(pickup_50)}),
               "dropoff":pd.DataFrame(data={'dropoff':arctern.ST_AsText(dropoff_50)})
              })
```

<img src="./pic/nyc_taxi_fare_gt_50.png">

You can expand the small triangle in the upper left corner of the result map to operate the current layer, such as hiding the pick-up point or the drop-off point. We found that the cost is greater than 50 US dollars, which is triggered from the city center to a place farther away .

#### 2.2 About distance

We can also calculate the straight-line distance between the pick-up point and the drop-off point:


```python
nyc_distance=arctern.ST_DistanceSphere(arctern.ST_Point(in_nyc_df.pickup_longitude,
                                                        in_nyc_df.pickup_latitude),
                                       arctern.ST_Point(in_nyc_df.dropoff_longitude,
                                                        in_nyc_df.dropoff_latitude))
nyc_distance.index=in_nyc_df.index
nyc_distance.describe()
```
The linear distance of the taxi is described as:


```
    count    192805.000000
    mean       3150.931171
    std        3326.144461
    min           0.000000
    25%        1224.998626
    50%        2088.286128
    75%        3753.104118
    max       35395.487197
    dtype: float64
```

Get the points with a straight-line distance greater than 20 kilometers, and draw all pick-up and drop-off points with a straight-line distance greater than 20 kilometers:

```python
nyc_with_distance=pd.DataFrame({"pickup_longitude":in_nyc_df.pickup_longitude,
                                "pickup_latitude":in_nyc_df.pickup_latitude,
                                "dropoff_longitude":in_nyc_df.dropoff_longitude,
                                "dropoff_latitude":in_nyc_df.dropoff_latitude,
                                "sphere_distance":nyc_distance
                               })

nyc_dist_gt = nyc_with_distance[nyc_with_distance.sphere_distance > 20e3]
pickup_gt = arctern.ST_Point(nyc_dist_gt.pickup_longitude,nyc_dist_gt.pickup_latitude)
dropoff_gt = arctern.ST_Point(nyc_dist_gt.dropoff_longitude,nyc_dist_gt.dropoff_latitude)

KeplerGl(data={"pickup": pd.DataFrame(data={'pickup':arctern.ST_AsText(pickup_gt)}),
               "dropoff":pd.DataFrame(data={'dropoff':arctern.ST_AsText(dropoff_gt)})
              })
```

<img src="./pic/nyc_taxi_distance_gt_20km.png">

Similarly, we found that straight-line distances greater than 20 kilometers are also triggered from the city center to a place farther away. In summary, we have completed the analysis of NYC taxi data on transaction amount and straight-line distance, more analysis functions can refer to **[Arctern API](https://arctern.io/docs/versions/v0.2.x/development-doc-cn/html/api/pandas_api/pandas_api.html)**。
