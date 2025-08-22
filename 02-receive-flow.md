# Flow 2: Post RECEIVES to On-Hand

## Purpose
Processes validated RECEIVE transactions to add inventory to the On-Hand Material list. Creates new inventory records or updates existing ones.

## Flow Configuration

**Flow Name:** `TT - Receive → OnHand Upsert`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:** (Critical to prevent loops!)
```
@and(
  equals(triggerOutputs()?['body/PostStatus'], 'Validated'),
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'RECEIVE')
)
```

## Step-by-Step Build Instructions

### Step 1: Create New Flow
1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Receive → OnHand Upsert`
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
  equals(toUpper(triggerOutputs()?['body/TransactionType']), 'RECEIVE')
)
```
5. Click **"Done"**

### Step 3: Initialize Variables
Add 6 **"Initialize variable"** actions:

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

### Step 6: Configure YES Branch (Update Existing)

In the **Yes** branch, add **"Update item - SharePoint"** action:

**Action Name:** "Update Existing On-Hand"

**Configure:**
- Site Address: Your site
- List Name: On-Hand Material
- Id: `first(body('Get_On-Hand_for_Part+Batch')?['value'])?['ID']`
- Fields:
  - OnHandQty:
    ```
    add(
      float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
      float(variables('vQty'))
    )
    ```
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Receive`
  - LastMovementRefId: `@{variables('vId')}`
  - IsActive: `true`

### Step 7: Configure NO Branch (Create New)

In the **No** branch, add **"Create item - SharePoint"** action:

**Action Name:** "Create New On-Hand"

**Configure:**
- Site Address: Your site
- List Name: On-Hand Material
- Fields:
  - PartNumber: `@{variables('vPart')}`
  - Batch: `@{variables('vBatch')}`
  - UOM: `@{variables('vUOM')}`
  - Location: `@{variables('vLoc')}`
  - OnHandQty: `@{variables('vQty')}`
  - LastMovementAt: `utcNow()`
  - LastMovementType: `Receive`
  - LastMovementRefId: `@{variables('vId')}`
  - IsActive: `true`

### Step 8: Mark Transaction as Posted
Add **"Update item - SharePoint"** action (outside the condition):

**Action Name:** "Mark Tech Transaction Posted"

**Configure:**
- Site Address: Your site
- List Name: Tech Transactions
- Id: `@{variables('vId')}`
- Fields:
  - PostStatus: `Posted`
  - PostMessage: `Successfully added to inventory`
  - PostedAt: `utcNow()`

### Step 9: Add Debug Compose (Optional)
Add **"Compose"** action after Step 4:

**Action Name:** "Debug - On-Hand Results"

**Inputs:**
```
{
  "FoundRows": @{length(body('Get_On-Hand_for_Part+Batch')?['value'])},
  "FirstRow": @{first(body('Get_On-Hand_for_Part+Batch')?['value'])},
  "WillUpdate": @{greater(length(body('Get_On-Hand_for_Part+Batch')?['value']), 0)}
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

### Common Issues:

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
```
add(
  float(first(body('Get_On-Hand_for_Part+Batch')?['value'])?['OnHandQty']),
  float(variables('vQty'))
)
```

### Get First Item ID
```
first(body('Get_On-Hand_for_Part+Batch')?['value'])?['ID']
```

### Check Empty Results
```
@greater(length(body('Get_On-Hand_for_Part+Batch')?['value']), 0)
```

## Next Steps
Once this flow is working correctly:
- Test with multiple receive scenarios
- Verify On-Hand Material aggregates correctly
- Proceed to Flow 3: Issue transactions processing