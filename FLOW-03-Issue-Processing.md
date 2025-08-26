# Flow 3: Post ISSUES and RETURNED to On-Hand

## Purpose

Processes validated ISSUE and RETURNED transactions to remove inventory from the On-Hand Material list. Includes stock availability checking and prevents negative inventory.

## Flow Configuration

**Flow Name:** `TT - Issue/Returned → OnHand Decrement`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:** (Critical to prevent loops!)

```powerautomate
@and(
  equals(trim(coalesce(triggerBody()?['PostStatus'], '')), 'Validated'),
  or(
    equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'ISSUE'),
    equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'RETURNED')
  )
)
```

**Note:** Configure concurrency control at the trigger level and retry policies at individual action levels as described in the steps below.

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Issue/Returned → OnHand Decrement`
4. Choose trigger: **"When an item is created or modified - SharePoint"**
5. Configure:
   - Site Address: `@{environment('SharePointSiteUrl')}`
   - List Name: Tech Transactions
6. **Advanced Options:**
   - Limit Columns by View: Yes (performance optimization)

### Step 2: Configure Trigger Settings

#### Part A: Set Trigger Condition

1. Click the three dots on the trigger → **"Settings"**
2. Expand **"Trigger Conditions"**
3. Click **"Add"**
4. Paste the trigger condition:

```powerautomate
@and(
  equals(trim(coalesce(triggerBody()?['PostStatus'], '')), 'Validated'),
  or(
    equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'ISSUE'),
    equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'RETURNED')
  )
)
```

#### Part B: Configure Concurrency Control (CRITICAL)

5. In the same Settings panel, under **"Concurrency Control"**:
   - Toggle to **On**
   - Set **"Degree of Parallelism"** to **1** 
   - **⚠️ CRITICAL:** This MUST be set to 1 to prevent race conditions and ensure inventory accuracy
6. Click **"Done"** to save all trigger settings

### Step 3: Add Atomic Transaction Scope

Add **"Scope"** action named **"Atomic Transaction - Issue Processing"**

**Note:** The internal name will be "Atomic_Transaction_-_Issue_Processing" which is referenced in error handling expressions.

All subsequent steps (except final error handling) go inside this scope.

### Step 4: Initialize Variables

Inside the Atomic Transaction scope, add 10 **"Initialize variable"** actions:

#### Variable 1: vPartNumber

- **Name:** vPartNumber
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['Part']?['Value'], ''))`

#### Variable 1a: vPartId

- **Name:** vPartId
- **Type:** Integer
- **Value:** `int(coalesce(triggerBody()?['Part']?['Id'], 0))`

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

#### Variable 5: vId

- **Name:** vId
- **Type:** Integer
- **Value:** `int(triggerBody()?['ID'])`

#### Variable 6: vPO

