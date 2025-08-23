# Monitoring and Alerting Flows - Complete Implementation Guide

## Overview

This guide provides production-ready monitoring and alerting flows that ensure 24/7 visibility into the On-Hand Consumable system performance, enabling proactive issue resolution and maintaining 99.9% uptime.

## Architecture Components

```
┌─────────────────────────────────────────────┐
│        Monitoring Infrastructure            │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐    ┌──────────────┐     │
│  │ Performance  │    │    Error     │     │
│  │  Monitoring  │    │   Alerting   │     │
│  └──────┬───────┘    └──────┬───────┘     │
│         │                    │              │
│  ┌──────▼───────┐    ┌──────▼───────┐     │
│  │  Threshold   │    │    Daily     │     │
│  │   Breach     │    │Health Check  │     │
│  └──────┬───────┘    └──────┬───────┘     │
│         │                    │              │
│  ┌──────▼────────────────────▼───────┐     │
│  │   Real-Time Dashboard Updates     │     │
│  └────────────────────────────────────┘     │
│                                             │
└─────────────────────────────────────────────┘
```

## Required SharePoint Lists

### 1. Performance Metrics List
```
Columns:
- Title: Single line of text
- FlowName: Single line of text (Indexed)
- RunID: Single line of text
- StartTime: Date and Time
- EndTime: Date and Time
- Duration: Number (milliseconds)
- RecordsProcessed: Number
- RecordsFailed: Number
- MemoryUsage: Number (MB)
- APICallCount: Number
- Status: Choice (Success, Warning, Failed)
- ErrorDetails: Multiple lines of text
- PerformanceScore: Number (0-100)
```

### 2. Alert Configuration List
```
Columns:
- Title: Single line of text
- AlertType: Choice (Error, Performance, Threshold, Health)
- FlowName: Single line of text
- ThresholdValue: Number
- ThresholdOperator: Choice (Greater, Less, Equal, GreaterEqual, LessEqual)
- AlertChannels: Choice (Email, Teams, Both)
- Recipients: Person or Group (Allow multiple)
- IsActive: Yes/No
- CooldownMinutes: Number
- LastAlertTime: Date and Time
```

### 3. System Health List
```
Columns:
- Title: Single line of text
- CheckType: Choice (Daily, Hourly, RealTime)
- FlowName: Single line of text
- HealthScore: Number (0-100)
- LastCheckTime: Date and Time
- ComponentStatus: Multiple lines of text (JSON)
- Issues: Multiple lines of text
- Recommendations: Multiple lines of text
```

## Flow 1: Performance Monitoring Flow

### Trigger Configuration
```
Type: Recurrence
Frequency: Every 5 minutes
Time Zone: Your local time zone
```

### Step-by-Step Implementation

#### Step 1: Initialize Variables
```json
Variable: MetricsWindow
Type: Integer
Value: 5

Variable: PerformanceData
Type: Array
Value: []

Variable: AlertsToSend
Type: Array
Value: []
```

#### Step 2: Get Recent Flow Runs
```
Action: Send an HTTP request to SharePoint
Method: GET
Uri: _api/web/lists/getbytitle('Flow Run History')/items?$filter=Created ge datetime'@{addMinutes(utcNow(), mul(-1, variables('MetricsWindow')))}'&$top=1000&$orderby=Created desc
Headers:
  Accept: application/json;odata=verbose
  Content-Type: application/json;odata=verbose
```

#### Step 3: Parse and Analyze Performance Data
```
Action: Select
From: @{body('Get_Recent_Flow_Runs')['d']['results']}
Map:
{
  "FlowName": @{item()['FlowName']},
  "RunID": @{item()['RunID']},
  "Duration": @{sub(ticks(item()['EndTime']), ticks(item()['StartTime']))},
  "RecordsProcessed": @{item()['RecordsProcessed']},
  "Status": @{item()['Status']},
  "PerformanceScore": @{
    if(
      less(item()['Duration'], 60000),
      100,
      if(
        less(item()['Duration'], 120000),
        80,
        if(
          less(item()['Duration'], 300000),
          60,
          40
        )
      )
    )
  }
}
```

