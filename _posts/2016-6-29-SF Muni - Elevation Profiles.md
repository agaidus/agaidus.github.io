---
layout: post
title: The Hilliest Muni Route in San Francisco
---
![cc]({{ site.baseurl }}/images/cable_car.jpg)

## Introduction

I ride SF Muni almost every day and often think about the characteristics of different bus routes: when they are busiest, who rides them and for what purpose, what neighborhoods they pass through, etc. Being someone that loves playing with spatial data, I was excited to find that the Metropolitan Transportation Commission releases GIS data for all transit routes in the Bay Area and thought it would be fun to use the MTC data in conjunction with a dataset of elevation contour lines to answer a very simple question: What is the hilliest Muni line in the city? (spoiler alert: it’s not a bus). I also used a few other publicly available spatial datasets (streets and neighborhood boundaries) to create some interesting plots of elevation profiles that help to visualize the bus routes. 

In addition to being an really way to look at and compare bus lines, I use this post as a way to explore some of the open source GIS capablities of Python. I use ```Shapely``` and ```Geopandas``` for geometry manipulation and spatial overlays; ```Pandas``` for some more general data wrangling; and ```matplotib``` to make plots and to visualize the data. An IPython notebook with my code can be found [here](https://github.com/agaidus/SF_Muni_Elevation/blob/master/SF%20Muni%20-%20Elevation%20Profiles.ipynb).

## Muni Data

I downloaded Bay Area transportation routes from the MTC, read it into ```Geopandas``` and created a subset that included just the Muni lines. I then cleaned the data by parsing the route description into a route name and a route direction. This produced the following dataframe, which contains a ```shapely``` geometry object for each in-bound and out-bound Muni route in the city. The geometry objects are displayed in well-known text format, and are a just a way of mapping and analyzing vector geometry.

    Name  Direction
    38AX  OB           LINESTRING (553045.0462999996 4182964.82599999...
          IB           LINESTRING (543198.1178000001 4181476.8808, 54...
    38BX  OB           LINESTRING (553045.0462999996 4182964.82599999...
          IB           LINESTRING (545371.8230999997 4181533.7753, 54...
    38L   OB           LINESTRING (553160.7375999996 4182671.3609, 55...
          IB           LINESTRING (543198.1178000001 4181476.8808, 54...
    38    OB           LINESTRING (553160.7375999996 4182671.3609, 55...
          IB           LINESTRING (543148.8629999999 4180794.26679999...
    Name: geometry, dtype: object

	
![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_11_1.png)


## Elevation Data
I then downloaded contour lines as a shapefile from the San Francisco [open data portal](https://data.sfgov.org/). Like the Muni lines, I read the contours into a dataframe. Each record joins points of equal elevation and contains the elevation value (feet above sea level), as well as a geometry object. Plotting these and symbolizing by elevation value shows that the center of the city, which contains Mt. Sutro, Twin Peaks, and Mt. Davidson, is at a much higher elevation than the rest of the city.

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles-Copy1_15_1.png)

## Calculating Bus Route Elevation Change

Overlaying bus routes and contour lines allowed me to calculate stats on the elevation change of each bus route. I wrote a function that intersects each bus route with contour lines, resulting in a set of ordered points along each bus route with the elevation value at that point. I also calculated the distance from the route start to each of these points, which allowed me to plot elevation profiles.

I then wrote a function that builds on this output and aggregates in-bound and out-bound routes to calculate the total elevation change for each bus line. This is done by summing up the absolute value of the 1st discrete difference for each of the elevation points. If I instead wanted to calculate the total elevation **gain**, I would sum up only the positive 1st discrete difference values.

In order to appropriately compare the relative hilliness of the entire set of Muni routes, I divided the total elevation change of each route by its length to determine the feet of elevation change per mile.
The top 5 hilliest routes are shown below, with feet of elevation change per mile. No surprise is the fact that 3 of the top 5 are the notoriously steep cable car routes that run over Nob Hill. The 35 and 36 bus lines both run through the center of the city along the eastern and western sides of Twin Peaks, so they both cover some hilly terrain as well.

    Name
    POWELL-HYDE     421.212388
    35              370.737733
    POWELL-MASON    325.903532
    CALIFORNIA      282.808092
    36              271.951503
    dtype: float64



## Plotting Bus Route Neighborhood Composition

Next, I wanted to see which neighborhoods the bus routes pass through and at what points along their routes. I thought it would be interesting to include this information in my finalized elevation profile plots. 
I downloaded a neighborhood boundary shapefile from the city open data portal and read it into a dataframe, which I plot below.

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_30_0.png)

I then wrote a function that intersects bus lines with neighborhoods and calculates for a given bus line the neighborhoods that it passes through and the distance from start of route that it “switches” neighborhoods. Running this function on the 38 inbound bus line, we see that the bus starts in the Outer Richmond and ends in the Financial district. The values represent the distance (in meters) at which the bus first enters that neighborhood. If I instead ran the function on the outbound bus route, the neighborhood order would be reversed.

    neighborho
    Outer Richmond               0.000000
    Inner Richmond            3705.582208
    Presidio Heights          5479.587075
    Western Addition          6496.081302
    Downtown/Civic Center     9387.312089
    Financial District       11029.476944
    Name: start_distance, dtype: float64

This function also produces the information needed to plot the proportion of a bus route’s length spent in each neighborhood. I run this on the 38 inbound below. As you can see, this bus spends nearly half of its route in the Richmond District (Inner + Outer).

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_38_0.png)


##  Calculating Cross Streets at Points of Max and Min Elevation

Before I finalized the plots of bus elevation profiles, I thought there was one more piece of information that would be interesting to include: the name of the cross streets at the maximum and minimum elevation values of a route.
With the help of a shapefile of San Francisco streets, I was able to determine the closest street segment that is perpendicular to the bus line at the maximum and minimum elevation points.
First, I needed a way to identify perpendicular segments, so I wrote a function that uses vector algebra to calculate the angle between two line segments. The following is an example using two sample lines:

```python
line1=[[0,-2],[10,1]]
line2=[[1,-1],[1,-10]]
```
![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_48_1.png)

```python
get_angle_degrees(line1, line2)
```
    106.69924423399362

Then I wrote a function that takes an input point along a bus route, and returns the name of the street segment that is the closest perpendicular segment to the muni line at the input point. Since no cross-street will likely be exactly perpendicular, I defined “perpendicular” as any street between 85 and 95 degrees. Below, I show an example where I calculate the point exactly halfway along the 38 outbound and then use the function defined above to find the nearest cross-street to this point.

```python
muni38_line=muni.loc[('38','OB')]
mid_way_point=muni38_line.interpolate(muni38_line.length/2)
nearest_cross_street(mid_way_point, ('38','OB'))
```
    'Presidio'

This algorithm identifies the nearest cross-street as being Presidio, which is also shown on the map below. The black dot represents the point on the Muni line and the nearest cross-street is highlighted in yellow.

![png]({{ site.baseurl }}/images/streetangles.png)

By simply applying this function to the maximum and minimum elevation points (instead of the half-way point as I did in this example) I was able to get that last piece of info that I wanted to include in the elevation profile plots.

## Final Elevation Profile Plots
Equipped with the necessary data and functions, I wrote a final plotting function that takes as its input a Muni route name and direction, plots the elevation profile, indicates the point along the route at which the bus line switches neighborhoods, and marks the point of max and min elevation along with the elevation and cross-street at those points. I also calculated the total elevation change per mile and included that in the plot title. 
Below I apply it to a few of my most commonly used Muni lines as well as the routes with the largest and smallest elevation gains.


![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_57_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_58_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_59_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_60_0.png)


![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_61_0.png)

And that's it! I found this to be a really interesting exercise both in terms of the subject matter and the tools and steps required to make it happen. I hope you found it useful as well!
