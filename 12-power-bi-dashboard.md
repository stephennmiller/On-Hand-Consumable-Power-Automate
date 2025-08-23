# Power BI Dashboard for On-Hand Consumable Performance Monitoring

## Executive Summary

This comprehensive Power BI dashboard provides real-time visibility into the On-Hand Consumable system performance, enabling data-driven decisions and proactive issue resolution through interactive visualizations and automated alerts.

## Dashboard Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Power BI Dashboard Structure                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Executive Dashboard                    │    │
│  │  KPI Cards │ Gauge Charts │ Performance Trends     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            Operational Dashboard                    │    │
│  │  Flow Status │ Error Analysis │ Queue Monitoring   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │             Inventory Dashboard                     │    │
│  │  Stock Levels │ Movement Analysis │ Predictions    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            Analytics Dashboard                      │    │
│  │  Trends │ Patterns │ Anomalies │ Forecasting      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## SharePoint List Schemas for Metrics

### 1. Real-Time Metrics List
```
List Name: PowerBI_RealTimeMetrics
Columns:
- Title: Single line of text
- MetricTimestamp: Date and Time (Indexed)
- FlowName: Single line of text (Indexed)
- MetricType: Choice (Performance, Error, Volume, Resource)
- MetricValue: Number
- MetricUnit: Choice (Milliseconds, Count, Percentage, MB)
- Status: Choice (Normal, Warning, Critical)
- TrendDirection: Choice (Up, Down, Stable)
- DailyAverage: Number
- WeeklyAverage: Number
- MonthlyAverage: Number
- AnomalyDetected: Yes/No
- AnomalyScore: Number (0-100)
```

### 2. Flow Execution History List
```
List Name: PowerBI_FlowExecutionHistory
Columns:
- Title: Single line of text
- ExecutionID: Single line of text (Indexed)
- FlowName: Single line of text (Indexed)
- StartTime: Date and Time (Indexed)
- EndTime: Date and Time
- Duration: Number (milliseconds)
- RecordsProcessed: Number
- RecordsSucceeded: Number
- RecordsFailed: Number
- ErrorCount: Number
- WarningCount: Number
- SuccessRate: Number (percentage)
- PerformanceScore: Number (0-100)
- CostEstimate: Currency
```

### 3. Inventory Snapshot List
```
List Name: PowerBI_InventorySnapshot
Columns:
- Title: Single line of text
- SnapshotTime: Date and Time (Indexed)
- ItemNumber: Single line of text (Indexed)
- ItemDescription: Single line of text
- Category: Choice (Managed by lookup)
- OnHandQuantity: Number
- ReservedQuantity: Number
- AvailableQuantity: Number
- MinimumStock: Number
- MaximumStock: Number
- ReorderPoint: Number
- StockStatus: Choice (Normal, Low, Critical, Excess)
- LastMovement: Date and Time
- TurnoverRate: Number
- Value: Currency
```

### 4. Alert History List
```
List Name: PowerBI_AlertHistory
Columns:
- Title: Single line of text
- AlertTime: Date and Time (Indexed)
- AlertType: Choice (Performance, Error, Threshold, System)
- Severity: Choice (Low, Medium, High, Critical)
- Component: Single line of text
- Message: Multiple lines of text
- MetricValue: Number
- ThresholdValue: Number
- ResponseTime: Number (minutes)
- ResolutionTime: Number (minutes)
- ResolutionStatus: Choice (Resolved, Pending, Escalated)
- RootCause: Multiple lines of text
```

## Dashboard Pages and Visualizations

### Page 1: Executive Dashboard

#### KPI Section (Top Row)
```
Card 1: System Health Score
Measure: 
SystemHealthScore = 
VAR PerformanceScore = [AveragePerformanceScore]
VAR ErrorScore = 100 - ([ErrorRate] * 100)
VAR AvailabilityScore = [UptimePercentage]
RETURN (PerformanceScore + ErrorScore + AvailabilityScore) / 3

Conditional Formatting:
- >= 90: Green
- >= 70: Yellow
- < 70: Red
```

