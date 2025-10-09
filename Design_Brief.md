# Design Brief

## Scenario
We chose the Cross-Border Health Analytics scenario. The purpose of our application is to collect user inputted health symptoms and alert users when there is a health concern.

## Data Source/Volume
The primary data source is user inputted health symptoms, which is stored in JSON format. Volume TBA.

## Batch Schedule
Data will be ingested at 00:00 local time every day, due by 06:00.

## SLA
* p95 < 2.5 hours, even for retries
* Cost < $16/daily run, for a total of $500/month
* There are no strict regulatory deadlines