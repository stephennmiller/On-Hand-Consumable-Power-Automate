# Flow 1: Tech Intake Validation

## Purpose
Validates all new Tech Transaction entries, ensuring data quality and business rules are enforced before processing.

## Flow Configuration

**Flow Name:** `TT - Intake Validate`

**Trigger:** When an item is created  
**List:** Tech Transactions

## Step-by-Step Build Instructions

### Step 1: Create New Flow
1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Intake Validate`
4. Choose trigger: **"When an item is created - SharePoint"**
5. Configure:
   - Site Address: Your SharePoint site
   - List Name: Tech Transactions

### Step 2: Initialize Variables
Add 7 **"Initialize variable"** actions:

#### Variable 1: vType
- **Name:** vType
- **Type:** String
- **Value:** 
```
toUpper(coalesce(triggerBody()?['TransactionType'], ''))
```

#### Variable 2: vPO
- **Name:** vPO
- **Type:** String
- **Value:**
```
trim(coalesce(triggerBody()?['PONumber'], ''))
```

#### Variable 3: vPart
- **Name:** vPart
- **Type:** String
- **Value:**
```
trim(coalesce(triggerBody()?['PartNumber'], ''))
```

#### Variable 4: vBatch
- **Name:** vBatch
- **Type:** String
- **Value:**
```
trim(coalesce(triggerBody()?['Batch'], ''))
```

#### Variable 5: vUOM
- **Name:** vUOM
- **Type:** String
- **Value:**
```
trim(coalesce(triggerBody()?['UOM'], ''))
```

#### Variable 6: vLoc
- **Name:** vLoc
- **Type:** String
- **Value:**
```
trim(coalesce(triggerBody()?['Location'], ''))
```

#### Variable 7: vQty
- **Name:** vQty
- **Type:** Float
- **Value:**
```
float(coalesce(triggerBody()?['Qty'], 0))
```

### Step 3: Validate Transaction Type
Add **"Condition"** action:

**Condition Name:** "Check Valid Transaction Type"

**Configure:**
- Click "Edit in advanced mode"
- Paste:
```
@or(equals(variables('vType'),'ISSUE'), equals(variables('vType'),'RECEIVE'))
```

**If No:**
- Add **"Update item - SharePoint"** action
- Site Address: Your site
- List Name: Tech Transactions
- Id: `triggerBody()?['ID']`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `Invalid TransactionType - must be Issue or Receive`
  - PostedAt: `utcNow()`
- Add **"Terminate"** action
  - Status: Succeeded

### Step 4: Validate Quantity
Add **"Condition"** action (outside previous condition):

**Condition Name:** "Check Qty > 0"

**Configure:**
- vQty | is greater than | 0

**If No:**
- Add **"Update item - SharePoint"** action
- Configure same as Step 3 but with:
  - PostMessage: `Quantity must be greater than 0`
- Add **"Terminate"** action

### Step 5: Validate Part Number
Add **"Condition"** action:

**Condition Name:** "Check Part Required"

**Configure:**
- Click "Edit in advanced mode"
- Paste:
```
@greater(length(variables('vPart')), 0)
```

**If No:**
- Update item with PostMessage: `PartNumber is required`
- Terminate

### Step 6: Validate Batch
Add **"Condition"** action:

**Condition Name:** "Check Batch Required"

**Configure:**
```
@greater(length(variables('vBatch')), 0)
```

**If No:**
- Update item with PostMessage: `Batch is required`
- Terminate

### Step 7: Validate UOM
Add **"Condition"** action:

**Condition Name:** "Check UOM Required"

**Configure:**
```
@greater(length(variables('vUOM')), 0)
```

**If No:**
- Update item with PostMessage: `UOM is required`
- Terminate

### Step 8: Validate PO for Issues
Add **"Condition"** action:

**Condition Name:** "Check if Issue Transaction"

**Configure:**
- vType | is equal to | ISSUE

**If Yes:**

#### Step 8a: Check PO Provided
Add **"Condition"** inside Yes branch:

**Configure:**
```
@greater(length(variables('vPO')), 0)
```

**If No:**
- Update item with PostMessage: `PONumber is required for Issue transactions`
- Terminate

#### Step 8b: Fetch PO Record
Add **"Get items - SharePoint"** action (in Yes of 8a):

**Configure:**
- Site Address: Your site
- List Name: PO List
- Filter Query: `PONumber eq '@{variables('vPO')}'`
- Top Count: 1

#### Step 8c: Validate PO Exists and Open
Add **"Condition"** action:

**Configure advanced mode:**
```
@and(
  greater(length(body('Get_items')?['value']), 0),
  equals(first(body('Get_items')?['value'])?['IsOpen'], true)
)
```

**If No:**
- Update item with PostMessage: `PO not found or is closed`
- Terminate

### Step 9: Mark as Validated
Add **"Update item - SharePoint"** action (at the end, outside all conditions):

**Configure:**
- Site Address: Your site
- List Name: Tech Transactions
- Id: `triggerBody()?['ID']`
- Fields:
  - PostStatus: `Validated`
  - PostMessage: ` ` (empty)
  - PostedAt: `utcNow()`

## Testing Checklist

### Test Case 1: Valid Receive
- [ ] Create item with TransactionType = "Receive"
- [ ] Provide all required fields
- [ ] Verify PostStatus = "Validated"

### Test Case 2: Valid Issue
- [ ] Create item with TransactionType = "Issue"
- [ ] Provide valid, open PO Number
- [ ] Verify PostStatus = "Validated"

### Test Case 3: Invalid Transaction Type
- [ ] Create item with TransactionType = "Transfer"
- [ ] Verify PostStatus = "Error"
- [ ] Check PostMessage mentions invalid type

### Test Case 4: Missing Fields
- [ ] Test with empty PartNumber
- [ ] Test with empty Batch
- [ ] Test with zero Qty
- [ ] Each should set appropriate error

### Test Case 5: Invalid PO
- [ ] Issue with non-existent PO
- [ ] Issue with closed PO
- [ ] Issue without PO
- [ ] Each should set appropriate error

## Troubleshooting

### Common Issues:

1. **Flow not triggering**
   - Check list name spelling
   - Verify site URL
   - Check flow is turned On

2. **Expression errors**
   - Verify all variable names match exactly
   - Check parentheses are balanced
   - Use expression builder for complex formulas

3. **PO validation failing**
   - Check PO List column names
   - Verify PONumber values match exactly
   - Check IsOpen is Yes/No column type

## Next Steps
Once this flow is working correctly, proceed to:
- Flow 2: Receive transactions processing
- Flow 3: Issue transactions processing