```
Card 2: Daily Processing Volume
Measure:
DailyVolume = 
CALCULATE(
    SUM(FlowExecutionHistory[RecordsProcessed]),
    DATESINPERIOD(
        FlowExecutionHistory[StartTime],
        TODAY(),
        -1,
        DAY
    )
)

Target Line: Previous 7-day average
```

```
Card 3: Current Queue Depth
Measure:
CurrentQueueDepth = 
VAR LatestSnapshot = MAX(RealTimeMetrics[MetricTimestamp])
RETURN 
CALCULATE(
    SUM(RealTimeMetrics[MetricValue]),
    RealTimeMetrics[MetricType] = "Volume",
    RealTimeMetrics[MetricTimestamp] = LatestSnapshot
)

Gauge Visual:
- Maximum: 10000
- Target: 1000
- Ranges: 0-1000 (Green), 1000-5000 (Yellow), 5000+ (Red)
```

```
Card 4: Average Processing Time
Measure:
AvgProcessingTime = 
AVERAGEX(
    FILTER(
        FlowExecutionHistory,
        FlowExecutionHistory[StartTime] >= TODAY()
    ),
    FlowExecutionHistory[Duration] / 1000
)

Format: #,##0.0 "seconds"
Sparkline: Last 24 hours trend
```

#### Performance Trends (Middle Section)
```
Line Chart: Processing Performance Over Time
X-Axis: Date/Time (Continuous)
Y-Axis: Duration (seconds)
Legend: Flow Names
Lines:
- Average Duration
- 95th Percentile
- 5th Percentile

DAX for 95th Percentile:
Duration95thPercentile = 
PERCENTILE.EXC(
    FlowExecutionHistory[Duration],
    0.95
) / 1000
```

```
Area Chart: Volume Trends
X-Axis: Date/Time
Y-Axis: Records Processed
Areas:
- Successful Records (Green)
- Failed Records (Red)
- Pending Records (Yellow)

Include forecast using:
Forecast = 
VAR LastValue = 
    CALCULATE(
        AVERAGE(FlowExecutionHistory[RecordsProcessed]),
        DATESINPERIOD(Date[Date], MAX(Date[Date]), -7, DAY)
    )
VAR GrowthRate = 1.02 -- 2% daily growth assumption
RETURN LastValue * POWER(GrowthRate, DATEDIFF(MAX(Date[Date]), Date[Date], DAY))
```

#### Error Analysis (Bottom Section)
```
Donut Chart: Error Distribution
Values: Error Count by Category
Categories:
- Timeout Errors
- Data Validation Errors
- Connection Errors
- Resource Errors
- Other

Drill-through to detailed error log
```

```
Heat Map: Error Frequency by Hour and Day
Rows: Day of Week
Columns: Hour of Day
Values: Error Count
Color Scale: White (0) to Dark Red (Max)

Pattern Detection:
PatternScore = 
VAR HourlyAvg = AVERAGEX(ALL(Time[Hour]), [ErrorCount])
VAR DailyAvg = AVERAGEX(ALL(Date[DayOfWeek]), [ErrorCount])
VAR CurrentValue = [ErrorCount]
RETURN 
IF(
    CurrentValue > HourlyAvg * 1.5 && CurrentValue > DailyAvg * 1.5,
    "Pattern Detected",
    "Normal"
)
```

### Page 2: Operational Dashboard

#### Flow Status Matrix
```
Matrix Visual: Flow Execution Status
Rows: Flow Names
Columns: Time Buckets (Last Hour, Last 24H, Last 7D, Last 30D)
Values: 
- Success Rate (%)
- Average Duration
- Total Runs
- Error Count

Conditional Formatting:
Success Rate:
- >= 95%: Green
- >= 80%: Yellow  
- < 80%: Red
```

