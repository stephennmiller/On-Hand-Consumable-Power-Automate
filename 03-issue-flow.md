# Flow 3: Post ISSUES to On-Hand

## Purpose

Processes validated ISSUE transactions to remove inventory from the On-Hand Material list. Includes stock availability checking and prevents negative inventory.

## Flow Configuration

**Flow Name:** `TT - Issue → OnHand Upsert`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:** (Critical to prevent loops!)

```powerautomate
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'ISSUE')
)
```

**Flow Settings:**

- **Concurrency Control:** On
- **Degree of Parallelism:** 1 (critical for stock accuracy)
- **Retry Policy:** Exponential backoff
- **Timeout:** PT5M (5 minutes)
- **Run Priority:** High (issues before receives)

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Issue → OnHand Upsert`
4. Choose trigger: **"When an item is created or modified - SharePoint"**
5. Configure:
   - Site Address: `@{parameters('SharePointSiteUrl')}`
   - List Name: Tech Transactions
6. **Advanced Options:**
   - Limit Columns by View: Yes (performance optimization)

### Step 2: Set Trigger Condition

1. Click the three dots on the trigger → **"Settings"**
2. Expand **"Trigger Conditions"**
3. Click **"Add"**
4. Paste the trigger condition:

```powerautomate
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'ISSUE')
)
```

1. Click **"Done"**

### Step 3: Add Atomic Transaction Scope

Add **"Scope"** action named **"Atomic Transaction - Issue Processing"**

All subsequent steps (except final error handling) go inside this scope.

### Step 4: Initialize Variables

Inside the Atomic Transaction scope, add 10 **"Initialize variable"** actions:

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

#### Variable 7: vPO

- **Name:** vPO
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['PONumber'], ''))`

#### Variable 8: vFlowRunId

- **Name:** vFlowRunId
- **Type:** String
- **Value:** `workflow()?['run']?['name']`

#### Variable 9: vOriginalQty

- **Name:** vOriginalQty
- **Type:** Float
- **Value:** `0` (will store original on-hand for rollback)

#### Variable 10: vUpdateCompleted

- **Name:** vUpdateCompleted
- **Type:** Boolean
- **Value:** `false` (tracks if inventory was modified)

### Step 5: Add Throttle Protection

Add **"Delay"** action:

- Count: 100
- Unit: Millisecond

### Step 6: Get and Lock On-Hand Row

Add **"Get items - SharePoint"** action:

**Action Name:** "Get On-Hand for Part+Batch with Lock"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Filter Query:

```powerautomate
(PartNumber eq '@{variables('vPart')}') and (Batch eq '@{variables('vBatch')}') and (UOM eq '@{variables('vUOM')}') and (IsActive eq true)
```

**Note:** If using Location field:

```powerautomate
(PartNumber eq '@{variables('vPart')}') and (Batch eq '@{variables('vBatch')}') and (UOM eq '@{variables('vUOM')}') and (Location eq '@{variables('vLoc')}') and (IsActive eq true)
```

