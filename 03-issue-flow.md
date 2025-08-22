# Flow 3: Post ISSUES to On-Hand

## Purpose
Processes validated ISSUE transactions to remove inventory from the On-Hand Material list. Includes stock availability checking and prevents negative inventory.

## Flow Configuration

**Flow Name:** `TT - Issue → OnHand Upsert`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:** (Critical to prevent loops!)
```
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'ISSUE')
)
```

## Step-by-Step Build Instructions

### Step 1: Create New Flow
1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Issue → OnHand Upsert`
4. Choose trigger: **"When an item is created or modified - SharePoint"**
5. Configure:
   - Site Address: Your SharePoint site
   - List Name: Tech Transactions

### Step 2: Set Trigger Condition
1. Click the three dots on the trigger → **"Settings"**
2. Expand **"Trigger Conditions"**
3. Click **"Add"**
4. Paste the trigger condition:
```
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'ISSUE')
)
```
5. Click **"Done"**

### Step 3: Initialize Variables
Add 7 **"Initialize variable"** actions:

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

### Step 4: Get Matching On-Hand Row
Add **"Get items - SharePoint"** action:

**Action Name:** "Get On-Hand for Part+Batch"

**Configure:**
- Site Address: Your site
- List Name: On-Hand Material
- Filter Query:
```
PartNumber eq '@{variables('vPart')}' and Batch eq '@{variables('vBatch')}' and UOM eq '@{variables('vUOM')}'
```

**Note:** If using Location field, add to filter:
```
PartNumber eq '@{variables('vPart')}' and Batch eq '@{variables('vBatch')}' and UOM eq '@{variables('vUOM')}' and Location eq '@{variables('vLoc')}'
```

- Top Count: 1

### Step 5: Check if Row Exists
Add **"Condition"** action:

**Condition Name:** "On-Hand Row Exists?"

**Configure:**
- Click "Edit in advanced mode"
- Paste:
```
@greater(length(body('Get_On-Hand_for_Part+Batch')?['value']), 0)
```

**If No:**
- Add **"Update item - SharePoint"** action
- Site Address: Your site
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `No inventory found for Part: @{variables('vPart')}, Batch: @{variables('vBatch')}`
  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 6: Compute New Quantity (In YES Branch)
Add **"Compose"** action in the YES branch:

**Action Name:** "Compute New Qty"

**Inputs:**
```
@sub(
  float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
  float(variables('vQty'))
)
```

### Step 7: Check Sufficient Stock
Add **"Condition"** action:

**Condition Name:** "Sufficient Stock?"

**Configure:**
- outputs('Compute_New_Qty') | is greater than or equal to | 0

**If No:**
- Add **"Update item - SharePoint"** action
- Site Address: Your site
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Error`
  - PostMessage: 
    ```
    Insufficient inventory. Available: @{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']}, Requested: @{variables('vQty')}
    ```
  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 8: Update On-Hand (In YES Branch of Step 7)
Add **"Update item - SharePoint"** action:

**Action Name:** "Update On-Hand Qty"

**Configure:**
- Site Address: Your site
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch')?['value'])?['ID']`
- Fields:
  - OnHandQty: `@{outputs('Compute_New_Qty')}`
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Issue`
  - LastMovementRefId: `@{variables('vId')}`
  - IsActive: 
    ```
    @if(greater(outputs('Compute_New_Qty'), 0), true, false)
    ```

### Step 9: Mark Transaction as Posted
Add **"Update item - SharePoint"** action (still in YES branch):

**Action Name:** "Mark Tech Transaction Posted"

**Configure:**
- Site Address: Your site
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Posted`
  - PostMessage: `Successfully issued from inventory. Remaining: @{outputs('Compute_New_Qty')}`
  - PostedAt: `utcNow()`

### Step 10: Add Debug Compose (Optional)
Add **"Compose"** action after Step 6:

**Action Name:** "Debug - Stock Check"

**Inputs:**
```
{
  "CurrentOnHand": @{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']},
  "RequestedQty": @{variables('vQty')},
  "NewQty": @{outputs('Compute_New_Qty')},
  "PONumber": "@{variables('vPO')}"
}
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
```
@greater(outputs('Compute_New_Qty'), 10)
```
This maintains a safety stock of 10 units.

## Troubleshooting

### Common Issues:

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
```
@sub(
  float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
  float(variables('vQty'))
)
```

### Conditional IsActive
```
@if(greater(outputs('Compute_New_Qty'), 0), true, false)
```

### Format Error Message
```
Insufficient inventory. Available: @{first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']}, Requested: @{variables('vQty')}
```

## Next Steps
Once this flow is working correctly:
- Test all stock scenarios thoroughly
- Verify PO tracking is maintained
- Consider implementing Flow 4 (Autofill) for better UX
- Consider implementing Flow 5 (Recalc) for data integrity