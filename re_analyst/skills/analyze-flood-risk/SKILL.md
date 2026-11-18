---
name: analyze-flood-risk
description: Investigates subject and neighboring property flood zones, drainage conditions, and environmental risks.
metadata:
  adk_additional_tools:
    - get_lot_data
---

# Analyze Flood Risk Skill

When evaluating environmental or geographic risks for a lot:

1. Call `get_lot_data` using the subject property's lot ID.
2. Identify and report the subject property's exact `flood_zone` classification.
3. Read the `adjacent_parcels` field and identify every neighboring lot ID.
4. Call `get_lot_data` separately for EVERY lot ID in `adjacent_parcels`. Do not skip any neighboring property.
5. For each neighboring property, retrieve and report:
   - Lot ID.
   - Exact FEMA flood zone classification.
   - Topography.
   - Soil classification.
   - Drainage easements or other drainage-related information, if available.
6. Compare each neighboring property's flood zone with the subject property's flood zone.
7. Analyze potential shared hydrological risks:
   - Nearby flood-zone exposure.
   - Potential runoff and drainage interactions.
   - Topography and soil conditions that may influence drainage.
   - Drainage easements and their potential implications.
   - Possible effects on the subject property's development feasibility.
8. Distinguish verified property data from potential hydrological implications. Never claim a specific runoff direction without supporting data.
9. If a neighboring property cannot be retrieved or its flood zone is unavailable, explicitly report the limitation.
10. Include the subject property and every retrieved neighboring property in the final Risk Assessment section, with their flood classifications and comparative hydrological analysis.