#### Step 4: Calculate Aggregated Metrics
```
Expression for Average Duration:
div(
  add(
    apply(
      body('Select_Performance_Data'),
      x => x['Duration']
    )
  ),
  length(body('Select_Performance_Data'))
)

Expression for Success Rate:
mul(
  div(
    length(
      filter(
        body('Select_Performance_Data'),
        x => equals(x['Status'], 'Success')
      )
    ),
    length(body('Select_Performance_Data'))
  ),
  100
)

Expression for Throughput:
div(
  sum(
    apply(
      body('Select_Performance_Data'),
      x => x['RecordsProcessed']
    )
  ),
  variables('MetricsWindow')
)
```

#### Step 5: Store Metrics in SharePoint
```
Action: Send an HTTP request to SharePoint
Method: POST
Uri: _api/web/lists/getbytitle('Performance Metrics')/items
Headers:
  Accept: application/json;odata=verbose
  Content-Type: application/json;odata=verbose
  X-RequestDigest: @{body('Get_Form_Digest')['d']['GetContextWebInformation']['FormDigestValue']}
Body:
{
  "__metadata": {"type": "SP.Data.PerformanceMetricsListItem"},
  "Title": "Metrics @{utcNow()}",
  "FlowName": "All Flows",
  "StartTime": "@{addMinutes(utcNow(), mul(-1, variables('MetricsWindow')))}",
  "EndTime": "@{utcNow()}",
  "Duration": @{variables('AverageDuration')},
  "RecordsProcessed": @{variables('TotalRecords')},
  "Status": "@{if(greater(variables('SuccessRate'), 95), 'Success', if(greater(variables('SuccessRate'), 80), 'Warning', 'Failed'))}",
  "PerformanceScore": @{variables('OverallScore')}
}
```

#### Step 6: Check Thresholds
```
Action: Get items from SharePoint
List: Alert Configuration
Filter: IsActive eq true and AlertType eq 'Performance'

Action: Apply to each alert configuration
  Condition: Check if threshold breached
  Expression: 
    switch(
      items('Apply_to_each')['ThresholdOperator'],
      'Greater', greater(variables('CurrentMetric'), items('Apply_to_each')['ThresholdValue']),
      'Less', less(variables('CurrentMetric'), items('Apply_to_each')['ThresholdValue']),
      'Equal', equals(variables('CurrentMetric'), items('Apply_to_each')['ThresholdValue']),
      false
    )
  
  If Yes: Add to AlertsToSend array
```

## Flow 2: Error Alerting Flow

### Trigger Configuration
```
Type: When an item is created or modified
List: Flow Run History
Filter: Status eq 'Failed'
```

### Step-by-Step Implementation

#### Step 1: Get Error Details
```
Action: Get item
List: Flow Run History
Id: @{triggerBody()?['ID']}
```

#### Step 2: Analyze Error Pattern
```
Expression for Error Category:
if(
  contains(body('Get_item')['ErrorDetails'], 'timeout'),
  'Timeout',
  if(
    contains(body('Get_item')['ErrorDetails'], 'throttl'),
    'Throttling',
    if(
      contains(body('Get_item')['ErrorDetails'], 'auth'),
      'Authentication',
      'General'
    )
  )
)

Expression for Severity:
if(
  contains(body('Get_item')['FlowName'], 'Critical'),
  'High',
  if(
    greater(body('Get_item')['RecordsFailed'], 100),
    'High',
    if(
      greater(body('Get_item')['RecordsFailed'], 10),
      'Medium',
      'Low'
    )
  )
)
```

#### Step 3: Check Alert Cooldown
```
Action: Get items
List: Alert Configuration
Filter: FlowName eq '@{body('Get_item')['FlowName']}' and AlertType eq 'Error'

Condition: Check if cooldown expired
Expression:
greater(
  ticks(utcNow()),
  ticks(addMinutes(items('Apply_to_each')['LastAlertTime'], items('Apply_to_each')['CooldownMinutes']))
)
```

