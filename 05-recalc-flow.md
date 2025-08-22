# Flow 5: On-Hand Recalculation (Maintenance Job)

## Purpose
Nightly job that rebuilds On-Hand inventory from transaction history to prevent data drift and ensure accuracy. This flow provides a safety net against any discrepancies that might occur.

## Flow Configuration

**Flow Name:** `OH - Nightly Recalc`

**Trigger:** Recurrence Schedule  
**Frequency:** Daily at 2:00 AM (or preferred off-peak time)

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
5. Click **"Create"**

### Step 2: Initialize Variables
Add these **"Initialize variable"** actions:

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

### Step 3: Get Date Range (Optional)
Add **"Compose"** action to calculate date range:

**Action Name:** "Calculate Date Range"

**Inputs:**
```
{
  "StartDate": "@{addDays(utcNow(), -90)}",
  "EndDate": "@{utcNow()}",
  "Description": "Processing last 90 days of transactions"
}
```

### Step 4: Clear or Mark Existing On-Hand
Choose one approach:

#### Option A: Soft Reset (Recommended)
Add **"Get items - SharePoint"** action:

**Action Name:** "Get All On-Hand Records"

**Configure:**
- Site Address: Your site
- List Name: On-Hand Material
- Filter Query: `IsActive eq true`
- Top Count: 5000

Then add **"Apply to each"** control:

**Action Name:** "Mark All as Pending Recalc"

**Select output:** `body('Get_All_On-Hand_Records')?['value']`

Inside the loop, add **"Update item - SharePoint"**:
- Id: `items('Mark_All_as_Pending_Recalc')?['ID']`
- OnHandQty: `0`
- LastMovementType: `Recalc-Pending`

#### Option B: Hard Reset (Risky)
**WARNING:** Only use if you're certain about data recovery

Add **"Send an HTTP request to SharePoint"** to batch delete:
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items`

### Step 5: Get All Transactions
Add **"Get items - SharePoint"** action:

**Action Name:** "Get Tech Transactions for Recalc"

**Configure:**
- Site Address: Your site
- List Name: Tech Transactions
- Filter Query: 
```
PostStatus eq 'Posted' and Created ge '@{addDays(utcNow(), -90)}'
```
- Top Count: 5000
- Order By: `Created asc`

**Note:** If you have >5000 transactions, implement pagination (see Advanced section).

### Step 6: Initialize Aggregation Object
Add **"Compose"** action:

**Action Name:** "Initialize Aggregation"

**Inputs:**
```
{}
```

### Step 7: Process Each Transaction
Add **"Apply to each"** control:

**Action Name:** "Process Each Transaction"

**Select output:** `body('Get_Tech_Transactions_for_Recalc')?['value']`

Inside the loop, add these actions:

#### 7a. Build Aggregation Key
Add **"Compose"** action:

**Action Name:** "Build Key"

**Inputs:**
```
@{items('Process_Each_Transaction')?['PartNumber']}||@{items('Process_Each_Transaction')?['Batch']}||@{coalesce(items('Process_Each_Transaction')?['Location'],'')}||@{items('Process_Each_Transaction')?['UOM']}
```

#### 7b. Calculate Delta
Add **"Compose"** action:

**Action Name:** "Calculate Movement Delta"

**Inputs:**
```
@if(
  equals(toUpper(items('Process_Each_Transaction')?['TransactionType']), 'RECEIVE'),
  float(items('Process_Each_Transaction')?['Qty']),
  mul(float(items('Process_Each_Transaction')?['Qty']), -1)
)
```

#### 7c. Increment Counter
Add **"Increment variable"** action:
- Name: vProcessedCount
- Value: 1

### Step 8: Write Aggregated Results
After the Apply to each loop, add another **"Apply to each"** for unique Part+Batch combinations:

**Note:** This is complex in Power Automate. Consider these alternatives:

#### Alternative 1: Use Power Automate Desktop
Export to Excel, aggregate, then import back.

#### Alternative 2: Group in SharePoint
Create a separate "Recalc Temp" list to store intermediate results.

#### Alternative 3: Simplified Approach
For each transaction, immediately update On-Hand:

Add **"Get items - SharePoint"** inside the loop:
- Filter Query: Use the key from Step 7a
- Top Count: 1

Then **"Condition"**: Check if exists
- If Yes: Update with new calculated value
- If No: Create new record

### Step 9: Generate Summary Report
Add **"Create item - SharePoint"** action:

**Action Name:** "Log Recalc Summary"

Create a "Recalc Log" list with these fields, then:

**Configure:**
- Site Address: Your site
- List Name: Recalc Log
- Fields:
  - RunDate: `@{variables('vStartTime')}`
  - ProcessedCount: `@{variables('vProcessedCount')}`
  - ErrorCount: `@{variables('vErrorCount')}`
  - Duration: `@{div(sub(ticks(utcNow()), ticks(variables('vStartTime'))), 10000000)}` (seconds)
  - Status: `Completed`

### Step 10: Send Notification (Optional)
Add **"Send an email (V2)"** action:

**Configure:**
- To: Admin email
- Subject: `Inventory Recalc Complete - @{utcNow()}`
- Body:
```
Recalculation Summary:
- Processed: @{variables('vProcessedCount')} transactions
- Errors: @{variables('vErrorCount')}
- Duration: [calculated duration] seconds
- Time: @{variables('vStartTime')} to @{utcNow()}
```

## Advanced Configuration

### Handling Large Datasets (>5000 items)

#### Pagination Implementation:
1. Add variable `vSkipCount` (Integer, default 0)
2. Add Do Until loop:
   - Condition: `length(body('Get_items')?['value']) lt 5000`
3. Inside loop:
   - Get items with `$skip=@{variables('vSkipCount')}`
   - Process items
   - Increment vSkipCount by 5000

### Incremental Recalc
Instead of full rebuild, only process recent changes:

```
Created ge '@{addDays(utcNow(), -7)}' or Modified ge '@{addDays(utcNow(), -7)}'
```

### Error Handling
Add **"Configure run after"** on critical steps:
- Set to run even if previous fails
- Log errors to separate list
- Continue processing other records

### Performance Optimization

#### Batch Processing:
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

#### Parallel Processing:
- Set concurrency on Apply to each: 20
- Monitor for throttling

## Testing Strategy

### Test Case 1: Small Dataset
- [ ] Create 10 transactions (mix of Issue/Receive)
- [ ] Note current On-Hand values
- [ ] Run recalc manually
- [ ] Verify On-Hand matches expected

### Test Case 2: Data Correction
- [ ] Manually edit On-Hand to wrong value
- [ ] Run recalc
- [ ] Verify correction to accurate amount

### Test Case 3: Missing On-Hand Records
- [ ] Delete an On-Hand record
- [ ] Run recalc
- [ ] Verify record recreated with correct qty

### Test Case 4: Performance Test
- [ ] Create 100+ transactions
- [ ] Time the recalc execution
- [ ] Check for timeout issues

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

### Common Issues:

1. **Timeout on large datasets**
   - Reduce date range (30 days instead of 90)
   - Implement pagination
   - Run multiple times per day

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

### Frequency Options:
- **Nightly**: Standard for most systems
- **Hourly**: High-volume or critical accuracy
- **Weekly**: Low-volume, stable systems
- **On-demand**: Manual trigger only

### Scope Options:
- **Full rebuild**: Most accurate, slowest
- **Incremental**: Faster, may miss old issues
- **Changed items only**: Fastest, targeted

### Retention Options:
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