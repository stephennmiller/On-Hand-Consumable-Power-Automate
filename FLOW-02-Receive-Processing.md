# Flow 2: Post RECEIVES to On-Hand

## Purpose

Processes validated RECEIVE transactions to add inventory to the On-Hand Material list. Creates new inventory records or updates existing ones.

## Flow Configuration

**Flow Name:** `TT - Receive → OnHand Upsert`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:** (Critical to prevent loops!)

```powerautomate
@and(
  equals(trim(coalesce(triggerOutputs()?['body/PostStatus'], '')), 'Validated'),
  equals(toUpper(trim(coalesce(triggerOutputs()?['body/TransactionType'], ''))), 'RECEIVE')
)
```

**Note:** Configure concurrency control at the trigger level and retry policies at individual action levels as described in the steps below.

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Receive → OnHand Upsert`
4. Choose trigger: **"When an item is created or modified - SharePoint"**
5. Configure:
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Tech Transactions
6. **Advanced Options:**
   - Limit Columns by View: Use a view that includes only necessary fields

### Step 2: Set Trigger Condition

1. Click the three dots on the trigger → **"Settings"**
2. Expand **"Trigger Conditions"**
3. Click **"Add"**
4. Paste the trigger condition:

```powerautomate
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'RECEIVE')
)
```

1. Click **"Done"**

### Step 3: Add Transaction Scope

Add **"Scope"** action named **"Transaction - Receive Processing"**

All subsequent steps go inside this scope for atomic transaction handling.

### Step 4: Initialize Variables

Inside the Transaction scope, add 8 **"Initialize variable"** actions:

#### Variable 1: vPart

- **Name:** vPart
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['PartNumber'], ''))`

#### Variable 2: vBatch

- **Name:** vBatch
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['Batch'], ''))`

#### Variable 3: vQty

- **Name:** vQty
- **Type:** Float
- **Value:** `float(coalesce(triggerBody()?['Qty'], 0))`

#### Variable 4: vUOM

- **Name:** vUOM
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['UOM'], ''))`

#### Variable 5: vLoc

