# Embedding Tiles

Overture POI embedding tiles for Boulder County, Colorado. Protobuf + JSON tiles on a **z14 web mercator grid** with 768-dim EmbeddingGemma vectors, visual descriptions from street-level imagery, and Mapillary image links.

Two datasets are provided:

| Dataset | Path | POIs | Description |
|---------|------|------|-------------|
| **Full** | `embedding-tiles-overture/z14/` | 12,406 | All Overture POIs in Boulder County with embeddings |
| **Visual-only** | `embedding-tiles-overture-visual/z14/` | 2,245 | POIs that have visual descriptions from Mapillary street-level imagery |

Each tile is available in two formats: `.pb` (protobuf, compact) and `.json` (human-readable).

## S3 Access

Tiles are publicly hosted on S3 and can be fetched directly:

```
# Full dataset
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x{X}_y{Y}.pb
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x{X}_y{Y}.json

# Visual-only dataset
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture-visual/z14/x{X}_y{Y}.pb
https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture-visual/z14/x{X}_y{Y}.json
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
```

### Protobuf Schema

```protobuf
message WaypointTile {
  repeated Waypoint waypoints = 1;
}

message Waypoint {
  string id = 1;                            // UUID
  string name = 2;                          // POI name
  double latitude = 3;                      // WGS84 latitude
  double longitude = 4;                     // WGS84 longitude
  string description = 5;                   // Overture POI description
  int64  created_at = 6;                    // Always 0
  repeated float embedding = 8;             // 768-dim EmbeddingGemma document embedding (packed)
  string embedding_document = 9;            // Text used to generate the document embedding
  string types = 11;                        // Overture POI type (e.g. "restaurant", "park")
  string visual_description = 14;           // AI-generated description from Mapillary street-level images
  repeated float visual_embedding = 15;     // 768-dim EmbeddingGemma visual embedding (packed)
  repeated RoadSegment enclosing_roads = 16;
  string phone = 17;
  string address = 18;
  string facebook_id = 19;
  double entrance_lat = 20;                 // Computed entrance latitude (road-facing facade midpoint)
  double entrance_lon = 21;                 // Computed entrance longitude
  repeated string mapillary_ids = 22;       // QA-verified Mapillary image IDs
  string website = 23;
  string overture_building_id = 24;         // Overture Maps building ID

}

message RoadSegment {
  string name = 1;                          // Road name
  string road_class = 2;                    // Road class (residential, primary, etc.)
}
```

### Waypoint Fields

| Field | Number | Type | Description |
|-------|--------|------|-------------|
| `id` | 1 | string | UUID identifier |
| `name` | 2 | string | POI name |
| `latitude` | 3 | double | WGS84 latitude |
| `longitude` | 4 | double | WGS84 longitude |
| `description` | 5 | string | Overture POI description |
| `created_at` | 6 | int64 | Timestamp (always 0) |
| `embedding` | 8 | repeated float | 768-dim EmbeddingGemma document embedding |
| `embedding_document` | 9 | string | Source text for the document embedding |
| `types` | 11 | string | Overture POI type category |
| `visual_description` | 14 | string | AI-generated description from street-level imagery |
| `visual_embedding` | 15 | repeated float | 768-dim EmbeddingGemma visual embedding |
| `enclosing_roads` | 16 | repeated RoadSegment | Adjacent road segments |
| `phone` | 17 | string | Phone number |
| `address` | 18 | string | Street address |
| `facebook_id` | 19 | string | Facebook page ID |
| `entrance_lat` | 20 | double | Entrance latitude (road-facing facade midpoint) |
| `entrance_lon` | 21 | double | Entrance longitude |
| `mapillary_ids` | 22 | repeated string | QA-verified Mapillary street-level image IDs |
| `website` | 23 | string | Website URL |
| `overture_building_id` | 24 | string | Overture Maps building feature ID |

### JSON Format

Each `.json` companion file mirrors the protobuf tile:

