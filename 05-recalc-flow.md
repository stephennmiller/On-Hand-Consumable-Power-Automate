# Flow 5: On-Hand Recalculation (Maintenance Job)

## Purpose

Nightly job that rebuilds On-Hand inventory from transaction history to prevent data drift and ensure accuracy. This flow provides a safety net against any discrepancies that might occur.

## Flow Configuration

**Flow Name:** `OH - Nightly Recalc`

**Trigger:** Recurrence Schedule  
**Frequency:** Daily at 2:00 AM (or preferred off-peak time)

**Flow Settings:**

- **Timeout:** PT2H (2 hours for Premium, 30 min for Standard)
- **Retry Policy:** Do not retry (scheduled will run next day)
- **Concurrency:** Not applicable (scheduled flow)

**Important:** This flow requires Premium connectors for >30 minute execution

## When to Implement This Flow

This flow is **OPTIONAL but HIGHLY RECOMMENDED** if:

- System handles high transaction volume (>100/day)
- Multiple users can modify On-Hand directly
- You need audit compliance
- Data integrity is critical
- You've experienced sync issues

Consider skipping if:

- Very low transaction volume (<10/day)
- Single user system
- Manual reconciliation preferred

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Scheduled cloud flow"**
3. Name: `OH - Nightly Recalc`
4. Set schedule:
   - Starting: Tomorrow at 2:00 AM
   - Repeat every: 1 Day
   - Time zone: Your local timezone
5. Click **"Create"**
6. **Advanced Options:**
   - Run only when: Weekdays (optional)
   - Catch up on missed runs: No

### Step 2: Add Master Try-Catch

Add **"Scope"** action named **"Try - Recalc Process"**

All subsequent steps go inside this scope.

### Step 3: Initialize Variables

Inside the Try scope, add these **"Initialize variable"** actions:

#### Variable 1: vProcessedCount

- **Name:** vProcessedCount
- **Type:** Integer
- **Value:** `0`

#### Variable 2: vErrorCount

- **Name:** vErrorCount
- **Type:** Integer
- **Value:** `0`

#### Variable 3: vStartTime

- **Name:** vStartTime
- **Type:** String
- **Value:** `utcNow()`

#### Variable 4: vBatchSize

- **Name:** vBatchSize
- **Type:** Integer
- **Value:** `1000` (adjust based on volume)

#### Variable 5: vLastTransactionId

- **Name:** vLastTransactionId
- **Type:** Integer
- **Value:** `0`

#### Variable 6: vHasMoreTransactions

- **Name:** vHasMoreTransactions
- **Type:** Boolean
- **Value:** `true`

#### Variable 7: vAggregations

- **Name:** vAggregations
- **Type:** Object
- **Value:** `{}`

#### Variable 8: vCheckpointCounter

- **Name:** vCheckpointCounter
- **Type:** Integer
- **Value:** `0`

### Step 3: Get Date Range (Optional)

Add **"Compose"** action to calculate date range:

**Action Name:** "Calculate Date Range"

**Inputs:**

```json
{
  "StartDate": "@{addDays(utcNow(), -90)}",
  "EndDate": "@{utcNow()}",
  "Description": "Processing last 90 days of transactions"
}
```

### Step 4: Implement Pagination for On-Hand Reset

#### Initialize Pagination Variables

Add **"Initialize variable"** actions:

- **vOnHandLastId:** Integer (0)
- **vOnHandBatchComplete:** Boolean (false)

#### Paginated Reset Loop

Add **"Do until"** control:

**Condition:** `@equals(variables('vOnHandBatchComplete'), true)`

Inside the Do until:

##### Get Batch of On-Hand Records

Add **"Get items - SharePoint"** action:

**Action Name:** "Get On-Hand Batch"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Filter Query: `ID gt @{variables('vOnHandLastId')} and IsActive eq true`
- Top Count: 100
- Order By: `ID asc`
- Select Query: `ID,PartNumber,Batch,Location,UOM`

##### Process Batch

Add **"Condition"**: Check if items returned

- Left: `length(body('Get_On-Hand_Batch')?['value'])`
- Operand: greater than
- Right: 0

**If Yes:**

