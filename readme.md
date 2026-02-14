A BI training project showing issue intake, resolution throughput, and cumulative backlog implemented with both Power BI & SQL/Postgres

# CSV version

## Data Model
Issues table (main source for dashboard)
Pages table
Reviews table
Date table for time modeling

## Key Metrics
Monthly Issues Identified
Monthly Issues Resolved
Cumulative Open Backlog

## Technical Notes
Used DAX to handle inactive relationships
Implemented cumulative backlog logic
Identified and corrected misleading secondary-axis scaling

## What I Learned
Filter context vs row context
Date dimension design
Visual scaling distortions can mislead interpretation

# SQL version
Goal: do all of the above, but with SQL as the data source and logic