- **Name:** vLoc
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['Location'], ''))`

#### Variable 6: vId

- **Name:** vId
- **Type:** String
- **Value:** `triggerBody()?['ID']`

#### Variable 7: vFlowRunId

- **Name:** vFlowRunId
- **Type:** String
- **Value:** `workflow()?['run']?['name']`

#### Variable 8: vTransactionSuccess

- **Name:** vTransactionSuccess
- **Type:** Boolean
- **Value:** `false`

### Step 5: Add Throttle Delay

Add **"Delay"** action:

- Count: 100
- Unit: Millisecond

#### Throttling Protection

This prevents SharePoint throttling when processing multiple transactions.

### Step 6: Get Matching On-Hand Row with Pagination

Add **"Get items - SharePoint"** action:

**Action Name:** "Get_On-Hand_for_Part+Batch"

**Note:** Power Automate generates internal action names based on what you type. To ensure expressions work correctly, rename this action to exactly "Get_On-Hand_for_Part+Batch" (without spaces or parentheses) in the action's settings.

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Filter Query:

```powerautomate
(PartNumber eq '@{replace(variables('vPart'),'''','''''')}') and (Batch eq '@{replace(variables('vBatch'),'''','''''')}') and (UOM eq '@{replace(variables('vUOM'),'''','''''')}') and (IsActive eq true)
```

**Note:** If using Location field, add to filter:

```powerautomate
(PartNumber eq '@{replace(variables('vPart'),'''','''''')}') and (Batch eq '@{replace(variables('vBatch'),'''','''''')}') and (UOM eq '@{replace(variables('vUOM'),'''','''''')}') and (Location eq '@{replace(variables('vLoc'),'''','''''')}') and (IsActive eq true)
```

- Top Count: 1
- **Select Query:** `ID,Title,OnHandQty,PartNumber,Batch,UOM,Location`
- **Order By:** `Modified desc`
- **Settings:**
  - Retry Policy: Fixed Interval
  - Count: 3
  - Interval: PT2S

### Step 7: Check if Row Exists

Add **"Condition"** action:

**Condition Name:** "On-Hand Row Exists?"

**Configure:**

- Click "Edit in advanced mode"
- Paste:

```powerautomate
@greater(length(body('Get_On-Hand_for_Part+Batch')?['value']), 0)
```

### Step 8: Configure YES Branch (Update Existing)

In the **Yes** branch:

#### Step 8a: Capture ETag for Optimistic Locking

Add **"Compose"** action:

**Action Name:** "Capture ETag"

**Inputs:**
```powerautomate
@{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['@odata.etag']}
```

#### Step 8a2: Store Original Title

Add **"Initialize variable"** action:

**Name:** vOriginalTitle
**Type:** String
**Value:** `@{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['Title']}`

#### Step 8b: Update with New Quantity (Using ETag)

Add **"Send an HTTP request to SharePoint"** action:

**Action Name:** "Update Existing On-Hand with ETag"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items(@{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['ID']})`
- Headers:
  - IF-MATCH: `@{outputs('Capture_ETag')}`
  - X-HTTP-Method: MERGE
  - Content-Type: application/json;odata=verbose
- Body:
```json
{
  "__metadata": {
    "type": "SP.Data.On_x002d_Hand_x0020_MaterialListItem"
  },
  "Title": "@{variables('vOriginalTitle')}",
  "OnHandQty": @{add(float(coalesce(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty'], 0)), float(variables('vQty')))},
  "LastMovementAt": "@{utcNow()}",
  "LastMovementType": "Receive",
  "LastMovementRefId": "@{variables('vId')}",
  "IsActive": true
}
```

**Note:** If this action fails with HTTP 412 (Precondition Failed), it means another flow has modified the record. Configure run after to handle this case.

#### Step 8c: Handle Concurrency Conflict (Configure Run After)

Add **"Condition"** action:

**Configure Run After:** Set to run when "Update Existing On-Hand with ETag" has failed

**Condition:** `@equals(outputs('Update_Existing_On-Hand_with_ETag')?['statusCode'], 412)`

**If Yes (Conflict - Last Writer Wins):**
- Add a **"Delay"** action: 2 seconds
- Add **"Get items - SharePoint"** to fetch latest values
- Recalculate and update with new quantity
- Consider logging conflict for audit

**If No (Other Error):**
- Add **"Terminate"** action with Status: Failed and Message: `@{string(outputs('Update_Existing_On-Hand_with_ETag'))}`

### Step 9: Configure NO Branch (Create New)

In the **No** branch, add **"Create item - SharePoint"** action:

**Action Name:** "Create New On-Hand"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Fields:
  - PartNumber: `@{variables('vPart')}`
  - Batch: `@{variables('vBatch')}`
  - UOM: `@{variables('vUOM')}`
  - Location: `@{if(greater(length(variables('vLoc')), 0), variables('vLoc'), null)}`
  - OnHandQty: `@{variables('vQty')}`
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Receive`
  - LastMovementRefId: `@{variables('vId')}`
  - IsActive: `true`
- **Settings:**
  - Retry Policy: Exponential
  - Count: 3
  - Interval: PT10S

### Step 10: Set Success Flag

Add **"Set variable"** action (outside the condition):

- Name: vTransactionSuccess
- Value: `true`

### Step 11: Mark Transaction as Posted

Add **"Update item - SharePoint"** action:

**Action Name:** "Mark Tech Transaction Posted"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Posted`
  - PostMessage: `Successfully added to inventory`
  - PostedAt: `utcNow()`
- **Settings:**
  - Retry Policy: Fixed Interval
  - Count: 3
  - Interval: PT5S

### Step 12: Add Error Handling

Add **"Scope"** action named **"Catch - Handle Errors"**

**Configure Run After:**

- Has failed, is skipped, has timed out

Inside Catch scope:

#### Step 12a: Check if Partial Success

Add **"Condition"** action:

- Left: `@{variables('vTransactionSuccess')}`
- Operand: is equal to
- Right: `false`

**If Yes (Transaction Failed):**

##### Rollback Actions

Add **"Condition"** to check if update was attempted:

- If On-Hand was created/updated but posting failed, consider reversal

##### Log Error

Add **"Create item - SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - Title: `TT - Receive → OnHand Upsert`
  - ItemID: `@{variables('vId')}`
  - ErrorMessage: `@{string(result('Transaction_-_Receive_Processing'))}`
  - Timestamp: `utcNow()`

##### Update Transaction with Error

Add **"Update item - SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `Failed to process receive. Run ID: @{variables('vFlowRunId')}`
  - PostedAt: `utcNow()`

##### Send Alert

Add **"Send an email (V2)"** action:

- To: `@{environment('AdminEmail')}`
- Subject: `ERROR: Receive Processing Failed`
- Body: Include transaction details and error message

### Step 13: Add Performance Monitoring (Optional)

Add **"Compose"** action at the end:

**Action Name:** "Performance Metrics"

**Inputs:**

```json
{
  "FlowRunId": "@{variables('vFlowRunId')}",
  "TransactionId": "@{variables('vId')}",
  "ProcessingTimeSeconds": "@{div(sub(ticks(utcNow()), ticks(workflow()?['run']?['startTime'])), 10000000)}",
  "PartNumber": "@{variables('vPart')}",
  "Quantity": @{variables('vQty')},
  "Success": @{variables('vTransactionSuccess')}
}
```

## Testing Checklist

### Test Case 1: New Part Receive

- [ ] Create validated RECEIVE for new Part+Batch
- [ ] Verify new row created in On-Hand Material
- [ ] Check OnHandQty equals transaction Qty
- [ ] Verify PostStatus = "Posted"

### Test Case 2: Existing Part Receive

- [ ] Create validated RECEIVE for existing Part+Batch
- [ ] Verify On-Hand row updated (not duplicated)
- [ ] Check OnHandQty increased by transaction Qty
- [ ] Verify LastMovementAt updated

### Test Case 3: Multiple Receives

- [ ] Create 3 RECEIVE transactions for same Part+Batch
- [ ] Verify single On-Hand row with summed quantity
- [ ] Check LastMovementRefId points to latest transaction

### Test Case 4: Different Locations

- [ ] If using Location field:
  - Create RECEIVE for Part+Batch at Location A
  - Create RECEIVE for same Part+Batch at Location B
  - Verify separate On-Hand rows per location

## Troubleshooting

### Common Issues

1. **Flow not triggering**
   - Verify PostStatus = "Validated" exactly
   - Check TransactionType = "Receive" (case insensitive)
   - Ensure trigger condition syntax is correct

2. **Duplicate On-Hand rows**
   - Check filter query matches exactly
   - Verify trimming is applied consistently
   - Ensure Part, Batch, UOM match precisely

3. **Quantity not updating**
   - Check OnHandQty expression syntax
   - Verify float conversion is applied
   - Test with decimal quantities

4. **Performance issues**
   - Ensure columns are indexed (PartNumber, Batch)
   - Use Top Count: 1 on Get items
   - Consider adding delays if necessary

## Expression Reference

### Add Quantities

```powerautomate
add(
  float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
  float(variables('vQty'))
)
```

### Get First Item ID

```powerautomate
first(body('Get_On-Hand_for_Part+Batch')?['value'])?['ID']
```

### Check Empty Results

```powerautomate
@greater(length(body('Get_On-Hand_for_Part+Batch')?['value']), 0)
```

## Next Steps

Once this flow is working correctly:

- Test with multiple receive scenarios
- Verify On-Hand Material aggregates correctly
- Proceed to Flow 3: Issue transactions processing
