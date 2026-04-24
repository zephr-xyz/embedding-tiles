# Embedding Tiles

Overture POI embedding tiles on a **z14 web mercator grid** with 768-dim EmbeddingGemma vectors, LLM-generated descriptions, visual descriptions from street-level imagery, navigation context, and Mapillary image links.

## Datasets

| Dataset | Path | POIs | Tiles | Region |
|---------|------|------|-------|--------|
| **Boulder** | `embedding-tiles-overture/z14/` | 12,406 | 575 | Boulder County, CO |
| **St. Louis** | `stlouis/z14/` | 1,890 | 2 | Downtown St. Louis, MO |

Each tile is available in two formats: `.pb` (protobuf, compact) and `.json` (human-readable).

## S3 Access

Boulder tiles are publicly hosted on S3:

```
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x{X}_y{Y}.pb
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x{X}_y{Y}.json
```

**Example:**
```bash
# Fetch a tile covering downtown Boulder
curl -O https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x3401_y6200.pb

# Fetch the JSON version
curl -O https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x3401_y6200.json
```

## Tile Format

### Tile Grid

Tiles follow the standard z14 web mercator (slippy map) tiling scheme. Each tile filename encodes its grid position: `x{X}_y{Y}.pb`.

To convert lat/lon to tile coordinates:

```python
import math

def lat_lon_to_tile(lat, lon, zoom=14):
    n = 2 ** zoom
    x = int((lon + 180.0) / 360.0 * n)
    y = int((1.0 - math.log(math.tan(math.radians(lat)) + 1.0 / math.cos(math.radians(lat))) / math.pi) / 2.0 * n)
    return x, y

# Downtown Boulder → x3401_y6200
lat_lon_to_tile(40.015, -105.27)

# Downtown St. Louis → x4087_y6283
lat_lon_to_tile(38.627, -90.197)
```

### Protobuf Schema

```protobuf
message WaypointsData {
  repeated WaypointProto waypoints = 1;
}

message WaypointProto {
  string id = 1;                            // Overture UUID
  string name = 2;                          // POI name
  double latitude = 3;                      // WGS84 latitude (facade-snapped)
  double longitude = 4;                     // WGS84 longitude (facade-snapped)
  string description = 5;                   // LLM-generated POI description
  int64  created_at = 6;                    // Always 0
  repeated float embedding = 8;             // 768-dim EmbeddingGemma document embedding (packed)
  string embedding_document = 9;            // Text used to generate the document embedding
  string types = 11;                        // Overture POI type (e.g. "restaurant", "park")
  float  rating = 12;                       // Always 0.0
  string visual_description = 14;           // AI-generated description from Mapillary street-level images
  repeated float visual_embedding = 15;     // 768-dim EmbeddingGemma visual embedding (packed)
  repeated RoadSegmentProto enclosing_roads = 16;
  string address = 17;                      // Street address
  string phone = 18;                        // Phone number
  string website = 19;                      // Website URL
  string overture_building_id = 20;         // Overture Maps building ID
  string facebook_id = 21;                  // Facebook page ID
  repeated string mapillary_ids = 22;       // Mapillary image IDs with views of this POI
  double entrance_lat = 23;                 // Facade-snapped entrance latitude
  double entrance_lon = 24;                 // Facade-snapped entrance longitude
  NavContextProto nav_context = 25;         // Navigation context
}

message NavContextProto {
  string building_gers_id = 1;              // Overture building ID
  string building_source = 2;              // e.g., "OpenStreetMap", "Microsoft", "Google"
  string entrance_type = 3;                // "facade", "corridor", or "centroid"
  double facade_bearing = 4;               // Compass direction facing out from entrance (0-360)
  string facade_cardinal = 5;              // N, NE, E, SE, S, SW, W, NW
  string entrance_street = 6;              // Street name the entrance faces
  string address_street = 7;               // Street from address (if different)
  bool address_street_mismatch = 8;        // True if entrance_street != address_street
  repeated string co_tenants = 9;          // Other businesses sharing this entrance
  string venue_type = 10;                  // "office" or "storefront"
  repeated string building_tenants = 11;   // All businesses in the building
  bool osm_confirmed = 12;                // Independent OSM node confirms placement
  int32 osm_match_tier = 13;              // Quality tier for OSM name match (1-6)
}

message RoadSegmentProto {
  string name = 1;                          // Road name
  string road_class = 2;                    // Road class (residential, primary, etc.)
  double start_lat = 3;                     // Segment start latitude
  double start_lon = 4;                     // Segment start longitude
  double end_lat = 5;                       // Segment end latitude
  double end_lon = 6;                       // Segment end longitude
}
```

### Waypoint Fields

