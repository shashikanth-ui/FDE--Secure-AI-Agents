---
name: analyze-opportunity
description: Investigates a land parcel for real estate investment opportunity analysis, including acquisition cost-per-acre calculation.
metadata:
  adk_additional_tools:
    - get_lot_data
---

# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity:

1. Use `get_lot_data` to retrieve the subject property's records using its lot ID.
2. Review acreage, last sale price, zoning, parcel APN, and other relevant details.
3. Calculate acquisition cost per acre using:

   Cost per acre = Last sale price / Acreage

4. Explicitly show the sale price, acreage, formula, and calculated result in dollars per acre, formatted to two decimal places.
5. Highlight cost per acre as a core acquisition valuation metric in both the Executive Summary and Acquisition Basis and Holding Costs sections.
6. If sale price or acreage is missing, invalid, or zero, state that the calculation cannot be completed. Never invent values.
7. Invoke the `analyze-flood-risk` skill to investigate the subject property's flood zone and all adjacent properties.
8. Invoke the `analyze-tax-implications` skill to investigate property tax holding costs using the parcel APN.
9. Integrate the findings into the final investment assessment, distinguishing retrieved facts from analysis and recommendations.
