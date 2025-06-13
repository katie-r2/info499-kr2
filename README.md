# \[INFO499 - Independent Study\]  GeoJson Pedestrian Data + Neo4j (Desktop)

## Introduction 

This independent study explored the appliction and functionality of Neo4j and graph databases with pedestrian focused data. Using data from the [Transportation Data Exchance Iniative](https://tdei.cs.washington.edu/tdei/) (TDEI) of the Microsoft Campus in Redmond, WA, this study imported and manipulated GeoJson files using cypher, Neo4j Desktop, and NeoDash. These files were divided into 6 separate files for points, nodes, linesstrings, edges, zones, and polygons. However, only the point, node, and edge files were used in this this process to focus on sidewalk relationships and attributes. 

## Set Up

1. Create new project and new local database

2. Install APOC plugin ("apoc5plus") for database to read json files using the tab on the side

3. Edit the database settings (three dots next to 'open' button): Find "Miscellaneous Configuration" section _*easiest to scroll to bottom and start scrolling up to find the specific line to edit – just above “JVM Parameters”_

	Add this line: _dbms.security.procedures.whitelist=apoc.coll.*,apoc.load.*,apoc.*_

4. Copy original unzipped geojson files to the **_import_** folder of this database folder on your desktop *can use date as indicator of which one it is

5. Inside the database folder, find the _**conf**_ file. Create “apoc.conf” text document, without the “.txt” at the end, and add the following lines to the notepad

	_apoc.import.file.enabled=true_
	
	_apoc.import.file.use_neo4j_config=true_
	
	_apoc.import.file.enabled=true_
	
	_apoc.export.file.enabled=true_
	
	_dbms.security.allow_csv_import_from_file_urls=true_

6. Restart database

7. Open Neo4j Browser

## Import Data

### Importing Node file WITH metadata
```cypher
  CALL apoc.load.json("file:///wa.microsoft.graph.nodes.OSW.geojson")
  YIELD value
  UNWIND value.features AS feature
  WITH feature,
       feature.geometry.coordinates AS coords,
       apoc.map.clean(feature.properties, [], [NULL, [], {}]) AS clean_props
  CREATE (n:Node)
  SET
    n.id = feature.properties._id,
    n.location = point({longitude: coords[0], latitude: coords[1]})
  SET n.source = "wa.microsoft.graph.nodes.OSW.geojson"
  SET n += clean_props
```

### Importing point file WITH metadata
```cypher
  CALL apoc.load.json("file:///wa.microsoft.graph.points.OSW.geojson")
  YIELD value
  UNWIND value.features AS feature
  WITH feature,
       feature.geometry.coordinates AS coords,
       apoc.map.clean(feature.properties, [], [NULL, [], {}]) AS clean_props
  CREATE (n:Node)
  SET
    n.id = feature.properties._id,
    n.location = point({longitude: coords[0], latitude: coords[1]})
 SET  n.source = "wa.microsoft.graph.points.OSW.geojson"
  SET n += clean_props
```


### Importing edge file WITH metadata and all internal nodes
```cypher
  CALL apoc.load.json("file:///wa.microsoft.graph.edges.OSW.geojson")
  YIELD value
  UNWIND value.features AS feature
  
  WITH feature.geometry.coordinates AS coords, feature.properties AS props
  WITH [i IN range(0, size(coords)-2) |
    {
      from: coords[i],
      to: coords[i+1],
      props: props
    }
  ] AS segments
  
  UNWIND segments AS seg
  
  MERGE (a:Node {lat: round(seg.from[1]*1e6)/1e6, lon: round(seg.from[0]*1e6)/1e6})
    ON CREATE SET a.location = point({latitude: seg.from[1], longitude: seg.from[0]})
  
  MERGE (b:Node {lat: round(seg.to[1]*1e6)/1e6, lon: round(seg.to[0]*1e6)/1e6})
    ON CREATE SET b.location = point({latitude: seg.to[1], longitude: seg.to[0]})
  
  MERGE (a)-[r:CONNECTED_TO]->(b)
  SET r.source = "wa.microsoft.graph.edges.OSW.geojson"
  SET r += seg.props
```
## Testing  

To test with an external tool, export data in Neo4j to test with online visualization tool: https://geojson.io/

### Exporting points and nodes  *need to open file, manually delete outter wrap {}, and save
```cypher
CALL apoc.export.json.query("CALL {
    MATCH (n:Point) 
    WHERE n.latitude IS NOT NULL AND n.longitude IS NOT NULL
    RETURN {
      type: 'Feature',
      geometry: {
        type: 'Point',
        coordinates: [n.longitude, n.latitude]
      },
      properties: {
        source_type: 'point_latlon',
        neo4j_id: id(n),
        _id: n._id,
        id: n.id
      }
    } as feature
    
    d
    UNION ALL
    
    MATCH (n:Point) 
    WHERE n.location IS NOT NULL AND n.latitude IS NULL
    RETURN {
      type: 'Feature',
      geometry: {
        type: 'Point',
        coordinates: [n.location.longitude, n.location.latitude]
      },
      properties: {
        source_type: 'point_location',
        neo4j_id: id(n),
        _id: n._id,
        id: n.id,
        amenity: n.amenity,
        highway: n.highway,
        barrier: n.barrier,
        man_made: n.man_made
      }
    } as feature
    
    UNION ALL
    
    MATCH (n:Node) 
    WHERE n.location IS NOT NULL
    RETURN {
      type: 'Feature',
      geometry: {
        type: 'Point',
        coordinates: [n.location.longitude, n.location.latitude]
      },
      properties: {
        source_type: 'node',
        neo4j_id: id(n),
        _id: n._id,
        id: n.id,
        kerb: n.kerb,
        barrier: n.barrier,
        tactile_paving: n.tactile_paving
      }
    } as feature
  }
  RETURN {
    type: 'FeatureCollection',
    features: collect(feature)
  } AS value
  ",
  "file:///exported-points.geojson",
  {stream: false, jsonFormat: "JSON"}
)

```
### Exporting Relationships *need to open file, manually delete outter wrap {}, and save
```cypher
 CALL apoc.export.json.query(
  "
  MATCH (a:Node)-[r:CONNECTED_TO]->(b:Node)
  WHERE a.location IS NOT NULL AND b.location IS NOT NULL
  WITH {
    type: 'Feature',
    geometry: {
      type: 'LineString',
      coordinates: [
        [a.location.longitude, a.location.latitude],
        [b.location.longitude, b.location.latitude]
      ]
    },
    properties: {
      source_type: 'relationship',
      relationship_type: type(r),
      from_id: id(a),
      to_id: id(b),
      rel_id: id(r)
    }
  } AS feature
  WITH collect(feature) AS features
  UNWIND [ { type: 'FeatureCollection', features: features } ] AS fc
  RETURN fc
  ",
  "relationships7.geojson",
  {useTypes: true, jsonFormat: "JSON"}
)
```

## Finding Shortest Path Between Two Nodes of a Sidewalk Linestring

### Optional: Check the ID's of the relationship file
```cypher
MATCH (a:Node)-[r:CONNECTED_TO]->(b:Node)
WHERE a.location IS NOT NULL AND b.location IS NOT NULL
RETURN
  id(r) AS rel_id,
  id(a) AS from_id,
  id(b) AS to_id,
  type(r) AS relationship_type
LIMIT 100
```

### Shortest Path Query
*replace id(start) and id(end) values with desired nodes

```cypher
MATCH (start:Node), (end:Node)
WHERE id(start) =  1 AND id(end) = 2  
MATCH path = shortestPath((start)-[:CONNECTED_TO*..100]->(end))
RETURN path
```

### Finding Weighted Shortest Path with GDS Graph 

IMORTANT! The code below is flawed and does not run. However, it is a start, that can be debugged in the future, to use a projected graph to calculate edge lengths and use them to weight edges for more accurate shortest paths.

```
// Step 1: Compute and store edge lengths using point.distance()
CALL {
  MATCH (a:Node)-[r:CONNECTED_TO]->(b:Node)
  WHERE a.location IS NOT NULL AND b.location IS NOT NULL
  SET r.length = point.distance(a.location, b.location)
  RETURN count(*) AS _
}
// Step 2: Unconditionally drop and recreate graph (if it exists)
CALL {
  // Safe even if graph doesn't exist
  CALL {
    CALL gds.graph.drop('geoGraph', false) YIELD graphName RETURN graphName
  } CATCH (e) {
    RETURN "skipped" AS graphName
  }
  RETURN graphName
}
// Step 3: Create new GDS projection
CALL gds.graph.project(
  'geoGraph',
  'Node',
  {
    CONNECTED_TO: {
      properties: 'length'
    }
  }
) YIELD graphName
// Step 4: Match source and target nodes
WITH 4131 AS startId, 4161 AS endId
MATCH (start:Node), (end:Node)
WHERE id(start) = startId AND id(end) = endId
// Step 5: Run A* with weights and spatial heuristic
CALL gds.shortestPath.astar.stream('geoGraph', {
  sourceNode: start,
  targetNode: end,
  relationshipWeightProperty: 'length',
  latitudeProperty: 'location.latitude',
  longitudeProperty: 'location.longitude'
})
YIELD nodeIds, totalCost
// Step 6: Return readable path
UNWIND nodeIds AS nid
MATCH (n) WHERE id(n) = nid
RETURN n.name AS nodeName, nid, totalCost
ORDER BY index(nodeIds, nid)
```

**Useful Documentation**

_https://neo4j.com/docs/graph-data-science/current/management-ops/graph-creation/graph-project/_

_https://neo4j.com/docs/graph-data-science/current/management-ops/_


## Using NeoDash

NeoDash provides simple UI tools to query and map results directly from your database. It has the ability to display results in the form of single values, graphs, charts, and most useful to this project, a map underlay for coordinates to be accurately plotted on. The dashboard layout allows users to show the results of many different queries at one time through a flexible format.

**Examples of NeoDash**

1. Displaying single values (i.e. counts, sums, or sizes) -- This display is not limited to aggregation, but is a helpful usecase for hiding the actual code of the queries and just displaying results.

This query finds the count of nodes that make up the shortest path between node 4131 and node 4161.

```cypher
MATCH (start:Node), (end:Node)
WHERE id(start) =  4131 AND id(end) = 4161
MATCH path = shortestPath((start)-[:CONNECTED_TO*..100]->(end))
RETURN size(nodes(path)) AS nodeCount
```

2. Displaying plotted points with Map Type -- NeoDash allows you to query as you would in Neo4j Browser, but adjust the way the data is displayed. This visualization uses the 'map' type and is set to not show node labels.  

This query finds the shortest path between node 4131 and node 4161.

```cypher
MATCH (start:Node), (end:Node)
WHERE id(start) =  4131 AND id(end) = 4161
MATCH path = shortestPath((start)-[:CONNECTED_TO*..100]->(end))
RETURN path

```
**Screenshot of Dashboard of Node Count and Plotted Shortest Path**
![image](https://github.com/user-attachments/assets/3785e2e7-634c-4037-9a6d-5deae22b149f)

## Conclusion

Harnessing the combination of Neo4j Browser and NeoDash has a lot of potential for efficent querying of pedestrian data. However, with sidewalk data, there are thousands of node and edge relationshps that make up a very small region (over 11,000 in the Microsoft Campus alone!). When trying to display the whole graph, this noticibly slowed down the software and created a bit of lag in the system, though still usable. This process shows the sucessful application of GeoJson data and Neo4j. Cypher syntax and its available procedures/functions are ideal for querying this type of data, as it allows users to plainly describe the desired result (i.e. the function for finding the shortest path between two nodes is shortestPath()). Overall, Neo4j proves to be a powerful tool for spatial data analysis, especially when combined with visualization platforms, though thoughtful query design and data management are essential for maintaining performance at scale.

## Project Details, Resources, and References

INFO499 - Independent Study SP25

Project completed by Kathryn Rochleau-Rice

Advisor: Bill Howe

**Resources & References**

_https://lyonwj.com/blog/spatial-cypher-cheat-sheet_

_https://neo4j.com/blog/developer/routing-web-app-neo4j-openstreetmap-leafletjs/_

_https://neo4j.com/labs/apoc/_

_https://neo4j.com/docs/cypher-manual/current/patterns/shortest-paths/_

_https://neo4j.com/docs/graph-data-science/2.18/algorithms/astar/_

_Chatgpt assisted code_