1. Add **"Select"** action to build batch update:
   - From: `body('Get_On-Hand_Batch')?['value']`
   - Map:

   ```json
   {
     "id": @{item()?['ID']},
     "OnHandQty": 0,
     "LastMovementType": "Recalc-Start"
   }

```text

2. Add **"Send an HTTP request to SharePoint"** for batch update:
   - Method: POST
   - Uri: `_api/$batch`
   - Body: Construct batch request with all updates

3. Update vOnHandLastId:
   - Value: `last(body('Get_On-Hand_Batch')?['value'])?['ID']`

**If No:**

- Set vOnHandBatchComplete = true

#### Option B: Hard Reset (Risky)

**WARNING:** Only use if you're certain about data recovery

**Note:** Batch deletion requires the `_api/$batch` endpoint with proper formatting. For simplicity, use a loop to delete items individually or implement the batch format as documented in Microsoft's SharePoint REST API documentation.

### Step 5: Process Transactions with Pagination

Add **"Do until"** control:

**Condition:** `@equals(variables('vHasMoreTransactions'), false)`

**Timeout:** PT1H30M (1.5 hours)
**Count:** 1000 (max iterations)

Inside the Do until:

#### Get Transaction Batch

Add **"Get items - SharePoint"** action:

**Action Name:** "Get Transaction Batch"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Filter Query:

```

ID gt @{variables('vLastTransactionId')} and PostStatus eq 'Posted' and Created ge '@{addDays(utcNow(), -90)}'

```text

- Top Count: `@{variables('vBatchSize')}`
- Order By: `ID asc`
- Select Query: `ID,PartNumber,Batch,Location,UOM,Qty,TransactionType`
- **Settings:**
  - Retry Policy: Fixed
  - Count: 3
  - Interval: PT10S

#### Check for More Data

Add **"Condition"**:

- Left: `length(body('Get_Transaction_Batch')?['value'])`
- Operand: less than
- Right: `@{variables('vBatchSize')}`

**If Yes:** Set vHasMoreTransactions = false
**If No:** Update vLastTransactionId = `last(body('Get_Transaction_Batch')?['value'])?['ID']`

### Step 6: Initialize Aggregation Object

Add **"Compose"** action:

**Action Name:** "Initialize Aggregation"

**Inputs:**

```

{}

```text

#### Process Transaction Batch

Add **"Apply to each"** control:

**Action Name:** "Process Transaction Batch"

**Select output:** `body('Get_Transaction_Batch')?['value']`

**Settings:**

- Concurrency: On
- Degree of Parallelism: 4

Inside the loop:

##### Build Aggregation Key

Add **"Compose"** action:

**Action Name:** "Build Key"

**Inputs:**

```

@{item()?['PartNumber']}||@{item()?['Batch']}||@{coalesce(item()?['Location'],'')}||@{item()?['UOM']}

```text

##### Calculate Delta

Add **"Compose"** action:

**Action Name:** "Calculate Movement Delta"

**Inputs:**

```

@if(
  equals(toUpper(items('Process_Each_Transaction')?['TransactionType']), 'RECEIVE'),
  float(items('Process_Each_Transaction')?['Qty']),
  mul(float(items('Process_Each_Transaction')?['Qty']), -1)
)

```text

#### 7c. Increment Counter

Add **"Increment variable"** action:

- Name: vProcessedCount
- Value: 1

#### Checkpoint and Continue Pattern

Add **"Increment variable"** action:

- Name: vCheckpointCounter
- Value: 1

Add **"Condition"**: Check if checkpoint needed

- Left: `mod(variables('vCheckpointCounter'), 10)`
- Operand: equals
- Right: 0

**If Yes (Every 10 batches):**

1. Check elapsed time:

   ```

   greater(dateDifference(variables('vStartTime'), utcNow(), 'Minute'), 25)

```text

2. If approaching timeout (>25 minutes for 30-min limit):
   - Save aggregation state to SharePoint list
   - Log checkpoint
   - Terminate gracefully
   - Next run will resume from checkpoint

### Step 6: Write Aggregated Results

After processing all transactions:

#### Option 1: Batch Update Pattern (Recommended)

Add **"Select"** action to transform aggregations:

- From: `variables('vAggregations')`
- Map to update operations

