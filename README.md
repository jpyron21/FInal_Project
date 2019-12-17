# Using Sentinel-1 Synthetic Aperture Radar (SAR) to Evaluate Flooding in Nassau, Bahamas

### Table of Contents
* [Introduction](#Introduction)
* [Objective](#Objective)
* [Methodology](#Methodology)
* [Prerequisites](#Prerequisites)
* [Preprocessing Data](#Preprocessing-Data)
* [Finding Critical Values](#Finding-Critical-Values)
* [Visualizations and Reclassifying Data](#Visualizations-and-Reclassifying-Data)
* [Results and Future Study](#Results-and-Future-Study)
* [Author](#Author)
* [Sources and Acknowledgements](#Sources-and-Acknowledgements)
### Introduction
In the age of data science, there is a shift in geospatial processing from a traditional GIS to R. Raster processing is especially useful in R as it provides a fast way to read data and transform data in a less-complex file format. While this will provide a more complex set of instruction, the attached .Rmd file can be used to test the code.

### Objective
I will be using Sentinel-1 SAR (synthetic aperture radar) data to evaluate flooding in Nassau, Bahamas from Hurricane Dorian.
This is my first time working with SAR and this project will indirectly contribute to my thesis, as I intend to use R and machine learning techniques to identify wetlands. 

### Methodology
In order to identify change I will be analyzing 3 SAR images, one in the month prior to Dorian, one immediately following Dorian, and one a month prior to Dorian. In order to identify areas of flooding by taking the absolute value of the each images and then subtracting them. I will show the change from pre-Dorian to during-Dorian and pre-Dorian to post-Dorian.
It should be noted that the during-Dorian data is actually taken immediately following but has to be distinguished from the post-Dorian imagery, which was taken one month after.

### Prerequisites
For this analysis I used 6 packages and they can be loaded as shown below.
```R
library(raster)
library(rgdal)
library(sf)
library(prettymapr)
library(markdown)
library(ggplot2)
```

### Preprocessing Data
The data is preprocessed primarily in Google Earth Engine, as shown below. In order to stay within the scope of this class I will only briefly explain this process, as I pulled data from Sentinel-1 with single VV polarization.
```JavaScript
var preDorian = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(geometry)
  .filterDate('2019-07-22', '2019-07-28')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))

var preDorRed = preDorian.reduce(ee.Reducer.median());

var dorian = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(geometry) 
  .filterDate('2019-08-24', '2019-08-28')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  
var dorianRed = dorian.reduce(ee.Reducer.median());

var postDorian = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(geometry)
  .filterDate('2019-09-22', '2019-09-28')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))

var postDorRed = postDorian.reduce(ee.Reducer.median())

Export.image.toDrive({
  image:preDorRed.toDouble(),
  description:'preDorian',
  scale:30,
  region:geometry
});

Export.image.toDrive({
  image:dorianRed.toDouble(),
  description:'Dorian',
  scale:30,
  region:geometry
});

Export.image.toDrive({
  image:postDorRed.toDouble(),
  description:'postDorian',
  scale:30,
  region:geometry
});
```

This data is sent to my drive and, once downloaded, can be read into R. Be mindful to set the working directory at the start of the project.
```R
pre <- raster("preDorian.tif")
during <- raster("Dorian.tif")
post <- raster("postDorian.tif")
nassau <- st_read("new_providence.shp")
```

Since the goal is to analyze floding and the study area is an island, it is important to apply a mask as to not include the ocean in the analysis.
```R
pre_masked <- mask(x = pre, mask = nassau)
during_masked <- mask(x = during, mask = nassau)
post_masked <- mask(x = post, mask = nassau)
```

Now it's time to reproject the data to a projected coordinate system. Typically this should be done first, but since all data had the same coordinate reference system, the order does not matter. This will be important for calculating area.
```R
new_proj <- "+proj=utm +zone=18 +datum=WGS84 +units=m +no_defs +ellps=WGS84 +towgs84=0,0,0"
pre_masked <- projectRaster(pre_masked, crs=new_proj)
during_masked <- projectRaster(during_masked, crs=new_proj)
post_masked <- projectRaster(post_masked, crs=new_proj)
nassau <- spTransform(nassau, crs(new_proj))
```

In order to find areas of difference, subtract the absolute value of each raster.
```R
during_diff <- abs(during_masked) - abs(pre_masked)
post_diff <- abs(post_masked)  - abs(pre_masked)
```

Next it is important to change areas of no change, or 0, to NA. Before doing that, archive these masked images. We will be manipulating them a lot in the next few steps and we will need to come back to this original masked image.
```R
during_diff_arch <- during_diff
post_diff_arch <- post_diff
during_diff[during_diff == 0] <- NA
post_diff[post_diff == 0] <- NA
```

### Finding Critical Values
To find areas of inundation you must first find some important values from the images. To do this you can use cellStats from the raster package. For each difference image, during and post, calculate the mean, standard deviation, minimum, and maximum using cellStats.
```R
during_mean <- cellStats(during_diff, mean)
post_mean <- cellStats(post_diff, mean)
during_sd <- cellStats(during_diff, sd)
post_sd <- cellStats(post_diff, sd)
during_min <- cellStats(during_diff_arch, min)
post_min <- cellStats(post_diff_arch, min)
during_max <- cellStats(during_diff_arch, max)
post_max <- cellStats(post_diff_arch, max)
```

Next we need to calculate the thresholds for inundated areas. These thresholds occur differently in areas that are and are not highly vegetated, so we will be accounting for both.
```R
during_inun_thresh <- during_mean - 1.5*during_sd
post_inun_thresh <- post_mean - 1.5*post_sd

during_veg_thresh <- during_mean + 2.5*during_sd
post_veg_thresh <- post_mean + 2.5*post_sd
```

### Visualizations and Reclassifying Data

Finally, it is time to prep the data for reclassification. This will allow us to see areas of inundation following Hurricane Dorian. First, values are appended to a vector with the minimum class break, maximum class break, and output class value. These are referred to as bins. Seeing as there are three bins, the vector is transformed into a 3x3 matrix.
```R
reclass_during <- c(-Inf, during_inun_thresh, 1,
                    during_inun_thresh, during_veg_thresh, NA,
                    during_veg_thresh, Inf, 1)
reclass_during_m <- matrix(reclass_during, ncol = 3, byrow = TRUE)

reclass_post <- c(-Inf, post_inun_thresh, 1,
                  post_inun_thresh, post_veg_thresh, NA,
                  post_veg_thresh, Inf, 1)
reclass_post_m <- matrix(reclass_post, ncol = 3, byrow = TRUE)
```

To obtain the final reclassified raster, use the reclassify tool, taking the parameters of the archived raster image and the reclassification matrix.
```R
during_classified <- reclassify(during_diff_arch, reclass_during_m)
post_classified <- reclassify(post_diff_arch, reclass_post_m)
```

Now it is time to visualize our images with the plot() function. Since both images should be symbolized in the same manner, a function is used to ensure that there is as little unnecessary variability as possible. Notably, a legend is added to the bottom right of the plot and the nassau shapefile is added in order to show the shape of the island.

```R
plot_images <- function(classified, title){
  plot(classified,
     legend = FALSE,
     col = c("blue"),
     axes = FALSE,
     main = title)

plot(nassau[0],
     add = TRUE)

legend("bottomright",
       legend = c("Flooded Areas"),
       fill = c("blue"),
       border = FALSE)

prettymapr::addnortharrow(scale = 0.6, text.col = 'black', cols = c('black', 'black'))
prettymapr::addscalebar(plotunit = 'm', widthhint = 0.25, lwd = 1, padin = c(0.15, 0.08), style = "bar", label.cex = 1.0)
}
```

To create maps of the analysis, we can initialize the title for each map and call the plot_images() function with the data and title as parameters.
```R
title = "Flooding in Nassau Immeidately Following Hurrican Dorian"
plot_images(during_classified)

title = "Flooding in Nassau One Month Following Hurrican Dorian"
plot_images(post_classified)
```

It is important to obtain qualitative values from the data. I used ggplot2 to create a simple bargraph of percent inundation to visualize these qualitative values. First, I had to calculate the area of the total landmass and the inundated areas. I set them up in a data.frame to be more compatible with ggplot.
```R
during_area <- (sum(!is.na(during_classified[])) * xres(during_classified) * xres(during_classified)) / 1000000
post_area <- (sum(!is.na(post_classified[])) * xres(post_classified) * xres(during_classified)) / 1000000
total_area <- nrow(pre_masked) * ncol(pre_masked) * xres(pre_masked) * xres(pre_masked) / 1000000

df <- data.frame(
  groups = c("One Day", "One Month"),
  inundated = c(round((during_area/total_area)*100, digits=2), round((post_area/total_area)*100, digits=2))
  )
 ```
 
 Finally, I created my bargraph of percent inundation vs time elapsed since Dorian. 
 ```R
 bp <- ggplot(data=df, aes(x=groups, y=inundated)) +
  geom_bar(stat="identity", width = 0.6, fill="skyblue4")+
  geom_text(aes(label=inundated), vjust=1.6, color="white", size=5)+
  ggtitle("Flooding Prior to Hurricane Dorian")+
  labs(x = "Time Elapsed", y = "Percent Inundated (%)")
bp
```

### Results and Future Study
The objectives for this project were to test my R skills as pertaining to the scope of GEOG 693 and familiarize myself with the data. For future studies I intend to examine other single and dual polarizations, as the results from my singular 'VV' polarization analysis suggests that there are better modes for evaluating flooding in the realm of SAR data.

### Author
Work by Jaimee Pyron of West Virginia University

### Sources and Acknowledgements
I would like to acknowledge Dr. Amy Hessl for instructing the course the course that made this project possible, my advisor Dr. Maxwell for inspiring the project, and Hartford Johnson, a good ol' pal who showed me how to make the highly functional table of contents.