#### Step 4: Send Teams Alert
```
Action: Post adaptive card in Teams channel
Team: Operations
Channel: Alerts
Card:
{
  "type": "AdaptiveCard",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "⚠️ Flow Error Alert",
      "weight": "Bolder",
      "size": "Large",
      "color": "Attention"
    },
    {
      "type": "FactSet",
      "facts": [
        {
          "title": "Flow Name:",
          "value": "@{body('Get_item')['FlowName']}"
        },
        {
          "title": "Run ID:",
          "value": "@{body('Get_item')['RunID']}"
        },
        {
          "title": "Error Category:",
          "value": "@{variables('ErrorCategory')}"
        },
        {
          "title": "Severity:",
          "value": "@{variables('Severity')}"
        },
        {
          "title": "Records Failed:",
          "value": "@{body('Get_item')['RecordsFailed']}"
        },
        {
          "title": "Time:",
          "value": "@{utcNow()}"
        }
      ]
    },
    {
      "type": "TextBlock",
      "text": "Error Details:",
      "weight": "Bolder"
    },
    {
      "type": "TextBlock",
      "text": "@{body('Get_item')['ErrorDetails']}",
      "wrap": true,
      "fontType": "Monospace"
    },
    {
      "type": "ActionSet",
      "actions": [
        {
          "type": "Action.OpenUrl",
          "title": "View Flow Run",
          "url": "https://make.powerautomate.com/environments/@{workflow()['tags']['environmentName']}/flows/@{body('Get_item')['FlowID']}/runs/@{body('Get_item')['RunID']}"
        },
        {
          "type": "Action.OpenUrl",
          "title": "View Dashboard",
          "url": "[Your Power BI Dashboard URL]"
        }
      ]
    }
  ]
}
```

#### Step 5: Send Email Alert
```
Action: Send an email (V2)
To: @{join(body('Get_Alert_Recipients'), ';')}
Subject: 🚨 Flow Error: @{body('Get_item')['FlowName']} - @{variables('Severity')} Severity
Body:
<html>
<body style="font-family: Arial, sans-serif;">
  <div style="background-color: #ff4444; color: white; padding: 10px;">
    <h2>Flow Error Alert</h2>
  </div>
  <div style="padding: 20px;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <td style="font-weight: bold; padding: 5px;">Flow Name:</td>
        <td>@{body('Get_item')['FlowName']}</td>
      </tr>
      <tr>
        <td style="font-weight: bold; padding: 5px;">Run ID:</td>
        <td>@{body('Get_item')['RunID']}</td>
      </tr>
      <tr>
        <td style="font-weight: bold; padding: 5px;">Error Category:</td>
        <td>@{variables('ErrorCategory')}</td>
      </tr>
      <tr>
        <td style="font-weight: bold; padding: 5px;">Severity:</td>
        <td>@{variables('Severity')}</td>
      </tr>
      <tr>
        <td style="font-weight: bold; padding: 5px;">Records Failed:</td>
        <td>@{body('Get_item')['RecordsFailed']}</td>
      </tr>
    </table>
    
    <h3>Error Details:</h3>
    <pre style="background-color: #f5f5f5; padding: 10px; border-radius: 5px;">
      @{body('Get_item')['ErrorDetails']}
    </pre>
    
    <h3>Recommended Actions:</h3>
    <ul>
      @{variables('RecommendedActions')}
    </ul>
  </div>
</body>
</html>
Importance: @{variables('Severity')}
```

## Flow 3: Threshold Breach Notifications

### Trigger Configuration
```
Type: Recurrence
Frequency: Every 15 minutes
```

### Step-by-Step Implementation

#### Step 1: Define Threshold Rules
```
Variable: ThresholdRules
Type: Array
Value:
[
  {
    "metric": "AverageProcessingTime",
    "threshold": 300000,
    "operator": "greater",
    "severity": "Warning",
    "message": "Processing time exceeds 5 minutes"
  },
  {
    "metric": "ErrorRate",
    "threshold": 5,
    "operator": "greater",
    "severity": "Critical",
    "message": "Error rate exceeds 5%"
  },
  {
    "metric": "QueueLength",
    "threshold": 1000,
    "operator": "greater",
    "severity": "Warning",
    "message": "Queue backlog exceeds 1000 items"
  },
  {
    "metric": "MemoryUsage",
    "threshold": 90,
    "operator": "greater",
    "severity": "Critical",
    "message": "Memory usage exceeds 90%"
  }
]
```

#### Step 2: Get Current Metrics
```
Action: Send an HTTP request to SharePoint
Method: GET
Uri: _api/web/lists/getbytitle('Performance Metrics')/items?$filter=Created ge datetime'@{addMinutes(utcNow(), -15)}'&$orderby=Created desc&$top=100
```

