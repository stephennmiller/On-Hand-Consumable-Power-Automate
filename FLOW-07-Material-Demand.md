# Flow 7: Material Demand Tracking (Enhancement)

## Purpose

Enhances the existing ISSUE processing flow (FLOW-03) to automatically update material demand records when parts are issued against Production Orders. Tracks what has been consumed vs. what was planned, enabling shortage reporting and better procurement planning.

## Prerequisites

Before implementing this flow enhancement:

1. **Create Material Demand List** (see START-HERE.md for schema)
2. **Create UOM Conversions List** (see START-HERE.md for schema)  
3. **FLOW-03 must be working** (Issue/Returned processing)
4. **PO List must have demand records** populated

## Flow Modification

**Flow Name:** `TT - Issue/Returned → OnHand Decrement` (existing FLOW-03)

**Location of Changes:** After Step 11 (successful inventory update)

## Step-by-Step Enhancement Instructions

### Step 1: Add New Scope for Demand Tracking

After the successful inventory update (Step 11 in FLOW-03), add:

1. Add **"Scope"** action
2. Rename to: **"Update Material Demand (Optional)"**
3. Configure settings:
   - Run After: Success only
   - Timeout: PT30S

### Step 2: Check Transaction Type

Inside the "Update Material Demand" scope:

1. Add **"Condition"** action
2. Rename to: **"Check if ISSUE Transaction"**
3. Expression:

```powerautomate
@equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'ISSUE')
```

Only proceed with demand update in the **Yes** branch.

### Step 3: Initialize Demand Variables

Inside the **Yes** branch of "Check if ISSUE Transaction":

Add 12 **"Initialize variable"** actions:

#### Variable 1: vDemandPartId

- **Name:** vDemandPartId
- **Type:** Integer
- **Value:** `int(coalesce(triggerBody()?['Part']?['Id'], 0))`

#### Variable 2: vDemandPOId

- **Name:** vDemandPOId
- **Type:** Integer
- **Value:** `int(coalesce(triggerBody()?['PO']?['Id'], 0))`

#### Variable 3: vDemandIssueQty

- **Name:** vDemandIssueQty
- **Type:** Float
- **Value:** `float(coalesce(triggerBody()?['Qty'], 0))`

#### Variable 4: vDemandIssueUOM