```json
{
  "waypoints": [
    {
      "id": "62b5a082-3366-4cbc-a527-16d3e1b04210",
      "name": "Nederland Elementary School",
      "latitude": 39.9614,
      "longitude": -105.5098,
      "description": "Public elementary school...",
      "types": "school",
      "embedding_document": "Nederland Elementary School, school. Public elementary school...",
      "embedding": [0.0123, -0.0456, ...],
      "visual_description": "Brown stone building with 'NEDERLAND ELEMENTARY' text mounted on exterior...",
      "visual_embedding": [0.0789, -0.0321, ...],
      "enclosing_roads": [{"name": "2nd Street", "road_class": "residential"}],
      "phone": "(303) 258-7092",
      "address": "258 Hwy 72 N, Nederland, CO 80466",
      "facebook_id": null,
      "entrance_lat": 39.9615,
      "entrance_lon": -105.5097,
      "mapillary_ids": ["1234567890123456"],
      "website": "https://ned.bvsd.org",
      "overture_building_id": "abc123-def456",
      "created_at": 0,
      "rating": 0.0,
      "metadata": null
    }
  ]
}
```

## Embeddings

Both embedding fields use **EmbeddingGemma** (768 dimensions):

- **`embedding`** (field 8) — Document embedding generated from `embedding_document` (field 9), which combines the POI name, type, and description into a searchable text representation
- **`visual_embedding`** (field 15) — Visual embedding generated from `visual_description` (field 14), which describes the physical appearance of the POI from Mapillary street-level images

The visual-only dataset contains only POIs where `visual_description` (field 14) is populated.

## Dataset Statistics

### Full Dataset (`embedding-tiles-overture/`)

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

### Visual-Only Dataset (`embedding-tiles-overture-visual/`)

| Metric | Count |
|--------|-------|
| Total tiles | 575 (227 non-empty) |
| Total POIs | 2,245 |
| With document embedding (768d) | 2,245 |
| With visual_description | 2,245 |
| With visual_embedding (768d) | 2,245 |
| With mapillary_ids | 1,852 |

## Enrichment Sources

The tiles combine data from multiple sources:

- **[Overture Maps](https://overturemaps.org/)** — Base POI data (name, location, type, address, building footprints)
- **[Mapillary](https://www.mapillary.com/)** — Street-level imagery
- **EmbeddingGemma** — 768-dim embedding vectors for semantic search
- **Building geometry** — Entrance coordinates computed from Overture building footprints and road segments

## Usage Examples

### Python (protobuf)

```python
from google.protobuf import descriptor_pb2, descriptor_pool, message_factory

# Build protobuf classes (or use a .proto file)
# ... (see schema above)

tile = WaypointTile()
with open("embedding-tiles-overture/z14/x3401_y6200.pb", "rb") as f:
    tile.ParseFromString(f.read())

for wp in tile.waypoints:
    print(f"{wp.name} ({wp.latitude}, {wp.longitude})")
    print(f"  Type: {wp.types}")
    print(f"  Embedding dims: {len(wp.embedding)}")
    if wp.visual_description:
        print(f"  Visual: {wp.visual_description[:100]}...")
    if wp.mapillary_ids:
        print(f"  Mapillary images: {len(wp.mapillary_ids)}")
```

### Python (JSON)

```python
import json

with open("embedding-tiles-overture/z14/x3401_y6200.json") as f:
    tile = json.load(f)

for wp in tile["waypoints"]:
    print(f"{wp['name']} — {wp['types']}")
    if wp.get("visual_description"):
        print(f"  {wp['visual_description'][:100]}...")
```

### JavaScript (browser, protobuf)

```javascript
// Using protobuf.js
const response = await fetch(
  'https://zephrpoint-public-models.s3.us-east-2.amazonaws.com/embedding-tiles-overture/z14/x3401_y6200.pb'
);
const buffer = await response.arrayBuffer();
const tile = WaypointTile.decode(new Uint8Array(buffer));

tile.waypoints.forEach(wp => {
  console.log(`${wp.name} (${wp.latitude}, ${wp.longitude})`);
});
```

## Coverage

The tiles cover **Boulder County, Colorado** (bbox: 39.89°N to 40.28°N, -105.58°W to -105.03°W).

## License

This data is derived from [Overture Maps](https://overturemaps.org/) (ODbL) and [Mapillary](https://www.mapillary.com/) (CC BY-SA). Embeddings and visual descriptions were generated by the [zephr-maps](https://github.com/zephr-xyz/zephr-maps) pipeline.
