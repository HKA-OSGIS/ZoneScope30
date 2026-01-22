# ZoneScope30

Automated identification and visualization of 30 km/h (Tempo 30 / 20 mph) zones based on OpenStreetMap (OSM) data and PostGIS analysis. The system analyzes road networks and nearby features to determine which road segments are candidates for reduced speed limits according to German traffic regulations.

A Node.js proxy is included to bypass CORS restrictions between the browser and GeoServer.

## License

**Source Code:**  
This project's source code (scripts, SQL, proxy, and frontend) is licensed under the MIT License.

**Data:**  
This project uses OpenStreetMap data © OpenStreetMap contributors. The data is licensed under the Open Database License (ODbL) 1.0. https://www.openstreetmap.org/copyright

**Basemap:**  
Satellite imagery is provided by Esri World Imagery and is used for visualization purposes only.

---

## Core Components

This project integrates several tools for importing, analyzing, and serving geospatial data:

| Component      | Layer        | Purpose                                                 |
| :------------- | :----------- | :------------------------------------------------------ |
| **osm2pgsql**  | Data         | Imports OSM data into PostgreSQL.                       |
| **PostGIS**    | Data         | Stores OSM geometries and performs spatial analysis.    |
| **GeoServer**  | Service      | Exposes analysis results as a WFS.                      |
| **Express.js** | Service      | Provides a CORS-bypassing proxy for WFS requests.       |
| **MapLibre**   | Presentation | Displays the identified Tempo-30 segments on a web map. |

---

## Overview of the Workflow

The analysis workflow consists of six sequential steps executed primarily through SQL queries:

1. **Select eligible roads:** Only primary, secondary, tertiary, and residential roads not already tagged as 30 km/h zones are considered. Living streets are excluded, as they already imply low-speed traffic.

2. **Buffer trigger objects:** Social facilities (schools, kindergartens, care homes), playgrounds, and zebra crossings are buffered by 50 m. Residential buildings are buffered by 5 m (noise protection applies only to primary roads).

3. **Identify affected road segments:** Roads intersecting the buffered trigger objects are flagged as candidates for Tempo 30 zones.

4. **Extend zones:** Candidate segments are buffered by 150 m to ensure a minimum zone length of 300 m, which reflects practical signage requirements.

5. **Close gaps:** Morphological operations (dilation + erosion) fill gaps smaller than 500 m between candidate zones to create continuous speed reduction areas.

6. **Finalize selection:** Only segments sharing the same road name as a trigger segment, or located within 1 m of a candidate, are retained. This prevents unrelated parallel roads from being incorrectly included.

---

## Installation and Setup

### Prerequisites

- **OSGeoLive 16.0** (recommended as virtual machine)
- PostgreSQL with PostGIS extension
- GeoServer 2.22.2
- Node.js with npm
- osm2pgsql
- Firefox (or another modern browser)

### 1. Install OSGeoLive 16.0 as a Virtual Machine

Download and set up OSGeoLive 16.0 as your base environment. This provides a pre-configured GIS stack including PostgreSQL, PostGIS, and GeoServer.

### 2. Clone This Repository

```bash
git clone https://github.com/HKA-OSGIS/ZoneScope30
```

### 3. Prepare OSM Data

Place your OpenStreetMap data file (`.osm.pbf` format) into the `database/data/` directory before running the setup script. The script will automatically detect and import the first `.osm.pbf` file found in this directory.

### 4. Initialize Setup

Navigate to the repository folder and run the setup script to install all dependencies and configure the system:

```bash
cd ZoneScope30
bash setup.sh
```

This script performs the following tasks:
- Creates and configures the PostgreSQL database with PostGIS
- Imports OSM data using osm2pgsql
- Runs the spatial analysis to identify Tempo 30 candidates
- Configures GeoServer with the required workspace and layers
- Sets up the Node.js proxy server

### 5. Start the Application

Run the start script to launch all services and open the web map:

```bash
bash start.sh
```

This will:
- Start PostgreSQL (if not already running)
- Start GeoServer
- Start the Node.js proxy server
- Open the web map in Firefox

---

## Tempo 30 Zone Logic

The classification of road segments follows a multi-stage spatial logic based directly on OSM tagging and neighborhood analysis.

### 1. Eligible Roads

Only roads meeting all of the following criteria are evaluated:

* Road class in: `highway=primary`, `highway=secondary`, and `highway=tertiary`.
* Not a living street: `highway!=living_street`
* No existing valid 30-zone tagging. Ignored cases include:
  * Numeric `maxspeed > 30`
  * Non-numeric `maxspeed` not in: `DE:zone:30`

### 2. Primary Zone Identification

