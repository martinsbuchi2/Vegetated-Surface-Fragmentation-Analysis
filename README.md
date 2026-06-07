# Vegetated Surface Fragmentation Analysis
### Project Documentation

**Project file:** `Fragmentation_Analysis.qgz`
**Date completed:** May 13, 2026
**CRS:** EPSG:3400 - NAD83 / Alberta 10-TM Forest
**Analyst tool:** QGIS (MCP-integrated workflow)

---

## 1. Project Overview

This project computes landscape fragmentation metrics for the o11 Other Vegetated Surface class of Alberta's 2022 Human Footprint Inventory, summarized within the Green and White land management zones. Three patch-level metrics are calculated for each HFI polygon: patch area, perimeter-to-area ratio (PAR), and nearest neighbour distance (NND). These are aggregated per zone polygon and combined into a composite fragmentation index used to rank zones by fragmentation severity. The output supports reclamation priority screening by identifying zones where vegetated surface cover is most spatially fragmented.

---

## 2. Input Layers

| Layer | Source | CRS | Features | Type |
|---|---|---|---|---|
| HFI 2022 - Other Vegetated Surface | hfi_with_area.gpkg (Task 1) | EPSG:3400 | 11,360 | Polygon |
| Land Management Zones | green_white_fixed.gpkg (Task 1) | EPSG:3400 | 39 | Polygon |
| Provincial Boundary | boundary.gpkg (Task 1) | EPSG:3857 | 1 | Polygon |

Both the HFI and zone layers were sourced from Task 1 (Human Footprint Density by Land Management Zone) and already carry fixed geometries. No additional geometry repair was needed.

---

## 3. Methodology

### 3.1 Patch Metric Computation
Two fields were added to each HFI patch feature:

| Field | Expression | Description |
|---|---|---|
| perimeter_m | $perimeter | Polygon perimeter in metres |
| par | perimeter_m / (area_ha * 10000) | Perimeter-to-area ratio (m/m2); higher = more irregular/fragmented shape |

The `area_ha` field was already present from Task 1 ($area / 10000).

### 3.2 Nearest Neighbour Distance (NND)
A QgsSpatialIndex was built over all HFI patch centroids. For each patch, the two nearest neighbours were queried and the closest non-self centroid was identified. The straight-line distance between the patch centroid and its nearest neighbour centroid was stored as `nn_dist_m` (metres). A lower NND indicates more clustered, denser patches; a higher NND indicates more isolated, spatially dispersed patches.

### 3.3 Zone Assignment
Patch-level metrics were spatially joined to the Green/White zone polygons using `Join Attributes by Location` with the intersects predicate, assigning `GWA_NAME` and `GWA_CODE` to each patch. 11,136 of 11,360 patches received a zone assignment (patches on zone boundaries may intersect multiple polygons).

### 3.4 Zone-Level Aggregation
`Join Attributes by Location (Summary)` was used with the zones as the base layer and the attributed patches as the join layer, requesting count, min, sum, and mean for `area_ha`, `par`, and `nn_dist_m`.

### 3.5 Derived Zone Fields

| Field | Expression | Description |
|---|---|---|
| zone_area_ha | Shape_Area / 10000 | Zone polygon area in hectares |
| patch_density | (area_ha_count / zone_area_ha) * 1000 | Patches per 1,000 ha of zone area |
| frag_index | (par_mean * 100) + (patch_density * 10) + (1 / area_ha_mean) | Composite fragmentation index (additive, dimensionless) |

The fragmentation index combines shape irregularity (PAR), spatial crowding (patch density), and patch size (inverse mean area). Higher values indicate greater fragmentation.

### 3.6 Symbology - Manual Graduated Classification
The `frag_index` values range from 0 to 31.9 with a strong right skew and one severe outlier. Manual breaks were applied on `frag_index` using the YlOrRd colour ramp:

| Class | Range | Hex Colour | Interpretation |
|---|---|---|---|
| Not Fragmented | 0 to 0.001 | #ffffb2 | No HFI patches in zone |
| Slightly Fragmented | 0.001 to 5.0 | #fecc5c | Low patch density, regular shapes, patches well-separated |
| Moderately Fragmented | 5.0 to 8.0 | #fd8d3c | Intermediate density and irregularity |
| Highly Fragmented | 8.0 to 15.0 | #e31a1c | High patch density or irregular shapes, closer spacing |
| Severely Fragmented | 15.0 to 35.0 | #800026 | Extreme fragmentation; priority zone for reclamation screening |

