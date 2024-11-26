# Climate_Indices
Extract the climate indices such as: MNDWI, NDWI, NDBI, NDSI, Precipitation, Land Surface Temperature (LST) using Landsat-8 for 10 years
-----
This code performs analysis on Landsat-8 satellite imagery and CHIRPS precipitation data to compute and visualize various indices, precipitation, and land surface temperature (LST).

---

### **1. Center the map view**
```javascript
Map.centerObject(roi);
```
- Focuses the map view on the region of interest (ROI). The `roi` is a predefined geometry (e.g., a polygon or point) that specifies the study area.

---

### **2. Filter Landsat-8 imagery**
```javascript
var landsat = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
              .filterBounds(roi)
              .filterDate('2013-04-01', '2013-05-01')
              .filter(ee.Filter.lessThan('CLOUD_COVER', 20));
```
- Loads the Landsat 8 Surface Reflectance Tier 1 Level 2 dataset.
- Filters images to include only those overlapping with `roi`.
- Restricts the time range to April 1, 2013, to May 1, 2013.
- Filters images with less than 20% cloud cover.

---

### **3. Define functions to calculate indices**
#### **a. MNDWI (Modified Normalized Difference Water Index)**
```javascript
var calculateMNDWI = function(image) {  
  var mndwi = image.normalizedDifference(['SR_B3', 'SR_B6']).rename('MNDWI');  
  return image.addBands(mndwi);  
};
```
- Calculates MNDWI using the green band (`SR_B3`) and SWIR1 band (`SR_B6`).
- Adds the MNDWI as a new band to the image.

#### **b. NDVI (Normalized Difference Vegetation Index)**
```javascript
var calculateNDVI = function(image) {  
  var ndvi = image.normalizedDifference(['SR_B5', 'SR_B4']).rename('NDVI');  
  return image.addBands(ndvi);  
};
```
- Calculates NDVI using the NIR band (`SR_B5`) and red band (`SR_B4`).
- Adds the NDVI as a new band to the image.

#### **c. NDBI (Normalized Difference Built-up Index)**
```javascript
var calculateNDBI = function(image) {  
  var ndbi = image.normalizedDifference(['SR_B5', 'SR_B6']).rename('NDBI');  
  return image.addBands(ndbi);  
};
```
- Calculates NDBI using the NIR band (`SR_B5`) and SWIR1 band (`SR_B6`).
- Adds the NDBI as a new band to the image.

#### **d. NDSI (Normalized Difference Snow Index)**
```javascript
var calculateNDSI = function(image) {  
  var ndsi = image.normalizedDifference(['SR_B3', 'SR_B5']).rename('NDSI');  
  return image.addBands(ndsi);  
};
```
- Calculates NDSI using the green band (`SR_B3`) and NIR band (`SR_B5`).
- Adds the NDSI as a new band to the image.

---

### **4. Apply the index functions**
```javascript
var landsatWithIndices = landsat.map(calculateMNDWI)
                                 .map(calculateNDVI)
                                 .map(calculateNDBI)
                                 .map(calculateNDSI);
```
- Applies the four index calculation functions to each image in the Landsat collection, adding MNDWI, NDVI, NDBI, and NDSI bands.

---

### **5. Extract mean values for each index**
```javascript
var extractMeanIndices = function(image) {  
  var date = image.date().format('YYYY-MM-dd');  
  var meanMNDWI = image.select('MNDWI').reduceRegion({  
    reducer: ee.Reducer.mean(),  
    geometry: roi,  
    scale: 30,  
    maxPixels: 1e13  
  }).get('MNDWI');  
  ...
  return ee.Feature(null, {  
    date: date,  
    MNDWI: meanMNDWI,  
    NDVI: meanNDVI,  
    NDBI: meanNDBI,  
    NDSI: meanNDSI  
  });  
};
```
- Computes the mean values of MNDWI, NDVI, NDBI, and NDSI for the `roi` using the `reduceRegion` function.
- Creates a feature with the mean values and the acquisition date of the image.

---

### **6. Create a chart for indices**
```javascript
var landsatchart = ui.Chart.feature.byFeature(uniqueMeanIndices, 'date', ['MNDWI', 'NDVI', 'NDBI', 'NDSI']);
```
- Plots a line chart showing the time series of the indices (MNDWI, NDVI, NDBI, NDSI) over time.

---

### **7. Load CHIRPS precipitation data**

It is essential that the dates of indices and precipitation match.

```javascript
var CHIRPS = ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY")
              .filterBounds(roi)
              .filterDate('2013-01-01', '2023-12-31');
```
- Loads the daily CHIRPS precipitation dataset.
- Filters data to the `roi` and time range (2013–2023).

---

### **7-a. Extract NDVI and precipitation**
```javascript
var extractNDVIandPrecipitation = function(landImage) {  
  var date = landImage.date();  
  var ndvi = landImage.select('NDVI').reduceRegion({...}).get('NDVI');  
  var precipitation = CHIRPS.filterDate(date, date.advance(1, 'day')).mean()
                             .reduceRegion({...}).get('precipitation');  
  return ee.Feature(null, {date: date.format('YYYY-MM-dd'), NDVI: ndvi, Precipitation: precipitation});  
};
```
- Extracts the mean NDVI from Landsat images.
- Filters CHIRPS data to match the Landsat image date and calculates mean daily precipitation.
- Combines NDVI and precipitation into a single feature.

---

### **7-b. Create a precipitation chart**
```javascript
var precipitationChart = ui.Chart.feature.byFeature(precipitationFeatureCollection, 'date', ['Precipitation']);
```
- Plots a line chart showing precipitation over time.

---

### **8. Calculate Land Surface Temperature (LST)**
#### **a. Compute LST**
```javascript
function calculateLST(image) {  
  var thermalBand = image.select('ST_B10');  
  var brightnessTemp = thermalBand.multiply(0.00341802).add(149.0).subtract(273.15);  
  var emissivity = 0.95;  
  var lst = brightnessTemp.divide(emissivity);  
  return image.addBands(lst.rename('LST'));  
}
```
- Extracts the thermal band (`ST_B10`) from Landsat images.
- Converts the thermal band to brightness temperature in Celsius.
- Divides brightness temperature by emissivity (assumed to be 0.95) to compute LST.

#### **b. Extract mean LST**
```javascript
function extractMeanLST(image) {  
  var date = image.date().format('YYYY-MM-dd');  
  var meanLST = image.select('LST').reduceRegion({...}).get('LST');  
  return ee.Feature(null, {date: date, meanLST: meanLST});  
}
```
- Computes the mean LST for the `roi` and creates a feature with the date and mean LST.

---

### **c. Create an LST chart**
```javascript
var landsatlst = ui.Chart.feature.byFeature(uniqueMeanLST, 'date', 'meanLST');
```
- Plots a line chart showing LST over time.

---

### **9. Display charts**
```javascript
print(landsatchart); 
print(landsatlst);
print(precipitationChart);
```
- Displays the three charts (indices, LST, and precipitation) in the GEE interface.