#### Real-Time Monitoring
```
Streaming Dataset Configuration:
{
  "name": "RealTimeFlowMetrics",
  "defaultMode": "pushStreaming",
  "properties": {
    "timestamp": "datetime",
    "flowName": "string",
    "status": "string",
    "duration": "number",
    "recordsProcessed": "number"
  }
}

Line Chart: Real-Time Performance
X-Axis: Timestamp (streaming, 5-minute window)
Y-Axis: Duration
Auto-refresh: Every 1 second
```

#### Queue Analytics
```
Waterfall Chart: Queue Processing Pipeline
Categories:
- Starting Queue
+ New Arrivals
- Processed Successfully
- Failed (retry pending)
- Failed (dead letter)
= Ending Queue

Measure for Queue Velocity:
QueueVelocity = 
VAR ProcessedCount = [RecordsProcessed]
VAR TimeWindow = DATEDIFF(MIN(Date[Date]), MAX(Date[Date]), HOUR)
RETURN DIVIDE(ProcessedCount, TimeWindow)
```

#### Resource Utilization
```
Combo Chart: Resource Usage vs Performance
X-Axis: Time
Line Y-Axis: API Calls Used (% of limit)
Column Y-Axis: Processing Time

Resource Efficiency Score:
ResourceEfficiency = 
VAR RecordsPerAPICall = DIVIDE([RecordsProcessed], [APICallsUsed])
VAR OptimalRatio = 100 -- Target records per API call
RETURN MIN(DIVIDE(RecordsPerAPICall, OptimalRatio) * 100, 100)
```

### Page 3: Inventory Dashboard

#### Stock Level Overview
```
Treemap: Inventory Distribution
Size: On-Hand Quantity
Color: Stock Status (Critical=Red, Low=Yellow, Normal=Green, Excess=Blue)
Categories: Item Categories
Details: Item Number, Description

Drill-through enabled to item history
```

#### Movement Analysis
```
Scatter Chart: Inventory Turnover Analysis
X-Axis: Average On-Hand Quantity
Y-Axis: Turnover Rate
Size: Item Value
Color: Category
Play Axis: Date (for animation)

Quadrant Analysis:
- High Turnover, Low Stock: Fast Movers
- High Turnover, High Stock: Review Stock Levels
- Low Turnover, Low Stock: Slow but Controlled
- Low Turnover, High Stock: Excess Inventory
```

#### Stock Predictions
```
Line Chart with Forecast: Predicted Stock Levels
X-Axis: Date (extended 30 days)
Y-Axis: Quantity
Lines: 
- Actual On-Hand
- Predicted On-Hand
- Minimum Stock Level
- Reorder Point

Prediction Model (DAX):
PredictedStock = 
VAR HistoricalConsumption = 
    CALCULATE(
        AVERAGE(DailyConsumption[Quantity]),
        DATESINPERIOD(Date[Date], MAX(Date[Date]), -30, DAY)
    )
VAR CurrentStock = [CurrentOnHand]
VAR DaysAhead = DATEDIFF(MAX(Date[Date]), Date[Date], DAY)
RETURN CurrentStock - (HistoricalConsumption * DaysAhead)
```

#### Critical Items Dashboard
```
Table: Items Requiring Attention
Columns:
- Item Number
- Description
- Current Stock
- Days Until Stockout
- Recommended Order Quantity
- Priority Score

Priority Score Calculation:
PriorityScore = 
VAR DaysToStockout = [DaysUntilStockout]
VAR ItemCriticality = RELATED(Items[Criticality])
VAR ConsumptionVariability = [ConsumptionStdDev] / [ConsumptionAvg]
RETURN 
    (100 / MAX(DaysToStockout, 1)) * ItemCriticality * (1 + ConsumptionVariability)

Conditional Formatting:
- Priority > 80: Red background
- Priority > 50: Yellow background
- Priority <= 50: Green background
```

### Page 4: Analytics Dashboard

#### Trend Analysis
```
Decomposition Tree: Performance Breakdown
Analyze: Average Processing Time
By:
- Flow Name
- Time Period
- Data Volume Category
- Error Occurrence

AI Insights:
- Anomaly detection
- Key influencers
- What-if parameters
```

