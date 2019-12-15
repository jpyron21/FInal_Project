# Final Project

### Introduction
In the age of data science, there is a shift in geospatial processing from a traditional GIS to R. Raster processing is especially useful in R as it provides a fast way to read data and transform data in a less-complex file format. While this will provide a more complex set of instruction, the attached .Rmd file can be used to test the code.

### Objective
I will be using Sentinel-1 SAR (synthetic aperture radar) data to evaluate flooding in Nassau, Bahamas from Hurricane Dorian.

### Methodology

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
