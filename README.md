# Final Project

### Introduction
In the age of data science, there is a shift in geospatial processing from a traditional GIS to R. Raster processing is especially useful in R as it provides a fast way to read data and transform data in a less-complex file format. While this will provide a more complex set of instruction, the attached .Rmd file can be used to test the code.

### Objective
I will be using Sentinel-1 SAR (synthetic aperture radar) data to evaluate flooding in Nassau, Bahamas from Hurricane Dorian.

### Methodology
In order to identify change I will be analyzing 3 SAR images, one in the month prior to Dorian, one immediately following Dorian, and one a month prior to Dorian. In order to identify areas of flooding by taking the absolute value of the each images and then subtracting them. I will show the change from pre-Dorian to during-Dorian and pre-Dorian to post-Dorian.

### Prerequisites
For this analysis I used 5 packages and they can be installed as shown below.
```R
library(raster)
library(rgdal)
library(sf)
library(spatstat)
library(ggplot2)
```

### Functions
Since I am using repetitive processes on multiple datasets, I chose to utilize functions to streamline this process. 

#### Reprojecting Data
Prior to starting any geospatial project it is important to ensure that all data is proejcted the same way. With this function, I take either a shapefile, class 'sf', or a raster, class 'raster.'
```R
reproject <- function(proj, data){
  if (class(data) == "sf"){
    data <- spTransform(data, crs(proj))
  }
  else{
    data <- projectRaster(data, crs=crs(proj))
  }
}
```
#### Visualizing Change