#### Pattern Recognition
```
Line and Clustered Column Chart: Seasonal Patterns
X-Axis: Day of Month
Line Values: Current Month Performance
Column Values: Historical Average (same day, previous months)
Secondary Line: Standard Deviation Bands

Seasonality Index:
SeasonalityIndex = 
VAR CurrentDayOfMonth = DAY(TODAY())
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(Metrics[Value]),
        DAY(Metrics[Date]) = CurrentDayOfMonth,
        Metrics[Date] < TODAY()
    )
VAR CurrentValue = [CurrentMetricValue]
RETURN DIVIDE(CurrentValue, HistoricalAvg)
```

#### Anomaly Detection
```
Scatter Chart: Anomaly Detection
X-Axis: Expected Value (based on model)
Y-Axis: Actual Value
Color: Anomaly Score
Size: Impact (deviation * volume)

Anomaly Detection (DAX):
AnomalyScore = 
VAR Mean = AVERAGE(Metrics[Value])
VAR StdDev = STDEV.P(Metrics[Value])
VAR CurrentValue = Metrics[Value]
VAR ZScore = DIVIDE(CurrentValue - Mean, StdDev)
RETURN 
    SWITCH(
        TRUE(),
        ABS(ZScore) > 3, "Critical Anomaly",
        ABS(ZScore) > 2, "Moderate Anomaly",
        ABS(ZScore) > 1.5, "Minor Anomaly",
        "Normal"
    )
```

#### Predictive Analytics
```
Key Influencers Visual: Failure Prediction
Analyze: Likelihood of Flow Failure
Factors:
- Time of Day
- Day of Week
- Queue Length
- Previous Run Duration
- Error Count in Last Hour
- System Resource Usage

What-If Parameters:
Queue Length Slider: 0 to 10000
Resource Usage Slider: 0% to 100%
Show impact on predicted failure rate
```

## Real-Time Data Connection Setup

### Power Automate to Power BI Streaming

#### Step 1: Create Streaming Dataset
```json
POST https://api.powerbi.com/v1.0/myorg/datasets
{
  "name": "OnHandConsumableRealTime",
  "defaultMode": "pushStreaming",
  "tables": [
    {
      "name": "RealTimeMetrics",
      "columns": [
        {"name": "timestamp", "dataType": "DateTime"},
        {"name": "flowName", "dataType": "String"},
        {"name": "metricType", "dataType": "String"},
        {"name": "metricValue", "dataType": "Double"},
        {"name": "status", "dataType": "String"}
      ]
    }
  ]
}
```

#### Step 2: Power Automate Push Data Flow
```
Action: HTTP Request to Power BI
Method: POST
URI: https://api.powerbi.com/v1.0/myorg/datasets/{datasetId}/rows
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json
Body:
{
  "rows": [
    {
      "timestamp": "@{utcNow()}",
      "flowName": "@{workflow()['name']}",
      "metricType": "Performance",
      "metricValue": @{variables('ProcessingTime')},
      "status": "@{variables('FlowStatus')}"
    }
  ]
}
```

### SharePoint List Connection

#### Step 1: Configure SharePoint Data Source
```
Power BI Desktop:
1. Get Data > SharePoint List
2. Site URL: https://yourcompany.sharepoint.com/sites/OnHandConsumable
3. Authentication: Organizational Account
4. Select Lists:
   - PowerBI_RealTimeMetrics
   - PowerBI_FlowExecutionHistory
   - PowerBI_InventorySnapshot
   - PowerBI_AlertHistory
```

#### Step 2: Configure Incremental Refresh
```
Power Query M:
let
    Source = SharePoint.Tables("https://yourcompany.sharepoint.com/sites/OnHandConsumable"),
    FilteredData = Table.SelectRows(Source, each [Modified] >= RangeStart and [Modified] < RangeEnd)
in
    FilteredData

Incremental Refresh Policy:
- Historical Data: 12 months
- Incremental Data: 1 day
- Refresh Frequency: Every hour
```