#### Step 3: Calculate Threshold Metrics
```
Expression for Average Processing Time:
div(
  sum(
    apply(
      body('Get_Current_Metrics')['d']['results'],
      x => x['Duration']
    )
  ),
  length(body('Get_Current_Metrics')['d']['results'])
)

Expression for Error Rate:
mul(
  div(
    length(
      filter(
        body('Get_Current_Metrics')['d']['results'],
        x => equals(x['Status'], 'Failed')
      )
    ),
    length(body('Get_Current_Metrics')['d']['results'])
  ),
  100
)

Expression for Queue Length:
body('Get_Queue_Status')['d']['ItemCount']

Expression for Memory Usage:
div(
  body('Get_System_Metrics')['MemoryUsed'],
  body('Get_System_Metrics')['MemoryTotal']
)
```

#### Step 4: Check Each Threshold
```
Action: Apply to each threshold rule
  Source: @{variables('ThresholdRules')}
  
  Action: Compose current metric value
  Inputs:
    switch(
      items('Apply_to_each')['metric'],
      'AverageProcessingTime', variables('AvgProcessingTime'),
      'ErrorRate', variables('ErrorRate'),
      'QueueLength', variables('QueueLength'),
      'MemoryUsage', variables('MemoryUsage'),
      0
    )
  
  Condition: Check if threshold breached
  Expression:
    switch(
      items('Apply_to_each')['operator'],
      'greater', greater(outputs('Compose_metric'), items('Apply_to_each')['threshold']),
      'less', less(outputs('Compose_metric'), items('Apply_to_each')['threshold']),
      false
    )
  
  If Yes: Send threshold breach notification
```

#### Step 5: Create Consolidated Alert
```
Action: Create HTML table
From: @{variables('BreachedThresholds')}
Columns:
- Metric
- Current Value
- Threshold
- Severity
- Message

Action: Post message in Teams channel
Channel: Operations
Message:
## ⚠️ Threshold Breach Alert

Multiple performance thresholds have been breached:

@{body('Create_HTML_table')}

**Time:** @{utcNow()}
**Environment:** Production

[View Dashboard](PowerBI_URL) | [View Logs](SharePoint_URL)
```

## Flow 4: Daily Health Check Report

### Trigger Configuration
```
Type: Recurrence
Frequency: Daily
Time: 7:00 AM
Time Zone: Your local time zone
```

### Step-by-Step Implementation

#### Step 1: Initialize Report Variables
```
Variable: ReportDate
Type: String
Value: @{formatDateTime(utcNow(), 'yyyy-MM-dd')}

Variable: HealthChecks
Type: Array
Value: []

Variable: OverallHealth
Type: Integer
Value: 100
```

#### Step 2: Check Component Health
```
Action: Parallel Branch

Branch 1: Check SharePoint Lists
  Action: Send HTTP request to SharePoint
  Uri: _api/web/lists
  
  Parse response and check:
  - List accessibility
  - Item counts
  - Last modified dates
  
  Output: SharePointHealth (0-100)

Branch 2: Check Flow Run History
  Action: Get items
  List: Flow Run History
  Filter: Created ge datetime'@{addDays(utcNow(), -1)}'
  
  Calculate:
  - Total runs
  - Success rate
  - Average duration
  
  Output: FlowHealth (0-100)

Branch 3: Check Data Integrity
  Action: Run data validation queries
  
  Check:
  - Orphaned records
  - Data consistency
  - Referential integrity
  
  Output: DataHealth (0-100)

Branch 4: Check System Resources
  Action: Get system metrics
  
  Check:
  - API limits
  - Storage usage
  - Connection health
  
  Output: SystemHealth (0-100)
```

#### Step 3: Calculate Overall Health Score
```
Expression:
div(
  add(
    variables('SharePointHealth'),
    variables('FlowHealth'),
    variables('DataHealth'),
    variables('SystemHealth')
  ),
  4
)
```

