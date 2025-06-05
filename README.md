# info499-kr2

## PROCESS

1. Create new project with local DBMS

	*open database with browser to query

2. Installed APOC plugin for database (tab on the side) to read json files
EDITED this line in the middle of the [project → database→ settings] *scroll to bottom and start scrolling up to find the specific line to edit – just above “JVM Parameters” section under “Miscellaneous Configuration”
	dbms.security.procedures.whitelist=apoc.coll.*,apoc.load.*,apoc.*

3. Copied original unzipped geojson files to the import folder OF THIS DB

4. Created “apoc.conf” text document (without the “.txt”) at the end and added these lines to the notepad
	apoc.import.file.enabled=true
  apoc.import.file.use_neo4j_config=true
  apoc.import.file.enabled=true
  apoc.export.file.enabled=true
  dbms.security.allow_csv_import_from_file_urls=true

5. Restarted database

7. Import data

**Importing Node file WITH metadata**

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
  n.source = "wa.microsoft.graph.nodes.OSW.geojson"
  SET n += clean_props


**Importing point file WITH metadata**

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
  n.source = "wa.microsoft.graph.points.OSW.geojson"
  SET n += clean_props

**Importing edge file WITH metadata and all internal nodes**

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

# TESTING 

## Export data in Neo4j to test with online visualization tool: https://geojson.io/

### Exporting points and nodes
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

### Exporting Relationships *created file but had to manually delete outter wrap {}

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