| Field | # | Type | Description |
|-------|---|------|-------------|
| `id` | 1 | string | Overture UUID |
| `name` | 2 | string | POI name |
| `latitude` | 3 | double | WGS84 latitude (facade-snapped) |
| `longitude` | 4 | double | WGS84 longitude (facade-snapped) |
| `description` | 5 | string | LLM-generated POI description |
| `created_at` | 6 | int64 | Timestamp (always 0) |
| `embedding` | 8 | repeated float | 768-dim EmbeddingGemma document embedding |
| `embedding_document` | 9 | string | Source text: `"{name}, {types}. {description}"` |
| `types` | 11 | string | Overture POI type category |
| `rating` | 12 | float | Always 0.0 |
| `visual_description` | 14 | string | AI-generated description from street-level imagery |
| `visual_embedding` | 15 | repeated float | 768-dim EmbeddingGemma visual embedding |
| `enclosing_roads` | 16 | repeated RoadSegmentProto | Adjacent road segments with coordinates |
| `address` | 17 | string | Street address |
| `phone` | 18 | string | Phone number |
| `website` | 19 | string | Website URL |
| `overture_building_id` | 20 | string | Overture Maps building feature ID |
| `facebook_id` | 21 | string | Facebook page ID |
| `mapillary_ids` | 22 | repeated string | Mapillary street-level image IDs |
| `entrance_lat` | 23 | double | Facade-snapped entrance latitude |
| `entrance_lon` | 24 | double | Facade-snapped entrance longitude |
| `nav_context` | 25 | NavContextProto | Navigation context (see below) |

### NavContextProto Fields

| Field | # | Type | Description |
|-------|---|------|-------------|
| `building_gers_id` | 1 | string | Overture building ID |
| `building_source` | 2 | string | Data source (e.g., "OpenStreetMap") |
| `entrance_type` | 3 | string | "facade", "corridor", or "centroid" |
| `facade_bearing` | 4 | double | Compass direction facing out from entrance (0-360) |
| `facade_cardinal` | 5 | string | Cardinal direction: N, NE, E, SE, S, SW, W, NW |
| `entrance_street` | 6 | string | Street name the entrance faces |
| `address_street` | 7 | string | Street from address (if different from entrance_street) |
| `address_street_mismatch` | 8 | bool | True if entrance_street != address_street |
| `co_tenants` | 9 | repeated string | Other businesses sharing this entrance |
| `venue_type` | 10 | string | "office" or "storefront" |
| `building_tenants` | 11 | repeated string | All businesses in the building |
| `osm_confirmed` | 12 | bool | Independent OSM node confirms placement |
| `osm_match_tier` | 13 | int32 | Quality tier for OSM name match (1-6) |

### RoadSegmentProto Fields

| Field | # | Type | Description |
|-------|---|------|-------------|
| `name` | 1 | string | Road name |
| `road_class` | 2 | string | Overture class (residential, primary, etc.) |
| `start_lat` | 3 | double | Segment start latitude |
| `start_lon` | 4 | double | Segment start longitude |
| `end_lat` | 5 | double | Segment end latitude |
| `end_lon` | 6 | double | Segment end longitude |

### JSON Format

Each `.json` companion file mirrors the protobuf tile:

```json
{
  "waypoints": [
    {
      "id": "12537670-cef9-4814-8638-a27fb5e5ea57",
      "name": "At Your Convenience",
      "latitude": 38.6243428,
      "longitude": -90.2001062,
      "address": "1222 Spruce St",
      "types": "convenience_store",
      "phone": "+13144361046",
      "website": "http://www.gatewaycfc.org",
      "overture_building_id": "6b0ab265-b64b-4579-af40-8304edeb17ae",
      "description": "At Your Convenience is a convenience store offering a wide selection of everyday essentials...",
      "embedding_document": "At Your Convenience, convenience_store. At Your Convenience is a convenience store...",
      "embedding": [-0.0282, -0.0006, "...768 floats total..."],
      "facebook_id": null,
      "mapillary_ids": [],
      "visual_description": null,
      "visual_embedding": null,
      "created_at": 0,
      "rating": 0.0,
      "entrance_lat": 38.6243428,
      "entrance_lon": -90.2001062,
      "nav_context": {
        "building_gers_id": "6b0ab265-b64b-4579-af40-8304edeb17ae",
        "building_source": "OpenStreetMap",
        "entrance_type": "facade",
        "facade_bearing": 17.0,
        "facade_cardinal": "N",
        "entrance_street": "spruce street",
        "co_tenants": ["US Railroad Retirement Board", "..."],
        "venue_type": "office",
        "building_tenants": ["US Railroad Retirement Board", "..."]
      },
      "enclosing_roads": [
        {
          "road_name": "Spruce Street",
          "road_class": "residential",
          "start_lat": 38.6243543,
          "start_lon": -90.1997242,
          "end_lat": 38.624308,
          "end_lon": -90.199528
        }
      ]
    }
  ]
}
```

