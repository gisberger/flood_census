# SQL-based Flood Exposure Analysis - Landkreis Altötting, Bavaria

## What This Project Is About

The project is an attempt to quantify the number of people affected by different flood scenarios in a 569.4 km² district in Southern Germany based on publicly available data.

Potential for streamlining map import into the database using Python is also explored.

Results will be compared to the official numbers of affected population on the official BfG flood risk map.

This project asks: **how close can we get to the official numbers using only open census data and a spatial database?**

Using LfU Bayern's official flood hazard polygons (frequent, 100-year and extreme floods), and the 2022 German Census 100m population grid, this analysis estimates the number of residents affected by each flood scenario across all 16 municipalities in Landkreis (district of) Altötting.

An area-weighted approach accounts for census cells that are only partially covered by flood zones. The results are compared side-by-side against the official BfG reference values.

All SQL-code was hand-written, with trouble-shooting support by Claude model Opus 4.7.

---

## Project Outcome

The analysis produced population exposure estimates for three flood scenarios across 16 municipalities. For several municipalities, the results align closely with official BfG figures - Garching a.d. Alz showed 90 affected residents in this analysis vs. 80 in the official data for HQ100. Other municipalities showed larger deviations, which are discussed in the methodology section below.

### Final Comparison Table

| Municipality | HQ100 (own) | HQ100 (BfG) | HQextrem (own) | HQextrem (BfG) | HQhäufig (own) | HQhäufig (BfG) |
|---|---|---|---|---|---|---|
| Altötting | 755 | 0 | 3267 | 0 | 0 | 0 |
| Burghausen | 128 | 30 | 474 | 1 | 105 | 190 |
| Burgkirchen a.d. Alz | 115 | 80 | 1514 | 10 | 29 | 1610 |
| Emmerting | 7 | 1 | 2897 | 1 | 3 | 2660 |
| Garching a.d. Alz | 90 | 80 | 759 | 1 | 20 | 730 |
| Haiming | 1 | 0 | 1 | 0 | 0 | 0 |
| Kirchweidach | 0 | 0 | 4 | 0 | 0 | 0 |
| Marktl | 100 | 10 | 122 | 1 | 35 | 20 |
| Neuötting | 380 | 40 | 1866 | 20 | 2 | 1710 |
| Pleiskirchen | 1 | 0 | 2 | 0 | 0 | 0 |
| Reischach | 268 | 10 | 419 | 1 | 31 | 30 |
| Teising | 43 | 0 | 148 | 0 | 0 | 0 |
| Töging a. Inn | 5 | 0 | 10 | 0 | 0 | 0 |
| Tüßling | 586 | 0 | 2051 | 0 | 0 | 0 |
| Unterneukirchen | 26 | 40 | 92 | 1 | 7 | 60 |
| Winhöring | 74 | 0 | 142 | 0 | 0 | 0 |

Note: BfG values of 0 indicate that no official data was available for that municipality/scenario combination on the BfG Geoportal, not that zero residents are affected.

---

### Sample map of results
Below is a simple map showing the affected population in different sceanrios (100-year / extreme / frequent). The symbology represents a categorized vulnerability degree for frequent floods.

<img width="2455" height="1736" alt="image" src="https://github.com/user-attachments/assets/db728942-7b05-44f9-acfe-d943f54803fc" />


## Learning Journey

This project was built iteratively, with each step introducing a new concept or solving a problem that emerged from the previous one. The key learning milestones were:

### 1. Setting Up PostGIS in Docker

The database runs in a _Docker container_ using the kartoza/postgis image.

A _docker-compose.yml_ file contains the instructions for setting up the container and the database. With this file, it only takes one line of code to set them up (```docker compose up -d```).

Initial setup learnings included resolving authentication issues (peer vs. password auth, finding the correct environment variables) and enabling the PostGIS extension.


### 2. Importing Spatial Data

Layers can be imported into databases in different ways:

1) _Using the DB Management tool in QGIS - easy, manual but not repeatable_

Geodata in different formats can be imported with a few clicks. However, only one DB can be imported each time. This is defensible for importing layers in projects where only a few are needed.

2) _Via the command line - processes one file each time but is repeatable_

In order to add any spatial data to the database, I would first have to enter the container

```psql -h localhost -U <Benutzer> -d <DB Name>```

From there, the files need to be copied from the machine into a temporary directory in the container:

```docker cp <file_path> postgis_container:/tmp/```

Shapefiles are then imported using `shp2pgsql`, indicating the CBS, the file name inside the temp dir, the desired table name, username and database name:

```shp2pgsql -s 25832 -I /tmp/<file> table_name | psql -U user -h localhost -d db_name```

3) _Using a Python script - automated, suitable for bulk imports, can be used for any future imports_

While the scope of my project did not call for automation, I wanted to test the possibilites for optimization by using Python scripts for automated import.

Claude model Opus 4.7 has generated a script that automatically checks a given directory for .shp files, copies them into the container and prepares it for addition into the database.

