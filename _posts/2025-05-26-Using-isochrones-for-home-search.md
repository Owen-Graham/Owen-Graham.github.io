---
layout: post
title: "Isochrones for home location selection"
date: 2025-05-26
---

# Building Multi-Criteria Location Analysis with Open Source Tools

Finding the perfect place to live involves balancing multiple factors: proximity to work, schools, healthcare, and amenities. Instead of manually checking each location, I built an automated system to map areas that satisfy multiple criteria simultaneously using free, open-source tools.

## The Tool Stack

**OpenRouteService API** served as our routing engine, generating isochrones—map areas reachable within specific time limits using different transport modes (walking, cycling, driving). With a free tier of 2,000 requests daily, it's accessible for personal projects.

**Overpass API** provided the location data by querying OpenStreetMap's extensive database. We used it to find coordinates for supermarkets, schools, hospitals, and other amenities across entire countries. The query language is powerful enough to distinguish between Woolworths and Coles stores, or find specific facility types like athletics tracks.

**Python libraries** tied everything together: Folium created interactive maps with layer controls, Shapely handled geometric operations for finding intersections between different isochrones, and standard libraries managed API calls and data processing.

## The Process

The system works by: (1) querying Overpass API to find all relevant locations (e.g., "all hospitals in Australia"), (2) creating isochrones around each location using OpenRouteService, (3) finding geometric intersections where multiple criteria overlap, and (4) generating interactive maps with toggleable layers.

For example, finding areas within 1-hour drive of both a Woolworths and a school produces a map showing only regions satisfying both criteria. Users can toggle different combinations to see areas meeting 1, 2, or all specified requirements.

## Limitations and Alternatives

We discovered that approximating public transport using driving isochrones creates unrealistic circular coverage patterns, since rail networks follow linear corridors. For accurate transit analysis, alternatives include GTFS data with OpenTripPlanner (complex but accurate), Google Maps API (simple but costly at scale), or building custom graph algorithms following actual rail networks.

The open-source approach proves viable for continental-scale analysis while remaining accessible to individual researchers and decision-makers.

# Multi-Criteria Location Analysis Setup Guide

Follow this guide to set up your own location analysis system for finding areas that meet multiple criteria (e.g., close to schools AND hospitals AND supermarkets).

## Prerequisites

- Python 3.7+ installed
- Jupyter Notebook or Google Colab
- Basic familiarity with Python

## Step 1: Get Your OpenRouteService API Key

