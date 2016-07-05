---
layout: post
title: The Hilliest Muni Route in San Francisco
---
![cc]({{ site.baseurl }}/images/cable_car.jpg)

## Introduction

I ride SF Muni almost every day and often think about the characteristics of different bus routes - when they are busiest, who rides them and for what purposes, what neighborhoods do they go through, etc. I was excited to see that the Metropolitan Transportation Commission releases GIS data of all transit routes in the Bay Area. There are tons of really interesting and important ways that this can be analyzed. In this post here I use the transit route data along with a datset of elevation contour lines to answer a ver simple question: What is the hilliest Muni line in the city? (spoiler alert: it's not a bus). I also use a few other publically available spatial datasets (streets and neighborhood boundaries) to create some interesting plots of elevation profiles that help to visualize the bus routes. The following analysis is all done using open source spatial tools in Python. An IPython notebook with all my code can be found [here](https://github.com/agaidus/SF_Muni_Elevation/blob/master/SF%20Muni%20-%20Elevation%20Profiles.ipynb).

## Muni Data
Bay Area transportation routes were downloaded from the MTC website from which Muni lines were selected out from the entire set of Bay Area transportation. After a little more clean-up I get the following dataframe which contains a ```shapely``` geometry object for each in-bound and out-bound Muni route in the city. The geometry objects are displayed in well-known text format, and are a just a way of mapping and analyzing vector geometry.

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
Contour lines were downloaded as a shapefile from the San Francisco [open data portal](https://data.sfgov.org/). Like the Muni lines I read the contours into a dataframe. Each record joins points of equal elevation and contains the elevation value (feet above sea level) as well as a geometry object. Plotting these and symbolizing by elevation value shows that the center of the city, which contains Mt. Sutro, Twin Peaks, and Mt. Davidson is at a much higher elevation than the rest of the city.

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles-Copy1_15_1.png)


## Calculating Bus Route Elevation Change

Overlaying bus routes and contour lines allows me to calculate stats on the elevation change of each bus route. I write a function that intersects each bus route with contour lines, resulting in a set of ordered points along each bus route with the elevation value at that point. I also calculate the distance from route start that each point represents, which will allow me to plot elevation profiles. However, before that, I can very easily calculate the total elevation change for each route. This is done by summing up the absolute value of the 1st discrete difference of the elevation points. If I instead wanted to calculate the total elevation **gain**, I would sum up only the positive 1st discrete difference values.

I write a function that first aggregates together in-bound and out-bound routes, and then calculates for a particular bus-line in its entirety, the total elevation change. Then by applying this function to the entire set of Muni routes, I can compare the relative hilliness of a route. In order to appropriately make this comparison, I divide the total elevation change by the length to calculate feet of elevation change per mile.

The top 5 hilliest routes are shown below with feet of elevation change per mile. No surprise is the fact that 3 of the top 5 are the notoriously steep cable car routes that run over Nob Hill. The 35 and 36 bus lines both run through the center of the city along the eastern and western sides of Twin Peaks, so they both cover some hilly terrain as well.

    Name
    POWELL-HYDE     421.212388
    35              370.737733
    POWELL-MASON    325.903532
    CALIFORNIA      282.808092
    36              271.951503
    dtype: float64



## Plotting Bus Route Neighborhood Composition

The next step I take is to look at San Francisco neighborhood boundaries to see which neighborhoods bus routes pass through at what points along their route. This will be used as another piece of information that can be included on the elevation profile plots. I download a neighborhood boundary shapefile from the city open data portal and read it into a dataframe, which I plot below.

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_30_0.png)

I write a function that intersects buslines with neighborhoods and calculates for a given bus line the neighborhoods that it goes through along with the distance from start of route that it "switches" neighborhoods. Running this function on the 38 inbound bus line, we see that the bus starts in the Outer Richmond and ends in the Financial district. The values represent the distance (in meters) at which the bus first enters that neighborhood. If I instead ran the function on the outbound bus route, the neighborhood order would be reversed.

    neighborho
    Outer Richmond               0.000000
    Inner Richmond            3705.582208
    Presidio Heights          5479.587075
    Western Addition          6496.081302
    Downtown/Civic Center     9387.312089
    Financial District       11029.476944
    Name: start_distance, dtype: float64

This function also produces the information needed to plot the proportion of a bus route's length spent in each neighborhood. I run this on the 38 inbound below. As you can see, this bus spends nearly half of its route in the Richmond District (Inner + Outer). 

![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_38_0.png)


##  Calculating Cross Streets at Points of Max and Min Elevation

Before I go ahead and make plots of bus elevation profiles, there's one more piece of information that I think would be really interesting to have and include on these plots - the name of the cross-streets at the maximum and minimum elevation values of a route. Using a shapefile of San Francisco streets I can do precisely that. The overall methodology I used invovles using the street geometry to find the closest street segment that is perpendicular to the bus line at the maximum and minimum elevation points. 

First, I need a way to identify perpendicular segments. I write a function that uses vector algebra to calculate the angle between two line segments. I apply it to two sample lines below as an example.

```python
line1=[[0,-2],[10,1]]
line2=[[1,-1],[1,-10]]
```
![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_48_1.png)

```python
get_angle_degrees(line1, line2)
```
    106.69924423399362

Then I write a function that builds on top of this and take an input point along a bus route, and returns the name of the street segment that is the closest perpendicular segment to the muni line at the input point. No cross-street will be exactly perpendicular, so I define it as any street between 85 and 95 degrees. Below, I show an example where I calculate the point exactly halfway along the 38 outbound and then use the function defined above to find the nearest cross-street to this point. 

```python
muni38_line=muni.loc[('38','OB')]
mid_way_point=muni38_line.interpolate(muni38_line.length/2)
nearest_cross_street(mid_way_point, ('38','OB'))
```
    'Presidio'

This algorithm identifies the nearest cross-street as being Presidio, which is also shown on the map below. The black dot represents the point on the Muni line and the nearest cross-street is highlighted in yellow. 

![png]({{ site.baseurl }}/images/streetangles.png)

By simply appylying this function to the maximum and minimum elevation points (instead of the half-way point as I did in this example) I can get that last piece of info that I want to include in the elevvation profile plots.

## Final Elevation Profile Plots
Now by pulling all of these functions and data together, I write a final plotting function that takes as its input the name of a Muni route name and direction, plots the elevation profile, indicates the point along the route at which the bus line switches neighborhoods, and marks the point of max and min elevation along with the elevation and the cross-street at those points. I also calculate the total elevation change per mile and include that in the plot title. Below I apply it to a few of my most commonly used Muni lines as well as the routes with the largest and smallest elevation gains. 


![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_57_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_58_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_59_0.png)



![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_60_0.png)


![png]({{ site.baseurl }}/images/SF%20Muni%20-%20Elevation%20Profiles_61_0.png)