After a few adjustments, I successfully used it to import all of the .shp files into my database in one step. The script also proved useful in other projects and has the potential to significantly speed up import in projects with a larger number of layers.

The script was run in a previously set-up virtual environment.

### 3. CRS Mismatches

The initial cross-layer queries returned zero results. Diagnosis via SQL revealed three different coordinate reference systems across the imported tables (25832, 4326, 3857). This was resolved by reprojecting all layers to EPSG:25832 using `ST_Transform`:

```sql
ALTER TABLE zensus2022_100m
  ALTER COLUMN geom TYPE geometry
  USING ST_Transform(geom, 25832);
```

Lesson: always check SRIDs immediately after import.

### 4. Understanding Spatial Joins

The core analytical pattern - joining tables on spatial relationships - took several iterations to understand properly. The key insight was that `JOIN ... ON ST_Intersects(a.geom, b.geom)` is essentially a spatial filter: it pairs every geometry from table A with every geometry from table B that it touches, and discards non-matching rows.

Early confusion around LEFT JOIN vs. INNER JOIN, and when to use `ST_Intersects` vs. `ST_Within`, was resolved through experimentation and examining result sets.

Below is a visualization of the mechanisms of different types of SQL joins.

<img width="766" height="600" alt="image" src="https://github.com/user-attachments/assets/f119cd31-29ae-4140-b6bc-3badafec7163" />
*Source: C.L. Moffatt, 2008 - [Visual Representation of SQL Joins](https://www.codeproject.com/Articles/33052/Visual-Representation-of-SQL-Joins)*

### 5. The Double-Counting Problem

The first population query returned implausible numbers - some municipalities showed more affected residents than total inhabitants. Investigation revealed that the three-way JOIN between census cells, flood polygons, and municipality boundaries created a cross-product: a census cell near a municipal border could match with neighboring municipalities via the flood polygon, inflating the count.

The fix was restructuring the query to first identify which census cells fall in flood zones (inner subquery), then assign those to municipalities (outer query):

```sql
SELECT a.name_3 AS gemeinde,
       SUM(f."Einwohner") AS betroffene
FROM adm_adm_3 a
JOIN (
    SELECT z.id, z.geom, z."Einwohner"
    FROM zensus2022_100m z
    JOIN hwgfhq100_15_04_2026 h ON ST_Intersects(h.geom, z.geom)
) f ON ST_Intersects(f.geom, a.geom)
WHERE a.name_2 LIKE '%Altötting%'
GROUP BY a.name_3
ORDER BY betroffene DESC;
```

### 6. Area-Weighted Population Estimation

Using `ST_Intersects` alone overcounts because census cells at the edge of a flood zone contribute their full population even if only partially covered. The solution was to weight each cell's population by the fraction of its area that falls within the flood zone:

```sql
ROUND(SUM(z."Einwohner" * (ST_Area(ST_Intersection(z.geom, h.geom)) / ST_Area(z.geom)))::numeric, 0)
```

A side investigation compared using a fixed average cell area (10,009.86 m², calculated with SQL) vs. computing each cell's area individually. The standard deviation of cell areas was only 2 m², so both methods produced near-identical results (deviation < 0.1%). 

Since there was no siginifant difference in processing speed when calculating each cell's individual surfache area,  the per-cell calculation was used for accuracy.

### 7. Combining Three Flood Scenarios

Combining HQ100, HQextrem, and HQhäufig into a single result table required solving two problems:

**Cross-product explosion:** Joining three independent spatial result sets in a single query multiplied rows against each other (50 × 80 × 30 = 120,000 rows per municipality). The solution was to aggregate each scenario independently first, then join the results on municipality name - a text join, not a spatial one:

```sql
CREATE TABLE result_hq100 AS
SELECT a.name_3 AS gemeinde,
       ROUND(SUM(z."Einwohner" * (ST_Area(ST_Intersection(z.geom, h.geom))
       / ST_Area(z.geom)))::numeric, 0) AS betroffene
FROM zensus2022_100m z
JOIN hwgfhq100_15_04_2026 h ON ST_Intersects(h.geom, z.geom)
JOIN adm_adm_3 a ON ST_Intersects(z.geom, a.geom)
WHERE a.name_2 LIKE '%Altötting%'
GROUP BY a.name_3;
-- Repeated for HQextrem and HQhäufig

SELECT COALESCE(a.gemeinde, b.gemeinde, c.gemeinde) AS gemeinde,
       COALESCE(a.betroffene, 0)::integer AS hq100,
       COALESCE(b.betroffene, 0)::integer AS extrem,
       COALESCE(c.betroffene, 0)::integer AS haeufig
FROM result_hq100 a
FULL OUTER JOIN result_extrem b ON a.gemeinde = b.gemeinde
FULL OUTER JOIN result_haeufig c ON a.gemeinde = c.gemeinde
ORDER BY gemeinde;
```

**Preserving all municipalities:** `FULL OUTER JOIN` ensures municipalities that appear in only one or two scenarios are still included.

**Avoiding NULL values in the table** `COALESCE(number_of_people, 0)` is used to ensure that municipalities appearing in only _some_ flood scenarios still show a value of 0 instead of NULL, keeping the output table complete and readable.

### 8. Validation Against Official Data (+handling naming inconsistency)

BfG reference values were manually extracted from the LfU Bayern's Hochwasserrisikokarte (flood risk map), manually typed into a CSV table, which was then joined to the results.

I intentionally built in naming inconsistencies in the municipality name column in order to see if I can find a way for the join to work even with this common kind of inconsistency.

Using fuzzy name matching (`LIKE '%' || name || '%'`), the join was able handle differences like "Burgkirchen" vs. "Burgkirchen a.d. Alz".

---

## Why the Values Differ

Several factors explain deviations between this analysis and the official BfG figures:

- **Population data resolution:** The BfG/LfU approach distributes population to individual residential buildings using ATKIS Basis-DLM land use data, possibly even having access to non-public data such as reinforcements of buildings that make them fall below a threshold. 
  
- This analysis uses a 100m census grid where population is spread uniformly across each cell, including portions that may be roads, gardens, or fields.

- **Building geometry not considered:** This analysis treats the flood zone as a 2D surface. The official assessment considers building footprints and may account for water depth in relation to building use, which this analysis does not. This could explain the general tendency to over-estimating the number of affected population.

- **Missing municipalities in the official BfG data:** Several municipalities (Altötting, Tüßling, Teising, Haiming, and others) show no values in the BfG Geoportal. This possibly reflects incomplete coverage in the 2019 reporting cycle rather than zero flood risk, given that this analysis identifies significant exposure in those areas.

<img width="647" height="416" alt="image" src="https://github.com/user-attachments/assets/7e013e52-fb8d-4b44-a275-6914c6190ba6" />
*The official map shows no number for affected inhabitants for Altötting in any scenario depsite there being residential buldings in the municipality that are withing flood risk zones (here: extreme risk)*

- **Data vintage:** The BfG portal shows data from the 2019 reporting cycle. This analysis uses 2022 Census data and flood hazard polygons downloaded in April 2026. Population changes and updated flood models might contribute to differences.


**These differences will be explored in a follow-up analysis, incorporating ATKIS building footprints and possibly terrain for a more precise population distribution.**

---

## Data Sources

| Dataset | Source | Format | CRS |
|---|---|---|---|
| Hochwassergefahrenflächen HQ100, HQextrem, HQhäufig | LfU Bayern (provided on request) | Shapefile | EPSG:25832 |
| Zensus 2022 - Bevölkerung 100m-Gitter | Statistisches Bundesamt | GeoPackage | EPSG:3857 (reprojected to 25832) |
| Verwaltungsgrenzen (Gemeinden, Landkreise) | GADM / BKG | Shapefile | EPSG:4326 (reprojected to 25832) |
| BfG Hochwasserrisikokarte - Betroffene Einwohner | BfG Geoportal | (numeric values copied manually) | - |  

---

## Tools & Technologies

- **PostgreSQL + PostGIS** - Spatial database, all analysis performed via SQL
- **Docker** (kartoza/postgis image) - Database containerization
- **QGIS** - Data visualization and DB Manager for query development
- **GDAL** (`shp2pgsql`) - Data import and format conversion
- **VS Code** - Container setup, data import, script execution

---

## Limitations

- The 100m census grid distributes population uniformly across each cell. In reality, residents cluster in buildings. A cell that is 10% residential and 90% farmland will underestimate the population density where people actually live.
- The analysis considers only the 2D surface area of flood zones. It does not account for flood depth, flow velocity, or building elevation - all of which significantly affect actual flood impact.
- Census cells at municipal borders may be assigned to the wrong municipality depending on which side of the boundary their centroid or majority area falls.
- The area-weighting approach assumes uniform population distribution within each 100m cell, which is a simplification.
- Flood hazard polygons represent modeled scenarios, not observed events. Actual flood extents depend on conditions not captured in HQ models (e.g., infrastructure failures, debris blockages).

---

## Future Enhancements

- **ATKIS building footprint integration:** Distributing population to residential building polygons instead of a 100m grid would significantly improve spatial precision and allow direct comparison with the official BfG methodology.
- **Flood depth analysis:** Incorporating the LfU's water depth rasters (Wassertiefenkarten) would enable damage potential estimation, not just exposure counting.
- **Contamination pathway analysis:** Overlaying flood zones with Trinkwasserschutzgebiete and hydrogeological permeability data (HÜK250) to identify where floodwater may infiltrate into drinking water aquifers - a separate analysis currently in development.

---

## Repository Structure

This project started out without the intention of posting to GitHub and Git was not implemented. Files will not be shared. 

---

## Author

gisberger (Andreas Giglberger)

---
 assessment, and field-based infiltration measurements. The overarching project uses the same PostGIS infrastructure and will be documented separately.