Add **"Compose"** to create batch size chunks (100 items each)

Add **"Apply to each"** for each chunk:

Inside loop:

1. Build batch request JSON
2. **Send an HTTP request to SharePoint**:
   - Method: POST
   - Uri: `_api/$batch`
   - Headers:

     ```json
     {
       "Content-Type": "multipart/mixed; boundary=batch_boundary",
       "Accept": "application/json"
     }
     ```

   - Body: Batch operations

#### Option 2: Direct Update Pattern (Simpler but Slower)

For each aggregation key:

1. Parse key to get Part, Batch, Location, UOM
2. Get existing On-Hand record
3. Update or Create with calculated quantity

### Step 7: Error Handling Scope

Add **"Scope"** action named **"Catch - Handle Errors"**

**Configure Run After:**

- Has failed, is skipped, has timed out

Inside Catch scope:

#### Log Error

Add **"Create item - SharePoint"** action:

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - FlowName: `OH - Nightly Recalc`
  - ErrorMessage: `Recalc failed: @{result('Try_-_Recalc_Process')}`
  - Severity: `Critical`
  - Timestamp: `utcNow()`

#### Send Alert

Add **"Send an email (V2)"** action:

- To: `@{parameters('AdminEmail')}`
- Subject: `CRITICAL: Nightly Recalc Failed`
- Body: Include error details and impact assessment

### Step 8: Finally - Generate Report

Add **"Scope"** action named **"Finally - Summary Report"**

**Configure Run After:** All conditions

Inside Finally scope:

#### Generate Summary

Add **"Create item - SharePoint"** action:

**Action Name:** "Log Recalc Summary"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Recalc Log
- Fields:
  - RunDate: `@{variables('vStartTime')}`
  - ProcessedCount: `@{variables('vProcessedCount')}`
  - ErrorCount: `@{variables('vErrorCount')}`
  - Duration: `@{dateDifference(variables('vStartTime'), utcNow(), 'Second')}` seconds
  - Status: `@{if(greater(variables('vErrorCount'), 0), 'Completed with Errors', 'Success')}`
  - BatchesProcessed: `@{variables('vCheckpointCounter')}`

### Step 10: Send Notification (Optional)

Add **"Send an email (V2)"** action:

**Configure:**

- To: Admin email
- Subject: `Inventory Recalc Complete - @{utcNow()}`
- Body:

```text
Recalculation Summary:
- Processed: @{variables('vProcessedCount')} transactions
- Errors: @{variables('vErrorCount')}
- Duration: [calculated duration] seconds
- Time: @{variables('vStartTime')} to @{utcNow()}
```

## Performance Optimization Strategies

### For Large Datasets (>50,000 transactions)

#### Strategy 1: Date-Based Chunking

Process in weekly chunks:

```text
vCurrentDate = addDays(utcNow(), -90)
vEndDate = utcNow()

While vCurrentDate < vEndDate:
  vChunkEnd = addDays(vCurrentDate, 7)
  Process transactions between vCurrentDate and vChunkEnd
  vCurrentDate = vChunkEnd
```

#### Strategy 2: Parallel Processing

Split by Part Number ranges:

- Flow Instance 1: PartNumber A-H
- Flow Instance 2: PartNumber I-P  
- Flow Instance 3: PartNumber Q-Z
- Flow Instance 4: Numeric part numbers

#### Strategy 3: Incremental Recalc

Only process changes since last successful run:

```text
LastSuccessfulRun = Get from Recalc Log
Filter: Modified ge LastSuccessfulRun
```

### Checkpoint Recovery Pattern

For resumable processing after timeout:

1. **Save State Before Timeout:**

```json
{
  "LastProcessedId": 12345,
  "AggregationState": {...},
  "ProcessedCount": 5000,
  "Timestamp": "2024-01-15T02:25:00Z"
}
```text

2. **On Next Run:**

- Check for incomplete run
- Load saved state
- Resume from LastProcessedId
- Merge with new aggregations

3. **Cleanup After Success:**

- Clear checkpoint data
- Archive old checkpoints

### Error Handling

Add **"Configure run after"** on critical steps:

- Set to run even if previous fails
- Log errors to separate list
- Continue processing other records

### Performance Optimization

#### Batch Processing