## Alert Configuration

### Metric-Based Alerts

#### Alert 1: Performance Degradation
```
Condition: Average Processing Time > 5 minutes
Threshold: 300000 milliseconds
Check Frequency: Every 5 minutes
Actions:
- Send email to Operations team
- Post to Teams channel
- Create ServiceNow incident (if critical)

DAX Alert Measure:
PerformanceAlert = 
VAR CurrentAvg = [AvgProcessingTime]
VAR Threshold = 300000
RETURN 
IF(
    CurrentAvg > Threshold,
    CONCATENATE("ALERT: Processing time ", 
        CONCATENATE(FORMAT(CurrentAvg/1000, "#,##0"), " seconds")),
    "Normal"
)
```

#### Alert 2: Error Rate Spike
```
Condition: Error Rate > 5% in last hour
Calculation:
ErrorRateAlert = 
VAR LastHourErrors = 
    CALCULATE(
        COUNT(FlowExecutionHistory[ExecutionID]),
        FlowExecutionHistory[ErrorCount] > 0,
        DATESINPERIOD(FlowExecutionHistory[StartTime], NOW(), -1, HOUR)
    )
VAR LastHourTotal = 
    CALCULATE(
        COUNT(FlowExecutionHistory[ExecutionID]),
        DATESINPERIOD(FlowExecutionHistory[StartTime], NOW(), -1, HOUR)
    )
VAR ErrorRate = DIVIDE(LastHourErrors, LastHourTotal) * 100
RETURN 
IF(ErrorRate > 5, "CRITICAL: Error rate " & FORMAT(ErrorRate, "#.#%"), "Normal")
```

#### Alert 3: Inventory Critical Level
```
Condition: Any item below minimum stock
Check: Every 30 minutes
Alert Details:
- Item number and description
- Current quantity
- Minimum required
- Estimated stockout time

DAX for Critical Items:
CriticalInventoryCount = 
COUNTROWS(
    FILTER(
        InventorySnapshot,
        InventorySnapshot[OnHandQuantity] < InventorySnapshot[MinimumStock]
    )
)
```

### Anomaly-Based Alerts

#### Statistical Anomaly Detection
```
Anomaly Alert Configuration:
AnomalyDetected = 
VAR CurrentValue = [CurrentMetric]
VAR ExpectedValue = [PredictedValue]
VAR Tolerance = [StandardDeviation] * 2
RETURN 
IF(
    ABS(CurrentValue - ExpectedValue) > Tolerance,
    TRUE,
    FALSE
)

Alert Actions:
- Log anomaly details
- Trigger investigation workflow
- Update anomaly dashboard
```

## Visual Layout Specifications

### Grid System
```
Dashboard Grid: 12 columns x 8 rows
Standard Spacing: 10px between visuals

Executive Dashboard Layout:
Row 1: KPI Cards (3 units each)
Row 2-4: Performance Trends (full width)
Row 5-6: Error Analysis (6 units) | Queue Status (6 units)
Row 7-8: Alerts Table (full width)
```

### Color Scheme
```
Primary Colors:
- Success: #28a745
- Warning: #ffc107
- Danger: #dc3545
- Info: #17a2b8
- Neutral: #6c757d

Background:
- Page: #f8f9fa
- Cards: #ffffff
- Headers: #343a40
```

### Typography
```
Fonts:
- Headers: Segoe UI Semibold, 16pt
- KPI Values: Segoe UI Light, 36pt
- Body Text: Segoe UI, 10pt
- Table Text: Segoe UI, 9pt
```

## Performance Optimization

### Data Model Optimization
```
1. Use Star Schema:
   - Fact Tables: Metrics, Executions
   - Dimension Tables: Date, Time, Flows, Items

2. Create Relationships:
   - One-to-Many from Dimensions to Facts
   - Use single direction where possible

3. Optimize DAX:
   - Use variables to store repeated calculations
   - Avoid FILTER when possible
   - Use CALCULATE wisely
```

