Surface Water Extent Analysis
This repository contains Python scripts for analyzing and visualizing surface water extent using Google Earth Engine (GEE) and geemap. The provided code performs tasks such as loading GeoJSON files, calculating water extent time series, plotting time series, and visualizing water extent changes over time.

Table of Contents
Installation
Usage
Functions
load_geojson
get_surface_water_extent
plot_time_series
visualize_water_extent
visualize_water_extent_by_year
Example
License
Installation
Clone the repository:

bash
Copy code
git clone https://github.com/yourusername/surface-water-extent-analysis.git
cd surface-water-extent-analysis
Create a virtual environment and activate it:

bash
Copy code
python3 -m venv venv
source venv/bin/activate
Install the required packages:

bash
Copy code
pip install -r requirements.txt
Authenticate with Google Earth Engine:

python
Copy code
import ee
ee.Authenticate()
ee.Initialize()
Usage
The scripts include functions for loading GeoJSON files, calculating water extent time series, plotting time series, and visualizing water extent changes by year.

Functions
load_geojson
Load a GeoJSON file and return the geometry.

python
Copy code
def load_geojson(geojson_file):
    """Load a GeoJSON file and return the geometry."""
    with open(geojson_file) as f:
        geojson = json.load(f)
    return ee.Geometry(geojson['features'][0]['geometry'])
get_surface_water_extent
Calculate surface water extent time series for the given geometry and time range.

python
Copy code
def get_surface_water_extent(geometry, start_date, end_date):
    """Calculate surface water extent time series for the given geometry and time range."""
    dataset = ee.ImageCollection('JRC/GSW1_4/YearlyHistory')
    filtered_dataset = dataset.filterDate(start_date, end_date)

    def calculate_water_extent(image):
        water = image.select('waterClass').eq(2)
        water_area = water.multiply(ee.Image.pixelArea()).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=geometry,
            scale=30,
            maxPixels=1e10
        )
        date = ee.Date(image.get('system:time_start')).format('YYYY-MM-dd')
        return ee.Feature(None, {'date': date, 'water_area': water_area.get('waterClass')})

    water_extent_series = filtered_dataset.map(calculate_water_extent).getInfo()
    time_series = [{'date': f['properties']['date'], 'water_area': f['properties']['water_area']} for f in water_extent_series['features']]
    return pd.DataFrame(time_series)
plot_time_series
Plot the surface water extent time series.

python
Copy code
def plot_time_series(df):
    """Plot the surface water extent time series."""
    df['date'] = pd.to_datetime(df['date'])
    df.set_index('date', inplace=True)
    df.plot(kind='line', y='water_area', legend=False)
    plt.title('Surface Water Extent Time Series')
    plt.xlabel('Date')
    plt.ylabel('Water Area (square meters)')
    plt.show()
visualize_water_extent
Visualize water extent using false color composite maps.

python
Copy code
def visualize_water_extent(geometry, start_date, end_date):
    """Visualize water extent using false color composite maps."""
    dataset = ee.ImageCollection('JRC/GSW1_4/MonthlyHistory')
    filtered_dataset = dataset.filterDate(start_date, end_date)
    median_image = filtered_dataset.median().clip(geometry)

    vis_params = {
        'bands': ['water'],
        'min': 0,
        'max': 2,
        'palette': ['ffffff', '0000ff']
    }

    Map = geemap.Map()
    Map.centerObject(geometry, 10)
    Map.addLayer(median_image, vis_params, 'Water Extent')
    Map.addLayer(geometry, {'color': 'blue'}, 'Geometry')
    Map.addLayerControl()

    return Map
visualize_water_extent_by_year
Visualize water extent changes by year using an animation.

python
Copy code
def visualize_water_extent_by_year(geometry, start_date, end_date, out_gif):
    """Visualize water extent changes by year using an animation."""
    dataset = ee.ImageCollection('JRC/GSW1_4/MonthlyHistory')
    filtered_dataset = dataset.filterDate(start_date, end_date)

    vis_params = {
        'bands': ['water'],
        'min': 0,
        'max': 2,
        'palette': ['ffffff', '0000ff']
    }

    Map = geemap.Map()
    Map.centerObject(geometry, 10)
    Map.addLayer(geometry, {'color': 'blue'}, 'Geometry')

    annual_images = []

    for year in range(int(start_date[:4]), int(end_date[:4]) + 1):
        year_start = str(year) + '-01-01'
        year_end = str(year) + '-12-31'
        year_filtered_dataset = filtered_dataset.filterDate(year_start, year_end).mean().clip(geometry)
        annual_images.append(year_filtered_dataset.set('year', year))

    annual_collection = ee.ImageCollection.fromImages(annual_images)

    video_args = {
        'dimensions': 768,
        'region': geometry,
        'framesPerSecond': 1,
        'min': 0,
        'max': 2,
        'palette': ['ffffff', '0000ff'],
        'title': 'Year: {year}',
        'fontSize': 30,
        'fontColor': 'white',
        'fontOutlineColor': 'black'
    }

    out_gif = out_gif if out_gif.endswith('.gif') else out_gif + '.gif'
    geemap.download_ee_video(annual_collection, video_args, out_gif)

    for year_image in annual_images:
        year = year_image.get('year').getInfo()
        Map.addLayer(year_image, vis_params, f'Water Extent {year}')

    return Map
Example
python
Copy code
import ee
import geemap
import json
import pandas as pd
import matplotlib.pyplot as plt

# Authenticate and initialize the Earth Engine API.
ee.Authenticate()
ee.Initialize()

# Load GeoJSON file
geojson_file = '/content/Surface_waterBody.geojson'
geometry = load_geojson(geojson_file)

# Define date range
start_date = '2000-01-01'
end_date = '2021-12-31'

# Get the surface water extent time series
df = get_surface_water_extent(geometry, start_date, end_date)

# Plot the time series
plot_time_series(df)

# Visualize the water extent
Map = visualize_water_extent(geometry, start_date, end_date)
Map

# Visualize the water extent changes for each year over the specified time period and export animation
out_gif = 'water_extent_animation.gif'
Map = visualize_water_extent_by_year(geometry, start_date, end_date, out_gif)
Map