Use JSON batching for updates:

```json
{
  "requests": [
    {
      "method": "PATCH",
      "url": "lists/getbytitle('On-Hand Material')/items(1)",
      "body": { "OnHandQty": 100 }
    }
  ]
}
```

#### Parallel Processing

- Set concurrency on Apply to each: 20
- Monitor for throttling

## Testing Strategy

### Pre-Production Testing

#### Test Case 1: Small Dataset (Smoke Test)

- [ ] Create 10 transactions (mix of Issue/Receive)
- [ ] Note current On-Hand values  
- [ ] Run recalc manually
- [ ] Verify On-Hand matches expected
- [ ] Check execution time <1 minute

#### Test Case 2: Pagination Test

- [ ] Create 2000+ transactions
- [ ] Run recalc
- [ ] Verify pagination works
- [ ] Check no items missed
- [ ] Verify memory usage stable

### Test Case 2: Data Correction

- [ ] Manually edit On-Hand to wrong value
- [ ] Run recalc
- [ ] Verify correction to accurate amount

### Test Case 3: Missing On-Hand Records

- [ ] Delete an On-Hand record
- [ ] Run recalc
- [ ] Verify record recreated with correct qty

#### Test Case 3: Timeout Simulation

- [ ] Create 10,000+ transactions
- [ ] Set timeout to 5 minutes (test)
- [ ] Run recalc
- [ ] Verify checkpoint saves
- [ ] Run again to test resume
- [ ] Verify final accuracy

#### Test Case 4: Concurrent Modification Test  

- [ ] Start recalc
- [ ] Create new transactions during run
- [ ] Verify new transactions handled correctly
- [ ] Check for lock conflicts

## Monitoring & Maintenance

### Create Monitoring Dashboard

Track these metrics:

- Daily transaction count
- Recalc duration trend
- Error frequency
- Drift corrections made

### Alert Conditions

Set up alerts for:

- Recalc duration > 30 minutes
- Error count > 10
- Failed runs

### Regular Reviews

Monthly tasks:

- Review recalc logs
- Analyze drift patterns
- Optimize date ranges
- Clean old transactions

## Troubleshooting

### Common Issues

1. **Timeout on large datasets**
   - Solution 1: Implement checkpoint/resume pattern
   - Solution 2: Reduce batch size to 500
   - Solution 3: Process in date chunks
   - Solution 4: Use Premium connector for 2-hour timeout
   - Solution 5: Split into multiple scheduled flows

2. **Memory/throttling errors**
   - Reduce concurrency
   - Add delays between operations
   - Use batch operations

3. **Incorrect calculations**
   - Verify RECEIVE vs ISSUE logic
   - Check float conversions
   - Validate aggregation keys

4. **Orphaned records**
   - Implement cleanup for 0-qty records
   - Archive old transactions
   - Set IsActive appropriately

## Decision Points

### Frequency Options

- **Every 4 hours**: High-volume (>10k transactions/day)
- **Nightly**: Standard (1k-10k transactions/day)
- **Weekly**: Low-volume (<1k transactions/day)
- **On-demand**: Testing or emergency correction

### Architecture Decision Matrix

| Daily Volume | Architecture | Frequency | Timeout Needed |
|-------------|--------------|-----------|----------------|
| <1,000 | Simple loop | Weekly | 30 min |
| 1,000-10,000 | Pagination | Nightly | 30 min |
| 10,000-50,000 | Checkpoint/Resume | Every 4 hrs | 2 hours |
| >50,000 | Queue-based | Continuous | N/A |

### Scope Options

- **Full rebuild**: Most accurate, slowest
- **Incremental**: Faster, may miss old issues
- **Changed items only**: Fastest, targeted

### Retention Options

- Keep all transactions forever
- Archive after 90 days
- Summarize monthly, delete daily

## Next Steps

1. **Implement in Test Environment**
   - Run manually first
   - Verify calculations
   - Check performance

2. **Schedule for Production**
   - Start with weekly
   - Monitor results
   - Increase frequency if needed

3. **Create Reconciliation Report**
   - Compare before/after
   - Track drift patterns
   - Document corrections

4. **Optimize Based on Results**
   - Adjust date ranges
   - Tune performance
   - Add business rules