### Query Optimization
```
Power Query Best Practices:
1. Filter early in the query
2. Remove unnecessary columns
3. Use native query folding
4. Aggregate at source when possible

Example:
let
    Source = SharePoint.Tables(SiteUrl),
    Filtered = Table.SelectRows(Source, each [Date] >= Date.AddMonths(DateTime.Date(DateTime.LocalNow()), -3)),
    RemovedColumns = Table.RemoveColumns(Filtered, {"Column1", "Column2"}),
    Aggregated = Table.Group(RemovedColumns, {"FlowName"}, {{"Count", each Table.RowCount(_), type number}})
in
    Aggregated
```

### Refresh Optimization
```
Incremental Refresh Configuration:
1. Historical Range: 12 months
2. Incremental Range: 1 day
3. Detect Data Changes: On
4. Only Refresh Complete Days: Off

Partition Strategy:
- Current Day: Every hour
- Last 7 Days: Daily at 2 AM
- Last Month: Weekly on Sunday
- Historical: Monthly on 1st
```

## Mobile Layout

### Phone Layout Configuration
```
Key Metrics View:
- System Health Score (full width)
- Current Queue Depth (full width)
- Error Count Today (full width)
- Processing Time Trend (full width)

Navigation:
- Bottom tab bar
- Swipe between pages
- Pinch to zoom on charts
```

### Responsive Design
```
Breakpoints:
- Desktop: > 1024px
- Tablet: 768px - 1024px
- Phone: < 768px

Adaptive Visuals:
- Cards stack vertically on mobile
- Tables become scrollable
- Charts simplify (fewer data points)
```

## Testing and Validation

### Dashboard Testing Checklist
```
Data Accuracy:
□ Verify calculations match source data
□ Test all DAX measures
□ Validate aggregations
□ Check date/time conversions

Performance:
□ Page load time < 3 seconds
□ Refresh time < 30 seconds
□ Interactive response < 1 second

Functionality:
□ All filters work correctly
□ Drill-through functions properly
□ Bookmarks navigate correctly
□ Export functions work

Visual Design:
□ Consistent formatting
□ Readable on all devices
□ Color accessibility compliance
□ Print layout works
```

### User Acceptance Testing
```
Test Scenarios:
1. Executive views morning dashboard
2. Operations investigates error spike
3. Inventory manager checks critical items
4. Analyst performs trend analysis
5. Mobile user checks system status

Success Criteria:
- Users find needed information < 30 seconds
- No training required for basic use
- Positive feedback on clarity
- Actionable insights identified
```

## Deployment Guide

### Step 1: Development Environment
```
1. Install Power BI Desktop (latest version)
2. Connect to development SharePoint
3. Build dashboard iteratively
4. Test with sample data
5. Validate calculations
```

### Step 2: Workspace Setup
```
1. Create Power BI Workspace
   Name: OnHandConsumable-Analytics
   License: Premium (for streaming)
   
2. Configure permissions:
   - Admins: Full control
   - Contributors: Edit reports
   - Viewers: Read-only access
```

### Step 3: Publish and Configure
```
1. Publish from Power BI Desktop
2. Configure dataset refresh schedule
3. Set up streaming dataset
4. Configure alerts
5. Create app for distribution
```

### Step 4: Production Deployment
```
1. Final testing in production
2. User training sessions
3. Documentation distribution
4. Go-live communication
5. Monitor adoption metrics
```

## Maintenance Procedures

### Daily Maintenance
```
□ Check refresh status
□ Review alert logs
□ Verify streaming data flow
□ Monitor performance metrics
```

### Weekly Maintenance
```
□ Review usage analytics
□ Check for failed refreshes
□ Update documentation
□ Optimize slow queries
```

### Monthly Maintenance
```
□ Review and adjust alerts
□ Update calculations if needed
□ Archive old data
□ Performance optimization
□ User feedback review
```

This comprehensive Power BI dashboard provides complete visibility into your On-Hand Consumable system, enabling proactive management and continuous optimization.