## Embeddings

Both embedding fields use **EmbeddingGemma** (768 dimensions):

- **`embedding`** (field 8) — Document embedding generated from `embedding_document` (field 9), which combines the POI name, type, and LLM-generated description into a searchable text representation. Prefix: `"title: {name} | text: {embedding_document}"`
- **`visual_embedding`** (field 15) — Visual embedding generated from `visual_description` (field 14), which describes the physical appearance of the POI from Mapillary street-level images. Prefix: `"task: sentence similarity | query: {visual_description}"`

## Dataset Statistics

### Boulder (`embedding-tiles-overture/`)

| Metric | Count |
|--------|-------|
| Total tiles | 575 |
| Total POIs | 12,406 |
| With document embedding (768d) | 12,406 |
| With embedding_document | 12,406 |
| With visual_description | 2,245 |
| With visual_embedding (768d) | 2,245 |
| With mapillary_ids | 4,246 |
| With entrance coordinates | 12,406 |
| With enclosing_roads | 11,898 |

### St. Louis (`stlouis/`)

| Metric | Count |
|--------|-------|
| Total tiles | 2 |
| Total POIs | 1,890 |
| With document embedding (768d) | 1,890 |
| With visual_description | 100 |
| With nav_context | 1,763 |
| With mapillary_ids | 89 |

## Enrichment Sources

The tiles combine data from multiple sources:

- **[Overture Maps](https://overturemaps.org/)** — Base POI data (name, location, type, address, building footprints)
- **[Mapillary](https://www.mapillary.com/)** — Street-level imagery for visual descriptions and image associations
- **[OpenStreetMap](https://openstreetmap.org/)** — Entrance confirmation and building data
- **EmbeddingGemma** — 768-dim embedding vectors for semantic search
- **Building geometry** — Entrance coordinates computed from Overture building footprints and road segments
- **LLM descriptions** — POI descriptions generated by language model from name, type, and available context

## Usage Examples

### Python (protobuf)

```python
from google.protobuf import descriptor_pb2, descriptor_pool, message_factory

# Build protobuf classes (or use a .proto file)
# ... (see schema above)

tile = WaypointsData()
with open("stlouis/z14/x4087_y6283.pb", "rb") as f:
    tile.ParseFromString(f.read())

for wp in tile.waypoints:
    print(f"{wp.name} ({wp.latitude}, {wp.longitude})")
    print(f"  Type: {wp.types}")
    print(f"  Embedding dims: {len(wp.embedding)}")
    if wp.visual_description:
        print(f"  Visual: {wp.visual_description[:100]}...")
    if wp.nav_context.entrance_type:
        print(f"  Entrance: {wp.nav_context.entrance_type} facing {wp.nav_context.facade_cardinal}")
    if wp.mapillary_ids:
        print(f"  Mapillary images: {len(wp.mapillary_ids)}")
```

### Python (JSON)

```python
import json

with open("stlouis/z14/x4087_y6283.json") as f:
    tile = json.load(f)

for wp in tile["waypoints"]:
    print(f"{wp['name']} — {wp['types']}")
    if wp.get("visual_description"):
        print(f"  {wp['visual_description'][:100]}...")
    nc = wp.get("nav_context")
    if nc:
        print(f"  Entrance: {nc['entrance_type']} facing {nc.get('facade_cardinal', 'unknown')}")
```

### JavaScript (browser, protobuf)

```javascript
// Using protobuf.js
const response = await fetch(
  'https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x3401_y6200.pb'
);
const buffer = await response.arrayBuffer();
const tile = WaypointsData.decode(new Uint8Array(buffer));

tile.waypoints.forEach(wp => {
  console.log(`${wp.name} (${wp.latitude}, ${wp.longitude})`);
});
```

## Coverage

| Dataset | Region | Bounding Box |
|---------|--------|-------------|
| Boulder | Boulder County, CO | 39.89°N–40.28°N, 105.58°W–105.03°W |
| St. Louis | Downtown St. Louis, MO | ~38.60°N–38.65°N, ~90.22°W–90.17°W |

## License

This data is derived from [Overture Maps](https://overturemaps.org/) (CDLA) and [Mapillary](https://www.mapillary.com/) (CC BY-SA). OpenStreetMap data was used in the aerial imagery to POI bounding boxes [OpenStreetMap](https://openstreetmap.org/) (ODbL). Embeddings and visual descriptions were generated by [zephr-maps](https://github.com/zephr-xyz/zephr-maps) pipeline.