A road segment becomes a Tempo 30 candidate if at least one of the following applies:

#### A. Automatic Assignment

* **Residential roads:**
  All `highway=residential` segments are automatically classified as Tempo 30.

#### B. Social Facilities (Protective Zones)

Applies to `highway=primary`, `highway=secondary`, and `highway=tertiary`.

A segment qualifies if it lies **within 50 m** of one or more of the following:

* Schools: `amenity=school`
* Kindergartens: `amenity=kindergarten`, `amenity=childcare`
* Senior & care facilities: `nursing_home`, `hospital`, `social_facility` with `social_facility:for=senior`
* Playgrounds: `leisure=playground`
* Pedestrian crossings: `highway=crossing` with `crossing=zebra` or `crossing_ref=zebra`

#### C. Noise Protection (Residential Exposure)

Applies to `highway=primary` only.

A segment qualifies if it lies **within 5 m** of residential buildings:

* `building=residential`
* `building=apartments`
* `building=house`
* `building=terrace`

### 3. Zone Extension (Network Consistency)

After primary segments are determined, zones are extended for consistency:

* **Protected zone length:**
  Identified segments are treated as 300 m protected corridors.

* **Gap filling:**
  If two Tempo-30 corridors are **less than 500 m apart**, the intermediate road segment is also classified as Tempo 30.

This produces continuous, real-world-aligned Tempo-30 areas instead of isolated fragments.

---

## Assumptions and Data Dependencies

### Data Assumptions

- **OSM data quality:** The analysis relies on the accuracy and completeness of OpenStreetMap data. Missing or incorrectly tagged features (e.g., schools, residential buildings) may lead to incomplete results.
- **Coordinate Reference System:** All spatial calculations are performed in EPSG:25832 (UTM Zone 32N), which is appropriate for Germany and Central Europe.
- **Building classification:** Only buildings explicitly tagged as residential types are considered for noise protection analysis.

### Data Dependencies

- **OpenStreetMap extract:** A `.osm.pbf` file covering the area of interest must be provided.
- **OSM tagging consistency:** Results depend on consistent use of OSM tags such as `highway`, `maxspeed`, `amenity`, `building`, and `leisure`.
- **Static data snapshot:** The analysis reflects the state of the data at the time of import. Changes in OSM data require re-running the setup process.

---

## Limitations

The following limitations apply to the current implementation:

- **Noise protection is based on simplified buffer zones** and does not reflect realistic noise propagation. Real-world noise levels depend on factors such as terrain, building density, and traffic volume, which are not modeled.

- **Traffic-related noise is not derived from real-time or dynamic data** (e.g., traffic flow, volume, or speed). The analysis uses static proximity to residential buildings as a proxy for noise exposure.

- **The logic for handling road cross-sections is basic** and may produce inaccurate results in complex layouts, such as intersections with multiple lanes, roundabouts, or elevated roads.

- **No validation against official Tempo 30 designations:** Results are candidates based on spatial criteria and do not reflect legally binding decisions.

- **Limited to German regulations:** The criteria (e.g., buffer distances, trigger objects) are based on German traffic law and may not apply to other countries.

- **Performance on large datasets:** Processing very large OSM extracts (e.g., entire countries) may require significant time and computational resources.

---

## Future Improvements

The following enhancements are planned or recommended for future development:

- **Integrate real-time or time-dependent traffic data** to improve noise estimation accuracy. Traffic flow APIs or historical traffic patterns could provide more realistic input for noise-related zone identification.

- **Implement a more robust and flexible cross-section analysis** to handle complex road geometries, including multi-lane roads, dual carriageways, and intersections.

- **Enable updating and refreshing of the underlying datasets** without requiring a full re-import. Incremental updates from OSM would allow the system to stay current with minimal effort.

- **Add a progress bar or status feedback** to provide users with information during data import and analysis, especially for large datasets.

- **Support for additional trigger objects** such as hospitals, places of worship, or sport facilities, depending on local regulations.

- **Configurable buffer distances** to allow users to adjust the analysis parameters without modifying the SQL code.

- **Export functionality** to allow users to download the analysis results as GeoJSON, Shapefile, or other common GIS formats.

- **Multi-language support** for the web interface and documentation.

---

## Architecture

The following diagram illustrates the system architecture and data flow:

![Architecture Diagram](Architecture.png)

**Data Flow:**
1. OSM data is imported into PostgreSQL using osm2pgsql
2. PostGIS performs spatial analysis and stores results
3. GeoServer exposes the results as WFS (Web Feature Service)
4. The Express.js proxy handles CORS for browser access
5. MapLibre GL JS renders the results in the web browser