- Top Count: 1
- **Select Query:** `ID,OnHandQty,PartNumber,Batch,UOM,Location,Title`
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
@greater(length(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value']), 0)
```

**If No:**

- Add **"Update item - SharePoint"** action
- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `No inventory found for Part: @{variables('vPart')}, Batch: @{variables('vBatch')}`
  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 8: Store Original Quantity (In YES Branch)

Add **"Set variable"** action in the YES branch:

- Name: vOriginalQty
- Value: `float(first(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value'])?['OnHandQty'])`

### Step 9: Attempt Lock (Optimistic Locking)

Add **"Update item - SharePoint"** action:

**Action Name:** "Lock On-Hand Record"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value'])?['ID']`
- Fields:
  - Title: `LOCKED-@{variables('vFlowRunId')}-@{utcNow('yyyyMMddHHmmss')}`
- **Settings:**
  - Retry Policy: None (fail fast if already locked)
  - Configure run after: Continue only if previous action succeeded

### Step 10: Compute New Quantity

Add **"Compose"** action:

**Action Name:** "Compute New Qty"

**Inputs:**

```powerautomate
@sub(
  float(variables('vOriginalQty')),
  float(variables('vQty'))
)
```

### Step 11: Check Sufficient Stock

Add **"Condition"** action:

**Condition Name:** "Sufficient Stock?"

**Configure:**

- outputs('Compute_New_Qty') | is greater than or equal to | 0

**If No:**

#### Release Lock First

Add **"Update item - SharePoint"** action:

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value'])?['ID']`
- Fields:
  - Title: ` ` (clear lock)

#### Then Update Transaction

- Add **"Update item - SharePoint"** action
- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage:

```powerautomate
    Insufficient inventory. Available: @{variables('vOriginalQty')}, Requested: @{variables('vQty')}
    ```

  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 12: Update On-Hand with Unlock (In YES Branch)

Add **"Update item - SharePoint"** action:

**Action Name:** "Update On-Hand Qty and Release Lock"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value'])?['ID']`
- Fields:
  - Title: ` ` (clear lock)
  - OnHandQty: `@{outputs('Compute_New_Qty')}`
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Issue`
  - LastMovementRefId: `@{variables('vId')}`
  - IsActive:

```powerautomate
    @if(greater(outputs('Compute_New_Qty'), 0), true, false)
    ```

- **Settings:**
  - Retry Policy: Exponential
  - Count: 3
  - Interval: PT10S

### Step 13: Set Update Completed Flag

Add **"Set variable"** action:

- Name: vUpdateCompleted
- Value: `true`

### Step 14: Mark Transaction as Posted

Add **"Update item - SharePoint"** action:

**Action Name:** "Mark Tech Transaction Posted"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Posted`
  - PostMessage: `Successfully issued from inventory. Remaining: @{outputs('Compute_New_Qty')}`
  - PostedAt: `utcNow()`
- **Settings:**
  - Retry Policy: Fixed Interval
  - Count: 3
  - Interval: PT5S

### Step 15: Add Compensating Transaction Scope

Add **"Scope"** action named **"Compensate - Rollback on Failure"**

**Configure Run After:**

- Has failed, is skipped, has timed out

Inside Compensate scope:

#### Step 15a: Check if Rollback Needed

Add **"Condition"** action:

- Left: `@{variables('vUpdateCompleted')}`
- Operand: is equal to
- Right: `true`

**If Yes (Need to Rollback):**

##### Rollback Inventory Update

Add **"Update item - SharePoint"** action:

**Action Name:** "Rollback On-Hand to Original"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch_with_Lock')?['value'])?['ID']`
- Fields:
  - Title: ` ` (clear any lock)
  - OnHandQty: `@{variables('vOriginalQty')}`
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Rollback-Issue`
  - LastMovementRefId: `ROLLBACK-@{variables('vId')}`
- **Settings:**
  - Retry Policy: Exponential
  - Count: 5 (critical to succeed)
  - Interval: PT30S

##### Log Rollback

Add **"Create item - SharePoint"** action:

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - FlowName: `TT - Issue → OnHand Upsert`
  - ErrorMessage: `Rollback executed for transaction @{variables('vId')}`
  - Severity: `Critical`
  - Timestamp: `utcNow()`

#### Step 15b: Always Execute - Error Logging

##### Log Error

Add **"Create item - SharePoint"** action:

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - FlowName: `TT - Issue → OnHand Upsert`
  - ErrorMessage: `@{result('Atomic_Transaction_-_Issue_Processing')}`
  - RecordId: `@{variables('vId')}`
  - Severity: `Critical`
  - Timestamp: `utcNow()`

##### Update Transaction with Error

Add **"Update item - SharePoint"** action:

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `Failed to process issue. Rollback @{if(variables('vUpdateCompleted'), 'completed', 'not needed')}. Run: @{variables('vFlowRunId')}`
  - PostedAt: `utcNow()`

##### Send Critical Alert

Add **"Send an email (V2)"** action:

- To: `@{parameters('AdminEmail')}`
- Subject: `CRITICAL: Issue Processing Failed - Rollback @{if(variables('vUpdateCompleted'), 'Executed', 'N/A')}`
- Body:

```powerautomate
Transaction ID: @{variables('vId')}
Part: @{variables('vPart')}
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

### Test Case 5: Multiple Locations

- [ ] If using locations:
  - Issue from specific location
  - Verify only that location's stock affected
  - Other locations remain unchanged

## Configuration Options

### Option 1: Allow Negative Inventory

To allow negative inventory (backorders), remove or modify Step 7:

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
   - Verify PostStatus = "Validated" exactly
   - Check TransactionType = "Issue" (case insensitive)
   - Ensure trigger condition syntax is correct

2. **Stock calculations wrong**
   - Verify float conversion on all quantities
   - Check subtraction expression syntax
   - Test with decimal quantities

3. **Can't find inventory**
   - Ensure Part, Batch, UOM match exactly
   - Check for extra spaces (trimming)
   - Verify Location if using that field

4. **IsActive not updating**
   - Check if() expression syntax
   - Verify comparison with 0
   - Test true/false values

## Expression Reference

### Subtract Quantity

```powerautomate
@sub(
  float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
  float(variables('vQty'))
)
```

### Conditional IsActive

```powerautomate
@if(greater(outputs('Compute_New_Qty'), 0), true, false)
```

### Format Error Message

```powerautomate
Insufficient inventory. Available: @{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']}, Requested: @{variables('vQty')}
```

## Next Steps

Once this flow is working correctly:

- Test all stock scenarios thoroughly
- Verify PO tracking is maintained
- Consider implementing Flow 4 (Autofill) for better UX
- Consider implementing Flow 5 (Recalc) for data integrity