1. **Visit**: [openrouteservice.org](https://openrouteservice.org)
2. **Sign up** for a free account
3. **Verify your email** (important - API won't work without this)
4. **Go to Dashboard** → **Request a Token**
5. **Copy your API key** - it looks like: `5b3ce3597851110001cf6248...`

**Free tier limits**: 2,000 requests/day, 40 requests/minute

## Step 2: Install Required Libraries

Run this in your Jupyter notebook or terminal:

```bash
pip install folium requests shapely geopandas pandas numpy
```

## Step 3: Test Your Setup

Create a new notebook and run this test code:

```python
# Multi-Criteria Location Analysis - Setup Test
import requests
import folium
import json
from shapely.geometry import Polygon, Point
from shapely.ops import unary_union
import pandas as pd
import numpy as np

# Enter your API key here
ORS_API_KEY = "YOUR_API_KEY_HERE"  # Replace with your actual key

def test_ors_connection(api_key):
    """Test if OpenRouteService API key works"""
    
    headers = {
        'Authorization': api_key,
        'Content-Type': 'application/json'
    }
    
    # Simple test - 10 min walk from Sydney Opera House
    test_data = {
        'locations': [[151.2153, -33.8568]], # [lon, lat]
        'range': [600], # 10 minutes in seconds
        'range_type': 'time'
    }
    
    url = "https://api.openrouteservice.org/v2/isochrones/foot-walking"
    
    try:
        response = requests.post(url, json=test_data, headers=headers)
        if response.status_code == 200:
            print("✅ OpenRouteService API: Connected successfully")
            return True
        else:
            print(f"❌ API Error {response.status_code}: {response.text}")
            return False
    except Exception as e:
        print(f"❌ Connection Error: {e}")
        return False

def test_overpass_connection():
    """Test if Overpass API works"""
    
    # Find supermarkets around Sydney Opera House
    query = '''
    [out:json][timeout:25];
    (
      node["shop"="supermarket"](around:2000,-33.8568,151.2153);
    );
    out;
    '''
    
    try:
        response = requests.post("https://overpass-api.de/api/interpreter", data=query)
        if response.status_code == 200:
            data = response.json()
            count = len(data.get('elements', []))
            print(f"✅ Overpass API: Found {count} supermarkets")
            return True
        else:
            print(f"❌ Overpass Error: {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

# Run tests
print("🧪 Testing setup...")
test_ors_connection(ORS_API_KEY)
test_overpass_connection()

# Create test map
test_map = folium.Map(location=[-33.8568, 151.2153], zoom_start=12)
folium.Marker([-33.8568, 151.2153], popup="Sydney Opera House").add_to(test_map)
print("✅ Map creation working")
test_map  # This will display the map
```

## Step 4: Copy the Main Analysis Code

Once your setup test works, copy this complete analysis system:

```python
# Complete Multi-Criteria Location Analysis System
import requests
import folium
from shapely.geometry import Polygon, MultiPolygon
from shapely.ops import unary_union
import time
from itertools import combinations

class LocationAnalyzer:
    def __init__(self, ors_api_key):
        self.ors_api_key = ors_api_key
        self.overpass_url = "https://overpass-api.de/api/interpreter"
        self.ors_base_url = "https://api.openrouteservice.org/v2/isochrones"
        
        self.ors_headers = {
            'Accept': 'application/json',
            'Authorization': self.ors_api_key,
            'Content-Type': 'application/json'
        }
        
        self.transport_modes = {
            'walking': 'foot-walking',
            'cycling': 'cycling-regular',
            'driving': 'driving-car'
        }
        
        self.places_data = {}
        self.isochrone_data = {}
    
    def find_places(self, place_type, bounds, custom_query=None):
        """Find places using Overpass API"""
        
        bbox_str = f"{bounds[0]},{bounds[1]},{bounds[2]},{bounds[3]}"
        
        queries = {
            'supermarket': f'''
                [out:json][timeout:60][bbox:{bbox_str}];
                (node["shop"="supermarket"];way["shop"="supermarket"];);
                out center;
            ''',
            'hospital': f'''
                [out:json][timeout:60][bbox:{bbox_str}];
                (node["amenity"="hospital"];way["amenity"="hospital"];);
                out center;
            ''',
            'school': f'''
                [out:json][timeout:60][bbox:{bbox_str}];
                (node["amenity"="school"];way["amenity"="school"];);
                out center;
            '''
        }
        
        query = custom_query or queries.get(place_type)
        if not query:
            print(f"No query for {place_type}")
            return []
        
        try:
            response = requests.post(self.overpass_url, data=query, timeout=120)
            response.raise_for_status()
            
            data = response.json()
            elements = data.get('elements', [])
            
            places = []
            for element in elements:
                if element['type'] == 'node':
                    lat, lon = element['lat'], element['lon']
                elif 'center' in element:
                    lat, lon = element['center']['lat'], element['center']['lon']
                else:
                    continue
                
                places.append({
                    'lat': lat, 'lon': lon,
                    'name': element.get('tags', {}).get('name', f'Unnamed {place_type}'),
                    'place_type': place_type
                })
            
            print(f"Found {len(places)} {place_type} locations")
            self.places_data[place_type] = places
            return places
            
        except Exception as e:
            print(f"Error finding {place_type}: {e}")
            return []
    
    def create_isochrones(self, place_type, transport_mode, time_minutes):
        """Create isochrones around all places of a type"""
        
        if place_type not in self.places_data:
            print(f"No data for {place_type}")
            return []
        
        places = self.places_data[place_type]
        ors_profile = self.transport_modes.get(transport_mode, 'foot-walking')
        isochrones = []
        
        print(f"Creating {transport_mode} isochrones ({time_minutes}min) for {len(places)} {place_type}s...")
        
        # Process in batches (ORS allows up to 5 locations per request)
        batch_size = 5
        for i in range(0, len(places), batch_size):
            batch = places[i:i + batch_size]
            locations = [[place['lon'], place['lat']] for place in batch]
            
            body = {
                "locations": locations,
                "range": [time_minutes * 60],
                "range_type": "time"
            }
            
            url = f"{self.ors_base_url}/{ors_profile}"
            
            try:
                response = requests.post(url, json=body, headers=self.ors_headers)
                if response.status_code == 200:
                    data = response.json()
                    for j, feature in enumerate(data.get('features', [])):
                        if 'geometry' in feature:
                            coords = feature['geometry']['coordinates'][0]
                            polygon_coords = [(coord[0], coord[1]) for coord in coords]
                            polygon = Polygon(polygon_coords)
                            isochrones.append({
                                'polygon': polygon,
                                'place': batch[j],
                                'transport_mode': transport_mode,
                                'time_minutes': time_minutes
                            })
                time.sleep(0.5)  # Rate limiting
            except Exception as e:
                print(f"Error creating isochrones: {e}")
        
        criteria_key = f"{place_type}_{transport_mode}_{time_minutes}min"
        self.isochrone_data[criteria_key] = isochrones
        print(f"Created {len(isochrones)} isochrones for {place_type}")
        return isochrones
    
    def find_intersections(self, criteria_keys):
        """Find areas satisfying multiple criteria"""
        
        criterion_unions = {}
        for criterion in criteria_keys:
            if criterion in self.isochrone_data:
                polygons = [iso['polygon'] for iso in self.isochrone_data[criterion]]
                if polygons:
                    union_poly = unary_union(polygons)
                    criterion_unions[criterion] = union_poly
        
        results = {}
        
        # Single criteria
        for criterion in criterion_unions:
            results[f"1_criterion_{criterion}"] = criterion_unions[criterion]
        
        # Multiple criteria combinations
        for r in range(2, len(criterion_unions) + 1):
            for combo in combinations(criterion_unions.keys(), r):
                intersection = criterion_unions[combo[0]]
                for criterion in combo[1:]:
                    intersection = intersection.intersection(criterion_unions[criterion])
                
                if not intersection.is_empty:
                    combo_key = f"{r}_criteria_" + "_".join(combo)
                    results[combo_key] = intersection
        
        return results
    
    def create_map(self, results, center_coords, zoom_start=10):
        """Create interactive map with results"""
        
        m = folium.Map(location=center_coords, zoom_start=zoom_start)
        
        colors = {1: '#FEE5D9', 2: '#FCAE91', 3: '#FB6A4A', 4: '#DE2D26', 5: '#A50F15'}
        
        for result_key, geometry in results.items():
            if geometry.is_empty:
                continue
            
            num_criteria = int(result_key.split('_')[0])
            color = colors.get(num_criteria, '#000000')
            
            layer_name = result_key.replace('_', ' ').title()
            feature_group = folium.FeatureGroup(name=layer_name)
            
            if isinstance(geometry, MultiPolygon):
                polygons = list(geometry.geoms)
            else:
                polygons = [geometry]
            
            for poly in polygons:
                if hasattr(poly, 'exterior'):
                    coords = list(poly.exterior.coords)
                    folium_coords = [[coord[1], coord[0]] for coord in coords]
                    
                    folium.Polygon(
                        locations=folium_coords,
                        color=color,
                        fillColor=color,
                        fillOpacity=0.6,
                        weight=2,
                        popup=layer_name
                    ).add_to(feature_group)
            
            feature_group.add_to(m)
        
        folium.LayerControl().add_to(m)
        return m

# Analysis function
def analyze_location_criteria(criteria_list, bounds, ors_api_key):
    """
    Main analysis function
    
    criteria_list format:
    [
        {'place_type': 'supermarket', 'transport_mode': 'walking', 'time_minutes': 10},
        {'place_type': 'hospital', 'transport_mode': 'driving', 'time_minutes': 15},
        {'place_type': 'school', 'transport_mode': 'walking', 'time_minutes': 15}
    ]
    
    bounds format: [south, west, north, east]
    """
    
    analyzer = LocationAnalyzer(ors_api_key)
    criteria_keys = []
    
    for criterion in criteria_list:
        print(f"\nProcessing: {criterion['place_type']}")
        
        # Find places
        places = analyzer.find_places(
            criterion['place_type'], 
            bounds,
            criterion.get('custom_query')
        )
        
        if places:
            # Create isochrones
            isochrones = analyzer.create_isochrones(
                criterion['place_type'],
                criterion['transport_mode'],
                criterion['time_minutes']
            )
            
            if isochrones:
                criteria_key = f"{criterion['place_type']}_{criterion['transport_mode']}_{criterion['time_minutes']}min"
                criteria_keys.append(criteria_key)
    
    # Find intersections
    if criteria_keys:
        results = analyzer.find_intersections(criteria_keys)
        
        # Create map
        center_lat = (bounds[0] + bounds[2]) / 2
        center_lon = (bounds[1] + bounds[3]) / 2
        return analyzer.create_map(results, [center_lat, center_lon])
    
    return None
```

## Step 5: Example Usage

```python
# Example: Find areas near supermarkets, hospitals, and schools
API_KEY = "your_actual_api_key_here"

# Define your criteria
criteria = [
    {
        'place_type': 'supermarket',
        'transport_mode': 'walking',
        'time_minutes': 10
    },
    {
        'place_type': 'hospital', 
        'transport_mode': 'driving',
        'time_minutes': 15
    },
    {
        'place_type': 'school',
        'transport_mode': 'walking',
        'time_minutes': 15
    }
]

# Define your area of interest [south, west, north, east]
# Example: Melbourne area
melbourne_bounds = [-37.9, 144.8, -37.7, 145.1]

# Run the analysis
result_map = analyze_location_criteria(criteria, melbourne_bounds, API_KEY)
result_map  # This will display your interactive map!
```

## Understanding Your Results

The map will show different colored areas:
- **Light colors**: Areas meeting 1 criterion
- **Medium colors**: Areas meeting 2 criteria  
- **Dark colors**: Areas meeting all criteria

Use the layer control (top right) to toggle different combinations on/off.

## Troubleshooting

**Common Issues:**

1. **"401 Unauthorized"**: Check your API key is correct and account is verified
2. **"No locations found"**: Try larger bounds or different place types
3. **Map not displaying**: Make sure you're running in Jupyter notebook
4. **Slow performance**: Reduce area size or number of locations

**Getting Help:**
- Check the [OpenRouteService documentation](https://openrouteservice-py.readthedocs.io/)
- Browse [Overpass API examples](https://wiki.openstreetmap.org/wiki/Overpass_API/Overpass_QL)
- Ask questions in the OpenStreetMap community forums

## Next Steps

Once you have the basic system working:
- Customize place types for your needs
- Adjust time limits and transport modes
- Try different geographic areas
- Export results for further analysis
- Integrate with real estate data for house hunting

Happy mapping! 🗺️