#### Step 4: Generate Health Report
```
Action: Compose HTML Report
Input:
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; }
    .header { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 20px; }
    .health-score { font-size: 48px; font-weight: bold; }
    .good { color: #4CAF50; }
    .warning { color: #FF9800; }
    .critical { color: #F44336; }
    table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    th, td { padding: 12px; text-align: left; border-bottom: 1px solid #ddd; }
    th { background-color: #f2f2f2; }
    .metric-card { background: #f9f9f9; border-radius: 8px; padding: 15px; margin: 10px 0; }
  </style>
</head>
<body>
  <div class="header">
    <h1>Daily System Health Report</h1>
    <p>Date: @{variables('ReportDate')}</p>
    <div class="health-score @{if(greater(variables('OverallHealth'), 80), 'good', if(greater(variables('OverallHealth'), 60), 'warning', 'critical'))}">
      Overall Health: @{variables('OverallHealth')}%
    </div>
  </div>
  
  <div class="content">
    <h2>Component Health Scores</h2>
    <table>
      <tr>
        <th>Component</th>
        <th>Score</th>
        <th>Status</th>
        <th>Issues</th>
      </tr>
      @{variables('ComponentHealthTable')}
    </table>
    
    <h2>24-Hour Metrics Summary</h2>
    <div class="metric-card">
      <h3>Flow Performance</h3>
      <ul>
        <li>Total Runs: @{variables('TotalRuns')}</li>
        <li>Success Rate: @{variables('SuccessRate')}%</li>
        <li>Average Duration: @{variables('AvgDuration')} ms</li>
        <li>Records Processed: @{variables('RecordsProcessed')}</li>
      </ul>
    </div>
    
    <div class="metric-card">
      <h3>Error Summary</h3>
      <ul>
        <li>Total Errors: @{variables('TotalErrors')}</li>
        <li>Critical Errors: @{variables('CriticalErrors')}</li>
        <li>Most Common Error: @{variables('TopError')}</li>
      </ul>
    </div>
    
    <h2>Recommendations</h2>
    <ol>
      @{variables('Recommendations')}
    </ol>
    
    <h2>Upcoming Maintenance</h2>
    @{variables('MaintenanceSchedule')}
  </div>
</body>
</html>
```

#### Step 5: Send Report
```
Action: Send an email (V2)
To: Operations Team; Management
Subject: 📊 Daily Health Report - @{variables('ReportDate')} - Health Score: @{variables('OverallHealth')}%
Body: @{outputs('Compose_HTML_Report')}
Importance: Normal

Action: Create file in SharePoint
Site: Your SharePoint site
Folder: /Reports/Daily
File Name: HealthReport_@{variables('ReportDate')}.html
File Content: @{outputs('Compose_HTML_Report')}
```

## Flow 5: Real-Time Dashboard Updates

### Trigger Configuration
```
Type: When an item is created or modified
List: Performance Metrics
```

### Step-by-Step Implementation

#### Step 1: Get Dashboard Configuration
```
Action: Get item
List: Dashboard Configuration
Id: 1 (or specific dashboard ID)

Parse configuration for:
- Update frequency
- Metrics to track
- Alert thresholds
- Display preferences
```

#### Step 2: Calculate Real-Time Metrics
```
Expression for Rolling Average (Last Hour):
div(
  sum(
    apply(
      filter(
        body('Get_Recent_Metrics'),
        x => greater(ticks(x['Created']), ticks(addHours(utcNow(), -1)))
      ),
      x => x['Value']
    )
  ),
  length(
    filter(
      body('Get_Recent_Metrics'),
      x => greater(ticks(x['Created']), ticks(addHours(utcNow(), -1)))
    )
  )
)

Expression for Trend Direction:
if(
  greater(
    variables('CurrentValue'),
    variables('PreviousValue')
  ),
  'up',
  if(
    less(
      variables('CurrentValue'),
      variables('PreviousValue')
    ),
    'down',
    'stable'
  )
)
```

#### Step 3: Update Power BI Dataset
```
Action: HTTP request to Power BI REST API
Method: POST
Uri: https://api.powerbi.com/v1.0/myorg/datasets/@{variables('DatasetId')}/rows
Headers:
  Authorization: Bearer @{body('Get_PowerBI_Token')['access_token']}
  Content-Type: application/json
Body:
{
  "rows": [
    {
      "Timestamp": "@{utcNow()}",
      "MetricName": "@{triggerBody()?['Title']}",
      "Value": @{triggerBody()?['Value']},
      "Status": "@{triggerBody()?['Status']}",
      "Trend": "@{variables('Trend')}",
      "AlertLevel": "@{variables('AlertLevel')}"
    }
  ]
}
```

#### Step 4: Send WebSocket Update (for real-time display)
```
Action: Send message to Azure SignalR
Connection String: @{parameters('SignalRConnection')}
Hub Name: DashboardHub
Target: updateMetric
Arguments:
[
  {
    "metric": "@{triggerBody()?['MetricName']}",
    "value": @{triggerBody()?['Value']},
    "timestamp": "@{utcNow()}",
    "trend": "@{variables('Trend')}",
    "alert": @{variables('HasAlert')}
  }
]
```