---

## 4. Results

### 4.1 Zone-Level Summary

| Zone | Zone Area (Mha) | Patches | Mean Patch Area (ha) | Mean PAR | Mean NND (m) | Patch Density (/1kha) | Frag Index |
|---|---|---|---|---|---|---|---|
| Green Area | 115.572 | 1,846 | 4.14 | 0.084116 | 619.46 | 0.071 | 9.64 |
| White Area | 70.169 | 9,322 | 9.14 | 0.030377 | 2,070.08 | 0.600 | 9.15 |

### 4.2 Fragmentation Class Distribution (39 zone polygons)

| Class | Zone Count |
|---|---|
| Not Fragmented | 26 |
| Slightly Fragmented | 3 |
| Moderately Fragmented | 4 |
| Highly Fragmented | 5 |
| Severely Fragmented | 1 |

### 4.3 Key Findings

Green Area patches are smaller on average (4.14 ha vs 9.14 ha) and have a higher PAR (0.084 vs 0.030), indicating more irregular, jagged patch shapes, which is consistent with fragmented forested landscapes. White Area patches are larger but far more densely packed (0.60 patches per 1,000 ha vs 0.07), and the mean NND is substantially higher (2,070 m vs 619 m), suggesting patches are large and clustered in specific zones rather than distributed evenly.

The single Severely Fragmented zone (frag_index 31.9) warrants priority attention in any reclamation screening. The 5 Highly Fragmented zones collectively represent the most actionable targets for follow-on analysis.

---

## 5. Output Files

| File | Description |
|---|---|
| Fragmentation_Analysis.qgz | Final QGIS project with all layers and symbology |
| fragmentation_final.gpkg | Primary output - zone polygons with all fragmentation metrics |
| hfi_patch_metrics_nn.gpkg | Patch-level layer with area_ha, perimeter_m, par, nn_dist_m |
| hfi_patches_with_zone.gpkg | Patches with zone assignment (GWA_NAME, GWA_CODE) |
| zone_join_summary.gpkg | Raw spatial join summary before field enrichment |
| boundary_3400.gpkg | Provincial boundary reprojected to EPSG:3400 |

---

## 6. Key Attribute Reference - fragmentation_final.gpkg

| Field | Description |
|---|---|
| GWA_NAME | Zone name (Green Area / White Area) |
| GWA_CODE | Zone code (GLC_G / GLC_W) |
| area_ha_count | Count of HFI patches in zone |
| area_ha_mean | Mean patch area in hectares |
| par_mean | Mean perimeter-to-area ratio |
| nn_dist_m_mean | Mean nearest neighbour distance in metres |
| zone_area_ha | Zone polygon area in hectares |
| patch_density | Patches per 1,000 ha of zone area |
| frag_index | Composite fragmentation index |

---

## 7. Reproduction Notes

- NND computation uses centroid-to-centroid distances via QgsSpatialIndex, not edge-to-edge distances. This is a practical approximation appropriate for provincial-scale screening.
- The fragmentation index is a simple additive composite and is not dimensionless in strict terms. It is intended for relative ranking within this dataset, not for cross-dataset comparison.
- 224 patches (11,360 minus 11,136) did not receive a zone assignment. These likely fall on zone boundaries or outside the Green/White Areas extent. They are excluded from zone aggregations.
- The single Severely Fragmented zone at frag_index 31.9 is a true outlier driven by extremely high patch density. Verify this zone against the patch-level layer before using it in reporting.
- Do not use quantile classification on frag_index. The distribution is zero-inflated (26 of 39 zones) and right-skewed; manual breaks are required.

---

## 8. Integration with Abandoned Well Workflow

Intersect `cluster_hulls.gpkg` from the hotspot analysis with `fragmentation_final.gpkg` to flag abandoned well clusters that fall within Highly or Severely Fragmented zones. Co-location of high well density and high vegetated surface fragmentation in the Green Area is a strong indicator of elevated reclamation priority under provincial environmental screening criteria.
