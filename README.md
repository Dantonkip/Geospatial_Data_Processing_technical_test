# Geospatial Data Processing

## Overview

This task utilizes Google Earth Engine (GEE) to process and analyze geospatial datasets, including Sentinel-1, Sentinel-2, MODIS Land Surface Temperature (LST), and SRTM Digital Elevation Model (DEM). The workflow involves:

1. Extracting satellite data within a specified area of interest (AOI)

2. Preprocessing and filtering the datasets

3. Computing vegetation indices (NDVI)

4. Resampling datasets to a uniform spatial resolution

5. Stacking multiple geospatial layers into a single datacube

6. Exporting the final datacube to Google Drive for further analysis

# Data cube data link
The following link [Data cube](https://drive.google.com/file/d/1YG2NDaMcD8ydjWrWnlCiFvRvqxyjKyPG/view?usp=drive_link) is a link to datacube data from the drive.

## Prerequisites

To run this project, you need:

1. A Google Earth Engine (GEE) account
2. Google Cloud project access
3. Google colab since it does nor require installation of libraries since they are already installed.


## Getting Started

### 1. Initialize Google Earth Engine

Authenticate and initialize GEE:

import ee
import geemap

#### Initialize GEE with project ID
ee.Initialize(project="ee-dantonkipngeno1")

### 2. Define Area of Interest (AOI)

Specify the geographic boundary for analysis:

aoi = ee.Geometry.Polygon([
    [
        [36.3068725, -0.5952726],
        [36.3068725, -0.7423752],
        [36.4520684, -0.7423752],
        [36.4520684, -0.5952726],
        [36.3068725, -0.5952726]
    ]
])

### 3. Set Time Range

Select data from the last three months:

start_date = ee.Date.fromYMD(2024, 11, 25)
end_date = ee.Date.fromYMD(2025, 2, 25)

### 4. Load and Process Satellite Datasets

Sentinel-1 (Radar Data)

Filter by AOI, date, and polarization mode:

sentinel1 = (ee.ImageCollection("COPERNICUS/S1_GRD")
    .filterBounds(aoi)
    .filterDate(start_date, end_date)
    .filter(ee.Filter.listContains("transmitterReceiverPolarisation", "VV"))
    .filter(ee.Filter.listContains("transmitterReceiverPolarisation", "VH"))
    .filter(ee.Filter.eq("instrumentMode", "IW"))
    .mean()
    .clip(aoi))

Sentinel-2 (Optical Data & NDVI Calculation)

Extract cloud-free optical imagery and compute NDVI:

sentinel2 = (ee.ImageCollection("COPERNICUS/S2_SR")
    .filterBounds(aoi)
    .filterDate(start_date, end_date)
    .filter(ee.Filter.lt("CLOUDY_PIXEL_PERCENTAGE", 10))
    .median()
    .clip(aoi))

ndvi = sentinel2.normalizedDifference(["B8", "B4"]).rename("NDVI").toFloat()

MODIS Land Surface Temperature (LST)

Extract and convert MODIS LST data:

modis_lst = (ee.ImageCollection("MODIS/061/MOD11A2")
    .filterBounds(aoi)
    .filterDate(start_date, end_date)
    .select("LST_Day_1km")
    .map(lambda img: img.multiply(0.02).subtract(273.15).toFloat()))

Retrieve the most recent LST image:

latest_lst = modis_lst.sort("system:time_start", False).first().clip(aoi)

SRTM Digital Elevation Model (DEM)

Load and clip SRTM DEM data:

srtm = ee.Image("USGS/SRTMGL1_003").clip(aoi)

### 5. Data Visualization

Display the datasets interactively using geemap:

Map = geemap.Map(center=[-0.65, 36.38], zoom=10)
Map.add_basemap("SATELLITE")
Map.addLayer(ndvi, {"min": 0, "max": 1, "palette": ["red", "yellow", "green"]}, "Sentinel-2 NDVI")
Map.addLayer(latest_lst, {"min": 10, "max": 50, "palette": ["blue", "yellow", "red"]}, "Land Surface Temperature")
Map.addLayer(srtm, {"min": -50, "max": 3000}, "Elevation (SRTM)")
Map

### 6. Resampling to Uniform Resolution

Resample temperature and elevation data to 10m resolution:

resampling_method = "bilinear"
temperature_resampled = latest_lst.resample(resampling_method).reproject(crs="EPSG:4326", scale=10).toFloat()
srtm_resampled = srtm.resample(resampling_method).reproject(crs="EPSG:4326", scale=10).toFloat()

### 7. Creating a Geospatial DataCube

Stack multiple bands into a single datacube:

datacube = sentinel1.select("VV").addBands(sentinel1.select("VH"))
    .addBands(ndvi)
    .addBands(temperature_resampled)
    .addBands(srtm_resampled)

### 8. Exporting Data to Google Drive

Save the datacube for further analysis:

export_task = ee.batch.Export.image.toDrive(
    image=datacube,
    description='Geospatial_Datacube',
    folder='Amini_technical_test',
    fileNamePrefix='Geospatial_Datacube',
    region=aoi,
    scale=10,
    crs='EPSG:4326',
    maxPixels=1e13
)
export_task.start()
print("Exporting datacube to Google Drive")

## Summary

This project leverages Google Earth Engine to:

Extract and preprocess geospatial data from multiple satellite sources

Compute vegetation and temperature indices

Standardize spatial resolution across datasets

Export a geospatial datacube for further analysis