#### Step 5: Update SharePoint Dashboard List
```
Action: Update item
List: Dashboard Metrics
Id: @{variables('MetricId')}
Fields:
  CurrentValue: @{triggerBody()?['Value']}
  LastUpdate: @{utcNow()}
  Trend: @{variables('Trend')}
  Status: @{variables('Status')}
  AlertActive: @{variables('HasAlert')}
  History: @{concat(body('Get_item')['History'], '|', formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm:ss'), ':', triggerBody()?['Value'])}
```

## Testing Procedures

### Test Scenario 1: Performance Degradation
1. Simulate slow processing by adding delays to test flow
2. Verify performance monitoring detects degradation
3. Confirm threshold alerts trigger correctly
4. Check dashboard updates reflect performance issues

### Test Scenario 2: Error Spike
1. Force errors in test flow runs
2. Verify error alerting triggers immediately
3. Confirm cooldown period prevents alert spam
4. Check error categorization accuracy

### Test Scenario 3: System Health Check
1. Run daily health check manually
2. Verify all components are checked
3. Confirm report generation and distribution
4. Test recommendation engine logic

## Production Deployment Checklist

### Pre-Deployment
- [ ] Create all required SharePoint lists
- [ ] Configure alert recipients and channels
- [ ] Set up Power BI workspace and datasets
- [ ] Configure SignalR service (if using real-time updates)
- [ ] Test all expressions in non-production environment
- [ ] Document threshold values and escalation procedures

### Deployment
- [ ] Import monitoring flows to production
- [ ] Configure connections and authentication
- [ ] Set appropriate run frequencies
- [ ] Enable flow analytics
- [ ] Configure flow failure notifications
- [ ] Test each flow individually

### Post-Deployment
- [ ] Monitor first 24 hours closely
- [ ] Verify alerts reach intended recipients
- [ ] Confirm dashboard updates correctly
- [ ] Review and adjust thresholds based on baseline
- [ ] Document any production-specific configurations
- [ ] Train operations team on alert response

## Maintenance Guidelines

### Daily Tasks
- Review morning health report
- Check for any critical alerts
- Verify dashboard is updating

### Weekly Tasks
- Review threshold effectiveness
- Analyze false positive rate
- Check alert recipient list accuracy
- Review top errors and patterns

### Monthly Tasks
- Analyze performance trends
- Optimize alert rules
- Clean up old metric data
- Review and update documentation
- Conduct alert response drill

## Integration Points

### With Optimized Recalc Flow
- Monitor checkpoint recovery operations
- Track batch processing performance
- Alert on circuit breaker state changes
- Report on overall system throughput

### With Power BI Dashboard
- Real-time metric streaming
- Historical data analysis
- Predictive analytics integration
- Custom visualization support

### With Migration Process
- Track parallel run comparisons
- Monitor migration progress
- Alert on data discrepancies
- Report validation results

## Advanced Features

### Predictive Alerting
```
Expression for Trend Prediction:
add(
  variables('CurrentValue'),
  mul(
    div(
      sub(
        variables('CurrentValue'),
        variables('PreviousValue')
      ),
      variables('TimeInterval')
    ),
    variables('PredictionWindow')
  )
)
```

### Adaptive Thresholds
```
Expression for Dynamic Threshold:
add(
  variables('BaselineValue'),
  mul(
    variables('StandardDeviation'),
    2
  )
)
```

### Correlation Analysis
```
Expression for Correlation Detection:
if(
  and(
    greater(variables('Metric1Change'), 20),
    greater(variables('Metric2Change'), 20),
    less(
      abs(
        sub(
          variables('Metric1ChangeTime'),
          variables('Metric2ChangeTime')
        )
      ),
      300000
    )
  ),
  true,
  false
)
```

## Troubleshooting Guide

### Alert Not Firing
1. Check alert configuration is active
2. Verify threshold values and operators
3. Check cooldown period hasn't been triggered
4. Review flow run history for errors
5. Verify recipient configuration

### Dashboard Not Updating
1. Check trigger is firing correctly
2. Verify Power BI authentication
3. Check dataset refresh settings
4. Review SignalR connection
5. Verify SharePoint list permissions

### Performance Metrics Inaccurate
1. Check calculation expressions
2. Verify time window settings
3. Review data filtering criteria
4. Check for timezone issues
5. Validate source data quality

This comprehensive monitoring and alerting system ensures your On-Hand Consumable flows operate at peak efficiency with immediate visibility into any issues that arise.