- **Name:** vPO
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['PO']?['Value'], ''))`

#### Variable 7: vFlowRunId

- **Name:** vFlowRunId
- **Type:** String
- **Value:** `workflow()?['run']?['name']`

#### Variable 8: vOriginalQty

- **Name:** vOriginalQty
- **Type:** Float
- **Value:** `0` (will store original on-hand for rollback)

#### Variable 9: vUpdateCompleted

- **Name:** vUpdateCompleted
- **Type:** Boolean
- **Value:** `false` (tracks if inventory was modified)

#### Variable 10: RetryCount

- **Name:** RetryCount
- **Type:** Integer
- **Value:** `0` (for ETag lock retry logic)

### Step 5: Add Throttle Protection

Add **"Delay"** action:

- Count: 100
- Unit: Millisecond

**Rationale**: This small delay helps prevent SharePoint API throttling when multiple transactions arrive simultaneously. The 100ms delay is negligible for users but helps distribute API calls, reducing the chance of hitting SharePoint's rate limits (600 calls/minute per flow). This is especially important since this flow has concurrency set to 1, meaning transactions queue up and could hit the API rapidly in succession.

### Step 6: Get and Lock On-Hand Row

Add **"Get items - SharePoint"** action:

**Action Name:** "Get_OnHand_for_Part_Batch_with_Lock"

**Note:** Power Automate generates internal action names from what you type. To ensure expressions work correctly, rename this action to exactly "Get_OnHand_for_Part_Batch_with_Lock" (with underscores, not spaces or special characters) in the action's settings.

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Filter Query:

```powerautomate
Part/Id eq @{variables('vPartId')} and Batch eq '@{replace(variables('vBatch'),'''','''''')}' and UOM eq '@{replace(variables('vUOM'),'''','''''')}' and IsActive eq true
```

- Top Count: 1
- **Order By:** `Modified desc`
- **Select Query:** `ID,OnHandQty,Part,Batch,UOM,Title`
- **Expand Query:** `Part`
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
@greater(length(body('Get_OnHand_for_Part_Batch_with_Lock')?['value']), 0)
```

**If No:**

- Add **"Update item - SharePoint"** action
- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `No inventory found for Part: @{variables('vPartNumber')}, Batch: @{variables('vBatch')}, UOM: @{variables('vUOM')}`
  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 8: Store Original Quantity (In YES Branch)

Add **"Set variable"** action in the YES branch:

- Name: vOriginalQty
- Value: `float(first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['OnHandQty'])`

### Step 9: Capture ETag and Original Data

Add **"Compose"** action:

**Action Name:** "Capture ETag"

**Inputs:**
```powerautomate
@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['@odata.etag']}
```

Add **"Initialize variable"** action:

**Name:** vETag
**Type:** String
**Value:** `@{outputs('Capture_ETag')}`

Add **"Initialize variable"** action:

**Name:** vOriginalTitle
**Type:** String
**Value:** `@{coalesce(first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['Title'], '')}`

Add **"Initialize variable"** action:

**Name:** vLockAcquired
**Type:** Boolean
**Value:** `false`

### Step 10: Attempt Optimistic Lock with ETag (Bounded Retries)

Add **"Do Until"** loop for lock acquisition:

**Settings:**
- Count: 3
- Timeout: PT1M

**Condition:** `@or(equals(variables('vLockAcquired'), true), greater(variables('RetryCount'), 2))`

Inside each iteration of the loop:

#### 10a. Send Lock Request

Add **"Send an HTTP request to SharePoint"** action.

**⚠️ CRITICAL:** After adding this action, immediately click the title bar and rename it to exactly **"Lock_OnHand_with_ETag_Attempt"** (this exact name is required for all the expressions in subsequent steps).

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items(@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']})`
- Headers:
  - If-Match: `@{variables('vETag')}`
  - X-HTTP-Method: MERGE
  - Accept: application/json;odata=nometadata
  - Content-Type: application/json;odata=nometadata
- Body:
```json
{
  "Title": "LOCKED-@{variables('vFlowRunId')}-@{utcNow('yyyyMMddHHmmss')}"
}
```

**Settings:**
- Retry Policy: None
- Configure run after: Continue if succeeded OR failed

#### 10b. Check Lock Success

Add **"Compose"** action:

**Action Name:** "Lock_Status_Code"

**Inputs:** `@{outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode']}`

Add **"Set variable"** - vLockAcquired:
- Value: `@{equals(outputs('Lock_Status_Code'), 204)}`

#### 10c. Check if Retry Needed

Add **"Condition"** - Check if retry needed:

**Condition:**
```powerautomate
@and(
  equals(variables('vLockAcquired'), false),
  or(
    equals(outputs('Lock_Status_Code'), 412),
    equals(outputs('Lock_Status_Code'), 429),
    greaterOrEquals(outputs('Lock_Status_Code'), 500)
  )
)
```

**If Yes (Retry needed):**
1. **Delay** action: PT2S (2 seconds)
2. **Get items - SharePoint** (Re-fetch for new ETag):
   - Same configuration as Step 6
   - Action Name: "Get_OnHand_for_Part_Batch_with_Lock_Retry"
3. **Set variable** - vETag:
   - Value: `@{first(body('Get_OnHand_for_Part_Batch_with_Lock_Retry')?['value'])?['@odata.etag']}`
4. **Increment variable** - RetryCount:
   - Value: 1

After the loop:

**Condition:** `@equals(variables('vLockAcquired'), false)`

**If Yes (Lock Failed after retries):**
- Add **"Terminate"** action with Status: Failed and Message: "Could not acquire lock after 3 attempts - concurrent modification detected"

### Step 11: Compute New Quantity

Add **"Compose"** action:

**Action Name:** "Compute New Qty"

**Inputs:**

```powerautomate
@sub(
  float(variables('vOriginalQty')),
  float(variables('vQty'))
)
```

### Step 12: Check Sufficient Stock

Add **"Condition"** action:

**Condition Name:** "Sufficient Stock?"

**Configure:**

- outputs('Compute_New_Qty') | is greater than or equal to | 0

**If No:**

#### Release Lock First (Only if Acquired)

Add **"Condition"** action:

**Condition:** `@equals(variables('vLockAcquired'), true)`

**If Yes:**

Add **"Get item - SharePoint"** action to refresh ETag:

**Action Name:** "Get_Current_ETag_For_Unlock"

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']}`

Add **"Send an HTTP request to SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items(@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']})`
- Headers:
  - X-HTTP-Method: MERGE
  - Accept: application/json;odata=nometadata
  - Content-Type: application/json;odata=nometadata
  - If-Match: `@{body('Get_Current_ETag_For_Unlock')?['@odata.etag']}`
- Body:
```json
{
  "Title": "@{variables('vOriginalTitle')}"
}
```

#### Then Update Transaction

- Add **"Update item - SharePoint"** action
- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage:
```powerautomate
Insufficient inventory. Available: @{round(variables('vOriginalQty'), 2)}, Requested: @{round(variables('vQty'), 2)}
```
- PostedAt: `utcNow()`

Add **"Terminate"** action:
- Status: Succeeded

### Step 13: Update On-Hand with Unlock (In YES Branch)

Add **"Get item - SharePoint"** action to refresh ETag:

**Action Name:** "Get_Current_ETag_For_Update"

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']}`

Add **"Send an HTTP request to SharePoint"** action:

**Action Name:** "Update_OnHand_Qty_and_Release_Lock"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items(@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']})`
- Headers:
  - X-HTTP-Method: MERGE
  - Accept: application/json;odata=nometadata
  - Content-Type: application/json;odata=nometadata
  - If-Match: `@{body('Get_Current_ETag_For_Update')?['@odata.etag']}`
- Body:
```json
{
  "Title": "@{variables('vOriginalTitle')}",
  "OnHandQty": @{outputs('Compute_New_Qty')},
  "LastMovementAt": "@{utcNow()}",
  "LastMovementType": "Issue",
  "LastMovementRefId": "@{variables('vId')}",
  "IsActive": @{if(greater(outputs('Compute_New_Qty'), 0), true, false)}
}
```

- **Settings:**
  - Retry Policy: Exponential
  - Count: 3
  - Interval: PT10S

### Step 14: Set Update Completed Flag

Add **"Set variable"** action:

- Name: vUpdateCompleted
- Value: `true`

### Step 15: Mark Transaction as Posted

Add **"Update item - SharePoint"** action:

**Action Name:** "Mark Tech Transaction Posted"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Posted`
  - PostMessage: `Successfully issued from inventory. Remaining: @{round(outputs('Compute_New_Qty'), 2)}`
  - PostedAt: `utcNow()`
- **Settings:**
  - Retry Policy: Fixed Interval
  - Count: 3
  - Interval: PT5S

### Step 16: Add Error Handling with Automatic Rollback

Add **"Scope"** action named **"Compensate - Rollback on Failure"**

**Important:** This scope handles errors and automatically rolls back inventory changes if the flow fails after updating On-Hand Material.

**Configure Run After:**
1. Click the three dots on the scope
2. Select "Settings"
3. Under "Configure run after", check:
   - ✅ Has failed
   - ✅ Is skipped
   - ✅ Has timed out
   - ❌ Is successful (unchecked)

Inside Compensate scope:

#### Step 16a: Check if Rollback Needed

Add **"Condition"** action:

**Purpose:** Only rollback if we successfully updated inventory but then failed

- Left: `@{variables('vUpdateCompleted')}`
- Operand: is equal to
- Right: `true`

**If Yes (Inventory was modified - Need to Rollback):**

##### Rollback Inventory Update

First, get the current ETag for rollback:

Add **"Get item - SharePoint"** action:
**Action Name:** "Get_ETag_For_Rollback"
- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']}`

Then, add **"Send an HTTP request to SharePoint"** action:

**Action Name:** "Rollback_OnHand_to_Original"

**Configure:**

- Site Address: `@{environment('SharePointSiteUrl')}`
- Method: POST
- Uri: `_api/web/lists/getbytitle('On-Hand Material')/items(@{first(body('Get_OnHand_for_Part_Batch_with_Lock')?['value'])?['ID']})`
- Headers:
  - If-Match: `@{body('Get_ETag_For_Rollback')?['@odata.etag']}`
  - X-HTTP-Method: MERGE
  - Accept: application/json;odata=nometadata
  - Content-Type: application/json;odata=nometadata
- Body:
```json
{
  "Title": "@{variables('vOriginalTitle')}",
  "OnHandQty": @{variables('vOriginalQty')},
  "IsActive": @{if(greater(variables('vOriginalQty'), 0), true, false)},
  "LastMovementAt": "@{utcNow()}",
  "LastMovementType": "Rollback-Issue",
  "LastMovementRefId": "ROLLBACK-@{variables('vId')}"
}
```
- **Settings:**
  - Retry Policy: Exponential
  - Count: 5 (critical to succeed)
  - Interval: PT30S

##### Log Rollback

Add **"Create item - SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - Title: `TT - Issue/Returned → OnHand Decrement`
  - ItemID: `@{variables('vId')}`
  - ErrorMessage: `Rollback executed for transaction @{variables('vId')}`
  - Timestamp: `utcNow()`

#### Step 16b: Always Execute - Error Logging (Outside the condition)

##### Log Error

Add **"Create item - SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - Title: `TT - Issue/Returned → OnHand Decrement`
  - ItemID: `@{variables('vId')}`
  - ErrorMessage: `@{string(result('Atomic_Transaction_-_Issue_Processing'))}`
  - Timestamp: `utcNow()`

##### Update Transaction with Error

Add **"Update item - SharePoint"** action:

- Site Address: `@{environment('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `Failed to process issue. Rollback @{if(variables('vUpdateCompleted'), 'completed', 'not needed')}. Run: @{workflow()?['run']?['name']}`
  - PostedAt: `utcNow()`

##### Send Critical Alert

Add **"Send an email (V2)"** action:

- To: `@{environment('AdminEmail')}`
- Subject: `CRITICAL: Issue Processing Failed - Rollback @{if(variables('vUpdateCompleted'), 'Executed', 'N/A')}`
- Body:

```powerautomate
Transaction ID: @{variables('vId')}
Part: @{variables('vPartNumber')}
Batch: @{variables('vBatch')}
Quantity: @{variables('vQty')}
PO: @{variables('vPO')}
Original On-Hand: @{variables('vOriginalQty')}
Update Completed: @{variables('vUpdateCompleted')}
Rollback Status: @{if(variables('vUpdateCompleted'), 'Executed', 'Not Required')}
Flow Run: @{variables('vFlowRunId')}
Time: @{utcNow()}

Immediate investigation required.
```

## Testing Checklist

### Test Case 1: Normal Issue

- [ ] Create validated ISSUE with sufficient stock
- [ ] Verify On-Hand quantity decreases correctly
- [ ] Check LastMovementType = "Issue"
- [ ] Verify PostStatus = "Posted"

### Test Case 2: Insufficient Stock

- [ ] Create ISSUE for more than available
- [ ] Verify PostStatus = "Error"
- [ ] Check error message shows available vs requested
- [ ] Verify On-Hand unchanged

### Test Case 3: Exact Depletion

- [ ] Issue exact quantity to reach 0
- [ ] Verify OnHandQty = 0
- [ ] Check IsActive = false
- [ ] Verify transaction posted successfully

### Test Case 4: No Inventory Found

- [ ] Issue for non-existent Part+Batch
- [ ] Verify PostStatus = "Error"
- [ ] Check error message indicates no inventory

### Test Case 5: RETURNED Transaction - Full Stock

- [ ] Create validated RETURNED transaction with valid PO
- [ ] Verify On-Hand quantity decreases (same as ISSUE)
- [ ] Check LastMovementType = "Returned"
- [ ] Verify PostStatus = "Posted"
- [ ] Confirm PostMessage indicates returned to vendor

### Test Case 6: RETURNED Transaction - Partial Stock

- [ ] Create RETURNED for partial quantity (e.g., 5 of 10 available)
- [ ] Verify remaining quantity is correct (5 left)
- [ ] Check transaction processes successfully
- [ ] Verify LastMovementRefId contains transaction ID

### Test Case 7: RETURNED Transaction - Insufficient Stock

- [ ] Create RETURNED for more than available
- [ ] Verify PostStatus = "Error"
- [ ] Check error message shows "Cannot return 20 units - only 10 available"
- [ ] Verify On-Hand unchanged

### Test Case 8: RETURNED Transaction - Invalid PO

- [ ] Create RETURNED without PO reference
- [ ] Verify validation fails (FLOW-01 should catch)
- [ ] Check PostStatus remains "New" (not validated)
- [ ] Verify PostMessage indicates missing PO

## Configuration Options

### Option 1: Allow Negative Inventory

To allow negative inventory (backorders), remove or modify Step 12 (Check Sufficient Stock):

- Remove the sufficient stock check entirely, OR
- Change condition to check for a minimum threshold (e.g., -100)

### Option 2: Soft Warning

Instead of blocking insufficient stock, allow but warn:

- In Step 7 "If No" branch, don't Terminate
- Set PostMessage to warning but continue processing

### Option 3: Reserve Safety Stock

Modify Step 7 to maintain minimum stock:

```powerautomate
@greater(outputs('Compute_New_Qty'), 10)
```

This maintains a safety stock of 10 units.

## Troubleshooting

### Common Issues

1. **Flow not triggering**
   - Verify PostStatus = "Validated" exactly (check Choice column values)
   - Check TransactionType = "Issue" (case insensitive due to toUpper)
   - Ensure trigger condition syntax: `@and(equals(trim(coalesce(triggerBody()?['PostStatus'], '')), 'Validated'), equals(toUpper(trim(coalesce(triggerBody()?['TransactionType'], ''))), 'ISSUE'))`

2. **Stock calculations wrong**
   - Verify float conversion: `float(variables('vQty'))`
   - Check subtraction: `@sub(float(variables('vOriginalQty')), float(variables('vQty')))`
   - Test with decimal quantities (e.g., 10.5)

3. **Can't find inventory**
   - Ensure exact matches using numeric Id: `int(coalesce(triggerBody()?['Part']?['Id'],0))`
   - Escape single quotes where applicable: `replace(variables('vBatch'),'''','''''')` and `replace(variables('vUOM'),'''','''''')`
   - Verify IsActive = true in filter query

4. **ETag/Lock errors**
   - Ensure using correct ETag capture from body not outputs: `@{body('Get_Current_ETag_For_Update')?['@odata.etag']}`
   - Verify Do Until loop with proper retry logic for 412/429/5xx errors
   - Check Lock status code: `equals(outputs('Lock_Status_Code'), 204)`
   - If using verbose mode, verify metadata type matches your list internal name\n   - With nometadata mode (recommended), omit __metadata entirely

5. **Performance issues**
   - Ensure columns are indexed (Part, Batch, UOM)
   - Use Top Count: 1 on Get items
   - Set trigger concurrency to 1 to prevent race conditions
   - Add throttle delays between operations

## Expression Reference

### Subtract Quantity

```powerautomate
@sub(
  float(variables('vOriginalQty')),
  float(variables('vQty'))
)
```

### Conditional IsActive

```powerautomate
@{if(greater(outputs('Compute_New_Qty'), 0), true, false)}
```

### Format Error Message with Locale-Safe Numbers

```powerautomate
Insufficient inventory. Available: @{round(variables('vOriginalQty'), 2)}, Requested: @{round(variables('vQty'), 2)}
```

## Next Steps

Once this flow is working correctly:

- Test all stock scenarios thoroughly
- Verify PO tracking is maintained
- Consider implementing Flow 4 (Autofill) for better UX
- Consider implementing Flow 5 (Recalc) for data integrity
