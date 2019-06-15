# Efficiency Analysis of Wind Farms

## 0. Introduction
We want to build a tool that identifies "troubled assets" - wind farms that are underperforming based on regional averages. Here we want to build an interactive web application that highlights wind farms that are unusually inefficient relative to neighboring wind farms within 100 km.<br>

It includes two steps: spatial analysis and web application. PostGIS is used for spatial analysis, and ArcGIS online StoryMap is used for the web application. <br>

## 1. Spatial Analysis
First, we want to create a database in PostgreSQL. Use the command:

```
sudo -u postgres createdb -O postgres windfarms
```
Here, postgres is the username, and the windfarms is the database name.<br><br>
Then, we want to connect to the database:
```
psql -h localhost -U postgres windfarms
```
Because, we are going to deal with spatial data, we want to create PostGIS extension for the database:
```
CREATE EXTENSION postgis;
```
Next, open the csv file of wind farms in QGIS, and save it as ESRI shapefile. Then, we want to import the shapefile to the database:
```
shp2pgsql -I -s 4326 "Windfarms.shp" wind_farms | psql -U postgres -d windfarms
```
Here, we use "wind_farms" for the table name. Next, we can open PgAdmin for spatial query. <br><br>
Since we want to analyze the average regional efficiency, we want to project the data to a projected coordinate system. To do this, we can type:
```
ALTER TABLE wind_farms
 ALTER COLUMN geom TYPE geometry(Point,2163) 
  USING ST_Transform(geom,2163);
```
Here, we use US National Atlas Equal Area coordinate system, whose spatial reference code is 2163.<br><br>
We note that in the table, some rows has no generation data. We want to remove these rows. Also, because very small wind farms are often used for research purposes or not professionally maintained, we want to filter out wind farms below a certain capacity. We want to keep the rows whose capacity is now less than than the 10th percentile of capacity. Additionally, we want to calculate the effiency based on the capacity and generation in this step. After this step, we will get a temporary table "wftemp".
```
CREATE TEMP TABLE wftemp AS (SELECT *, 
CAST(CASE WHEN generation = 'NA' THEN '0'
ELSE generation END AS NUMERIC)/(capacitymw*8760) AS efficiency
FROM wind_farms
WHERE capacitymw>=(SELECT PERCENTILE_DISC(0.1) WITHIN GROUP (ORDER BY wind_farms.capacitymw) FROM wind_farms) 
							 AND generation!= 'NA')
```
Let's move to the most important step. In this step, we want to:<br>
- Create the buffer of each wind farm in "wftemp". <br>
- Spatial join the wind farms and the buffers to find all wind farms falling within each buffer.<br>
- We have to note that each wind farm will join the buffer of itself. So, we want to remove the rows who represents this "self join".<br>
- Then, want to aggregate the data to get the average efficiency of wind farms falling into each buffer, and join the the results to the temporary table "wftemp".<br>
- Find the rows whose efficiency lower than the regional average (avg_efficiency) and label them.
```
CREATE TABLE final_wf AS (SELECT *, (CASE WHEN efficiency < avg_efficiency THEN 'No'
ELSE 'Yes' END) AS inefficient FROM wftemp 
JOIN (
SELECT buffer_id, AVG(efficiency) AS avg_efficiency FROM (SELECT wftemp.ID, wftemp.efficiency, buffer_id FROM wftemp 
JOIN(
	SELECT ID AS buffer_id, ST_Buffer(geom, 100000)::geometry(Polygon,2163) AS geom
FROM wftemp) buffer
ON ST_Within(wftemp.geom, buffer.geom)
WHERE ID != buffer_id) temp2
GROUP BY buffer_id) temp2_agg
ON wftemp.ID = temp2_agg.buffer_id)
```
Now, we get the results. Export the data to csv and move to the next part.

## 2. Web Application
We can build the web application via ArcGIS online, which can be viewed at:
https://www.arcgis.com/apps/StoryMapBasic/index.html?appid=8d30efe1fab9406ca8eb8358b3db6de4