- **Name:** vDemandIssueUOM
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['UOM'], ''))`

#### Variable 5: vDemandFound

- **Name:** vDemandFound
- **Type:** Boolean
- **Value:** `false`

#### Variable 6: vDemandConvertedQty

- **Name:** vDemandConvertedQty
- **Type:** Float
- **Value:** `0`

#### Variable 7: vDemandRecordId

- **Name:** vDemandRecordId
- **Type:** Integer
- **Value:** `0`

#### Variable 8: vDemandUOM

- **Name:** vDemandUOM
- **Type:** String
- **Value:** `''`

#### Variable 9: vCurrentIssuedQty

- **Name:** vCurrentIssuedQty
- **Type:** Float
- **Value:** `0`

#### Variable 10: vNewIssuedQty

- **Name:** vNewIssuedQty
- **Type:** Float
- **Value:** `0`

#### Variable 11: vDemandETag

- **Name:** vDemandETag
- **Type:** String
- **Value:** `''`

#### Variable 12: vConversionNote

- **Name:** vConversionNote
- **Type:** String
- **Value:** `''`

### Step 4: Get Matching Demand Record

1. Add **"Get items"** action
2. Rename to: **"Get Demand for Part and PO"**
3. Configure:
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Material Demand
   - Filter Query: `PONumber/Id eq @{variables('vDemandPOId')} and Part/Id eq @{variables('vDemandPartId')} and DemandActive eq true`
   - Select Query: `Id,UOM,IssuedQty,RequiredQty,Notes`
   - Order By: `Id asc`
   - Top Count: `1`
4. Settings:
   - Retry Policy: Fixed Interval
   - Count: 3
   - Interval: PT2S

### Step 5: Process Demand if Found

1. Add **"Condition"** action
2. Rename to: **"Check Demand Record Found"**
3. Expression:

```powerautomate
@greater(length(body('Get_Demand_for_Part_and_PO')?['value']), 0)
```

#### Yes Branch: Update Demand

##### Step 5a: Get Demand Details

Add 3 **"Set variable"** actions:

1. **vDemandFound** = `true`
2. **vDemandRecordId** (new Integer variable) = `first(body('Get_Demand_for_Part_and_PO')?['value'])?['Id']`
3. **vDemandUOM** (new String variable) = `coalesce(first(body('Get_Demand_for_Part_and_PO')?['value'])?['UOM'], '')`

##### Step 5b: Handle UOM Conversion

1. Add **"Condition"** action: **"Check UOM Conversion Needed"**
2. Expression: `@not(equals(variables('vDemandUOM'), variables('vDemandIssueUOM')))`

**Yes Branch (Conversion Needed):**

1. Add **"Get items"** action: **"Get UOM Conversion Factor"**
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: UOM Conversions
   - Filter Query: `Part/Id eq @{variables('vDemandPartId')} and FromUOM eq '@{toUpper(variables('vDemandIssueUOM'))}' and ToUOM eq '@{toUpper(variables('vDemandUOM'))}' and IsActive eq true`
   - Select Query: `ConversionFactor`
   - Order By: `Id asc`
   - Top Count: `1`

   **Note:** UOM values are normalized to uppercase for consistent matching. Ensure UOM Conversions list stores values in uppercase.

2. Add **"Condition"**: **"Check Conversion Found"**
   - Expression: `@greater(length(body('Get_UOM_Conversion_Factor')?['value']), 0)`

   **Yes**:
   - Set vDemandConvertedQty = `round(mul(variables('vDemandIssueQty'), float(first(body('Get_UOM_Conversion_Factor')?['value'])?['ConversionFactor'])), 4)`
   - Set vConversionNote = `concat('Converted: ', string(variables('vDemandIssueQty')), ' ', variables('vDemandIssueUOM'), ' to ', string(variables('vDemandConvertedQty')), ' ', variables('vDemandUOM'), ' (factor: ', string(first(body('Get_UOM_Conversion_Factor')?['value'])?['ConversionFactor']), ')')`

   **No**:
   - Set vDemandConvertedQty = `variables('vDemandIssueQty')` (1:1 fallback)
   - Set vConversionNote = `concat('WARNING: No conversion found for ', variables('vDemandIssueUOM'), ' to ', variables('vDemandUOM'), ' - using 1:1 ratio')`

**No Branch (Same UOM):**
- Set vDemandConvertedQty = `variables('vDemandIssueQty')`
- Set vConversionNote = `''` (no conversion needed)

##### Step 5c: Update Issued Quantity (with Concurrency Control)

**Note:** Since FLOW-03 already has trigger concurrency set to 1, demand updates are serialized per flow instance. For additional safety with parallel manual updates:

1. Get current IssuedQty and ETag:
   - Add **"Get item"** action: **"Get Demand with ETag"**
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Material Demand
   - Id: `@{variables('vDemandRecordId')}`

2. Extract values:
   - Add **"Set variable"**: vCurrentIssuedQty (Float)
   - Value: `float(coalesce(body('Get_Demand_with_ETag')?['IssuedQty'], 0))`
   - Add **"Set variable"**: vDemandETag (String)
   - Value: `body('Get_Demand_with_ETag')?['@odata.etag']`

3. Calculate new total:
   - Add **"Set variable"**: vNewIssuedQty (Float)
   - Value: `round(add(variables('vCurrentIssuedQty'), variables('vDemandConvertedQty')), 2)`

4. Add **"Update item"** action: **"Update Demand IssuedQty"**
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Material Demand
   - Id: `@{variables('vDemandRecordId')}`
   - IssuedQty: `@{round(variables('vNewIssuedQty'), 2)}`
   - Notes: See below for expression

Notes field expression:
```powerautomate
concat(
  coalesce(first(body('Get_Demand_for_Part_and_PO')?['value'])?['Notes'], ''),
  '\n[', formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm'), '] ',
  'Issue TT#', string(triggerBody()?['ID']), ': ',
  string(variables('vDemandIssueQty')), ' ', variables('vDemandIssueUOM'),
  if(not(equals(variables('vDemandUOM'), variables('vDemandIssueUOM'))),
    concat(' (converted to ', string(round(variables('vDemandConvertedQty'), 2)), ' ', variables('vDemandUOM'), ')'),
    ''
  ),
  ' | Total issued: ', string(round(variables('vNewIssuedQty'), 2)), ' ', variables('vDemandUOM'),
  if(not(equals(variables('vConversionNote'), '')),
    concat(' | ', variables('vConversionNote')),
    ''
  )
)
```

Settings:
- Retry Policy: Exponential Backoff
- Count: 3
- Initial Interval: PT10S

#### No Branch: Log Warning

1. Add **"Compose"** action: **"Log No Demand Found"**
2. Inputs:
```json
{
  "Warning": "No active demand record found",
  "PartId": @{variables('vDemandPartId')},
  "POId": @{variables('vDemandPOId')},
  "TransactionId": @{triggerBody()?['ID']},
  "Timestamp": "@{utcNow()}"
}
```

### Step 6: Error Handling for Demand Update

Outside the "Update Material Demand" scope, add:

1. Add **"Scope"** action: **"Handle Demand Update Errors"**
2. Configure to run after "Update Material Demand" scope:
   - Has failed: Yes
   - Is skipped: Yes
   - Has timed out: Yes
   - Has succeeded: No

Inside error handler:

1. Add **"Compose"** action: **"Build Error Details"**

   **Important:** Use the Expression tab in Power Automate, not plain text JSON:

   ```powerautomate
   json(concat('{
     "FlowName": "TT - Issue/Returned → OnHand Decrement",
     "ErrorLocation": "Material Demand Update",
     "TransactionId": ', string(triggerBody()?['ID']), ',
     "PartId": ', string(variables('vDemandPartId')), ',
     "POId": ', string(variables('vDemandPOId')), ',
     "ErrorMessage": "', replace(coalesce(result('Update_Material_Demand')?['error']?['message'], 'Unknown error'), '"', '\"'), '",
     "Timestamp": "', utcNow(), '"
   }'))
   ```

2. Add **"Create item"** action: **"Log Demand Error"**
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Flow Error Log
   - Title: `Demand Update Error - TT#@{triggerBody()?['ID']}`
   - ErrorMessage: `@{outputs('Build_Error_Details')}`
   - ItemID: `@{string(triggerBody()?['ID'])}`
   - Timestamp: `@{utcNow()}`
   - Configure to not fail (Settings → Configure run after → Continue on error)

## Testing the Enhancement

### Test Case 1: Standard Issue with Demand

1. Create demand record: Part ABC123, PO-001, RequiredQty: 100 EA
2. Issue transaction: Part ABC123, PO-001, Qty: 25 EA
3. Verify: Demand IssuedQty updates to 25

### Test Case 2: Issue with UOM Conversion

1. Create demand record: Part WIRE-001, PO-002, RequiredQty: 100 FT, UOM: FT
2. Create UOM conversion: WIRE-001, IN to FT = 0.083333 (1/12)
3. Issue transaction: Part WIRE-001, PO-002, Qty: 120 IN
4. Verify: Demand IssuedQty updates to 10 FT (120 * 0.083333)

### Test Case 3: Over-Issuance

1. Create demand record: Part XYZ789, PO-003, RequiredQty: 50 EA
2. Issue transaction: Part XYZ789, PO-003, Qty: 60 EA
3. Verify: Demand IssuedQty updates to 60 (over-issuance allowed)
4. Verify: Notes field contains the update history

### Test Case 4: No Demand Record

1. Issue transaction: Part with no matching demand record
2. Verify: Transaction completes successfully
3. Verify: Warning logged but flow doesn't fail

### Test Case 5: Missing UOM Conversion

1. Create demand record with different UOM than issue
2. Don't create conversion record
3. Issue transaction
4. Verify: Uses 1:1 conversion as fallback
5. Verify: Warning noted in demand Notes field with text "WARNING: No conversion found"
6. Verify: vConversionNote contains fallback message

## Performance Considerations

- Demand update is optional (won't fail main transaction)
- Runs in separate scope with 30-second timeout
- Fixed retry policy for reads, exponential for writes
- Composite key index recommended on Material Demand list

## Troubleshooting

### Issue: Demand not updating

1. Check Flow Error Log for entries
2. Verify PONumber and Part lookups match
3. Confirm DemandActive = Yes
4. Check UOM conversion exists (if different UOMs)

### Issue: Wrong quantity calculated

1. Verify UOM conversion factor is correct
2. Check conversion is active (IsActive = Yes)
3. Ensure FromUOM and ToUOM are in correct order

### Issue: Performance degradation

1. Add composite index to Material Demand list
2. Index Part+PONumber combination
3. Consider caching common UOM conversions

## Next Steps

After implementing this enhancement:

1. Create a Weekly Shortage Report flow that:
   - Aggregates demand across all active POs
   - Compares to current on-hand inventory
   - Emails shortage alerts to procurement team
2. Create Power BI dashboard for demand visualization
3. Add predictive analytics for future demand

## Success Metrics

- 100% of ISSUE transactions update demand (where records exist)
- Performance targets:
  - P50 (median): <3 seconds additional processing time
  - P95: <5 seconds additional processing time  
  - P99: <10 seconds additional processing time
- Zero failures in main transaction flow due to demand updates
- Accurate tracking enables 20% reduction in stockouts
