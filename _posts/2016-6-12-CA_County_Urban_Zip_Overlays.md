---
layout: post
title: County Urban Populations from Zip Classifications
---

I have a shapefile of California zip codes and a shapefile of California counties. I am given a set of rules that use housing unit density to classify zip codes into rural, urban, or exurban zip codes. The goal is to calculate the proportion of each county's population that lives within each of these zip code classifications.

I demonstrate how to do this below because I think it is an example that makes good use of the overlaying and aggregating features in Pandas/Geopandas and demonstrates how they can be used to answer this question quite easily.


```python
#Import necessary modules
%matplotlib inline
import pandas as pd
import geopandas as gpd
import os
from geopandas.tools import overlay
import shapely
```

First I read in the zip code shapefile to Geopandas. The GeoDataframe has both population and housing unit counts.


```python
zips=gpd.read_file("Data/Zips.shp").set_index('Zipcode')
zips.geometry=zips.geometry.buffer(0)
zips.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>HU</th>
      <th>POP</th>
      <th>geometry</th>
    </tr>
    <tr>
      <th>Zipcode</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>12</th>
      <td>0</td>
      <td>0</td>
      <td>POLYGON ((-84069.50625625165 251173.9768131553...</td>
    </tr>
    <tr>
      <th>16</th>
      <td>28</td>
      <td>32</td>
      <td>POLYGON ((146716.0443333071 -276914.8214906069...</td>
    </tr>
    <tr>
      <th>17</th>
      <td>56</td>
      <td>15</td>
      <td>POLYGON ((88071.41057265643 -49057.84702989308...</td>
    </tr>
    <tr>
      <th>18</th>
      <td>0</td>
      <td>0</td>
      <td>POLYGON ((93928.84706553049 -381979.052042436,...</td>
    </tr>
    <tr>
      <th>19</th>
      <td>26</td>
      <td>36</td>
      <td>POLYGON ((-9312.641563633806 307120.0380001596...</td>
    </tr>
  </tbody>
</table>
</div>



I am given the following set of rules to classify zip codes in urban, rural, and suburban zip codes:
- **Urban**: > 1600 housing units per square mile
- **Rural**: < 64 housing units per square mile
- **Exurban**: between 64 and 64 housing units per square mile

I create a series of housing unit densities and then populate a new field called "Class" that categorizes each zip code into one of these categories by querying HU density series. Below, I show that roughly a third of zip codes fall into each of these categories.


```python
#Calculate housing unit density
hu_p_sqmi=zips.HU/(zips.area/2.59e+6)
zips.loc[hu_p_sqmi>1600,'Class']='Urban'
zips.loc[hu_p_sqmi<64,'Class']='Rural'
zips.Class.fillna('Exurban',inplace=True)
zips.Class.value_counts() / len(zips)
```




    Rural      0.366999
    Exurban    0.347622
    Urban      0.285379
    dtype: float64



Then I read in a shapefile of California counties. The next step is to determine for each zip code, which county it is located within. However, this is not super straightforward because of the fact that zip codes are not nested neatly within counties. Therefore, I instead have to determine the 'best-matched' county, which I define as the county that the zip code has the largest portion of its area within. There are a number of other ways that this could be done such as choosing the county that the zip code's centroid is located in (not as accurate), or choosing the county that has the largest portion of its population within (more accurate, but also requiring block level population data). 


```python
counties=gpd.read_file("Data/Counties.shp").set_index('FIPS')[['NAME','geometry']]
counties.geometry=counties.geometry.buffer(0)
counties.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>NAME</th>
      <th>geometry</th>
    </tr>
    <tr>
      <th>FIPS</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>06001</th>
      <td>Alameda</td>
      <td>(POLYGON ((-170110.5438936634 -30635.387532317...</td>
    </tr>
    <tr>
      <th>06003</th>
      <td>Alpine</td>
      <td>POLYGON ((8739.968452133671 101531.7376470827,...</td>
    </tr>
    <tr>
      <th>06005</th>
      <td>Amador</td>
      <td>POLYGON ((-6755.187296494563 77002.27672929503...</td>
    </tr>
    <tr>
      <th>06007</th>
      <td>Butte</td>
      <td>(POLYGON ((-175988.7392603862 205877.926178496...</td>
    </tr>
    <tr>
      <th>06009</th>
      <td>Calaveras</td>
      <td>(POLYGON ((-36381.80589585088 7475.7498299703,...</td>
    </tr>
  </tbody>
</table>
</div>



Pursuing my method of matching counties to zip codes based on areal overlap, I use the overlay tools in Geopandas to calculate the intersection between zip codes and counties. I then group by zip code and county FIPS while summing up the area of each zip / county combination. Then by dividing this by the area of zip codes, I get a series that contains for each zip code, the proportion of its population in each county that it intersects. Most zip codes are pretty close to being entirely located within one county.


```python
zip_x_cnty=overlay(counties.reset_index(),zips.reset_index(),how='intersection')
zip_x_cnty_area=zip_x_cnty.groupby(['Zipcode','FIPS']).apply(lambda x:x.area.sum())/zips.area
zip_x_cnty_area.name='Area'
zip_x_cnty_area.head()
```

    Zipcode  FIPS 
    12       06035    0.994439
             06063    0.005561
    16       06029    1.000000
    17       06019    0.997102
             06027    0.002238
    Name: Area, dtype: float64



I then identify the "best-matched" county by grouping by zip code and selecting from each group, the county that has the largest area proportion value. As you can see, at least for the first few zip codes, this matches up with what we would expect to see given the proportions above.


```python
best_county=zip_x_cnty_area.reset_index().groupby('Zipcode').apply(lambda group:group.loc[group.Area.idxmax()])['FIPS']
best_county.head()
```




    Zipcode
    12    06035
    16    06029
    17    06019
    18    06111
    19    06035
    Name: FIPS, dtype: object



Now that we have a 1 county for each zip code, the next step is to determine the proportion of each county's population that falls within each of the 3 zip code categories. I group zip codes by county and by classification, while summing up population. Then by dividing the count in each type by the total, I get the proportion in each type.


```python
county_pop_counts=zips.groupby([best_county,zips.Class])['POP'].sum().unstack().fillna(0)
county_pop_pct=county_pop_counts.div(county_pop_counts.sum(1),axis=0)
county_pop_pct.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Class</th>
      <th>Exurban</th>
      <th>Rural</th>
      <th>Urban</th>
    </tr>
    <tr>
      <th>FIPS</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>06001</th>
      <td>0.364276</td>
      <td>0.032583</td>
      <td>0.603141</td>
    </tr>
    <tr>
      <th>06003</th>
      <td>0.000000</td>
      <td>1.000000</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>06005</th>
      <td>0.107546</td>
      <td>0.892454</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>06007</th>
      <td>0.440459</td>
      <td>0.391637</td>
      <td>0.167903</td>
    </tr>
    <tr>
      <th>06009</th>
      <td>0.004289</td>
      <td>0.995711</td>
      <td>0.000000</td>
    </tr>
  </tbody>
</table>
</div>



A quick sort of counties by proportion living in urban zips and by proportion living in rural zips shows about what we would expect: more people are living in urban areas in counties that are located in the Bay Area or the LA Basin; and more people are living in rural areas in counties located in the eastern and northern parts of the state.


```python
county_pop_pct.sort('Urban',ascending=False).reset_index()['FIPS'].map(counties['NAME'])[:5]
```




    0    San Francisco
    1           Orange
    2      Los Angeles
    3        San Mateo
    4      Santa Clara
    Name: FIPS, dtype: object




```python
county_pop_pct.sort('Rural',ascending=False).reset_index()['FIPS'].map(counties['NAME'])[:5]
```




    0      Siskiyou
    1    San Benito
    2        Lassen
    3         Modoc
    4          Mono
    Name: FIPS, dtype: object



Lastly, I make a quick plot that symbolizes counties based on their % living in urban zip codes. One surprise, perhaps, is that San Bernadino county (the large county in the southeastern part of the state that contains Joshua Tree and the Mojave Desert) is showing up as having a large urban population. This is because the overwhelming majority of the population in that county lives in the corner that is right outside Los Angeles. So while the county as a whole is very low density, the average person living in that county does not live in a low density area.


```python
gdf=gpd.GeoDataFrame(county_pop_pct, geometry=counties.geometry)
gdf.plot(column='Urban',scheme='QUANTILES', k=5, colormap='OrRd')
gdf.to_file('CountyPctZipCategory.shp')
```

![png]({{ site.baseurl }}/images/CA_County_Urban_Zip_Overlays_18_0.png)

