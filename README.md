# Microsoft Sentinel Ingest Planner
I built this small web app to help plan out Sentinel deployment ingest, as the Azure pricing calculator does get quite in-depth enough for truly planning out data lake and analytic commitment tiers. The app will automatically calculate costs based on the data sources you input as well as the commitment tier you select.

Click the `Add Log Source` button to add a new log source. Provide a name and planned ingest in megabytes (MB), as well as if you want to ingest this to the analytic tier or the data lake tier.

## Planned Updates
- Add in calculation for the Sentinel E5 benefit
- Add in retianed storage calculation for data lake tier log sources