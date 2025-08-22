# Flow 4: Description Autofill (Optional)

## Purpose

Automatically populates Description and UOM fields when a technician enters a PartNumber, improving data entry speed and accuracy.

## Flow Configuration

**Flow Name:** `TT - Autofill Description`

**Trigger:** When an item is created or modified  
**List:** Tech Transactions

**Trigger Condition:**

```powerautomate
@and(
  equals(coalesce(triggerOutputs()?['body/PostStatus'], ''), ''),
  greater(length(coalesce(triggerBody()?['PartNumber'], '')), 0)
)
```

## When to Implement This Flow

This flow is **OPTIONAL** but recommended if:

- Technicians frequently enter part numbers manually
- You want to reduce data entry errors
- Part descriptions are standardized in the Parts list
- You want to enforce consistent UOM usage

Skip this flow if:

- Using barcode scanning that populates all fields
- Part numbers are rarely entered manually
- Description/UOM vary by context

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Autofill Description`
4. Choose trigger: **"When an item is created or modified - SharePoint"**
5. Configure:
   - Site Address: Your SharePoint site
   - List Name: Tech Transactions

### Step 2: Set Trigger Condition

1. Click the three dots on the trigger → **"Settings"**
2. Expand **"Trigger Conditions"**
3. Click **"Add"**
4. Paste the trigger condition:

```powerautomate
@and(
  equals(coalesce(triggerOutputs()?['body/PostStatus'], ''), ''),
  greater(length(coalesce(triggerBody()?['PartNumber'], '')), 0)
)
```

1. Click **"Done"**

**Note:** This ensures the flow only runs when:

- PostStatus is empty (not yet processed)
- PartNumber has been entered

### Step 3: Initialize Variables

Add 2 **"Initialize variable"** actions:

#### Variable 1: vPartNumber

- **Name:** vPartNumber
- **Type:** String
- **Value:** `trim(coalesce(triggerBody()?['PartNumber'], ''))`

#### Variable 2: vCurrentDesc

- **Name:** vCurrentDesc
- **Type:** String
- **Value:** `coalesce(triggerBody()?['Description'], '')`

### Step 4: Check if Description Already Exists

Add **"Condition"** action:

**Condition Name:** "Description Empty?"

**Configure:**

- Click "Edit in advanced mode"
- Paste:

```powerautomate
@equals(length(variables('vCurrentDesc')), 0)
```

**If No:**

- Skip to end (description already populated)

**If Yes:**

- Continue with Step 5

### Step 5: Get Part Details (In YES Branch)

Add **"Get items - SharePoint"** action:

**Action Name:** "Get Part from Master"

**Configure:**

- Site Address: Your site
- List Name: Parts
- Filter Query: `PartNumber eq '@{replace(variables('vPartNumber'),'''','''''')}'`
- Top Count: 1

### Step 6: Check if Part Found

Add **"Condition"** action:

**Condition Name:** "Part Found in Master?"

**Configure:**

- Click "Edit in advanced mode"
- Paste:

```powerautomate
@greater(length(body('Get_Part_from_Master')?['value']), 0)
```

### Step 7: Update Transaction (In YES Branch)

Add **"Update item - SharePoint"** action:

**Action Name:** "Autofill Fields"

**Configure:**

- Site Address: Your site
- List Name: Tech Transactions
- Id: `triggerBody()?['ID']`
- Fields to Update:
  - Description: `first(body('Get_Part_from_Master')?['value'])?['Description']`
  - UOM: `@{if(equals(length(coalesce(triggerBody()?['UOM'], '')), 0), first(body('Get_Part_from_Master')?['value'])?['UOM'], triggerBody()?['UOM'])}`

### Step 8: Optional - Log Part Not Found (In NO Branch of Step 6)

You can optionally add logging or notification:

Add **"Compose"** action:

**Action Name:** "Log - Part Not Found"

**Inputs:**

```json
{
  "Message": "Part not found in master list",
  "PartNumber": "@{variables('vPartNumber')}",
  "TransactionId": "@{triggerBody()?['ID']}",
  "Timestamp": "@{utcNow()}"
}
```

## Advanced Configuration

### Option 1: Populate Additional Fields

You can autofill more fields from the Parts master:

- MaterialType
- MinOrderQty
- PreferredVendor
- StandardCost

Add these to the Update item action in Step 7.

### Option 2: Smart UOM Validation

Add validation to ensure the entered UOM matches the part's standard UOM:

```powerautomate
@if(
  equals(triggerBody()?['UOM'], first(body('Get_Part_from_Master')?['value'])?['UOM']),
  triggerBody()?['UOM'],
  first(body('Get_Part_from_Master')?['value'])?['UOM']
)
```

### Option 3: Default Values

Set default values when creating new transactions:

Add to Update item when part is found:

- Location: `coalesce(triggerBody()?['Location'], 'FLOOR')`
- Batch: `coalesce(triggerBody()?['Batch'], formatDateTime(utcNow(), 'yyyyMMdd'))`

## Testing Checklist

### Test Case 1: Valid Part Number

- [ ] Create new Tech Transaction
- [ ] Enter known PartNumber only
- [ ] Save and verify Description auto-populates
- [ ] Verify UOM auto-populates

### Test Case 2: Invalid Part Number

- [ ] Enter non-existent PartNumber
- [ ] Verify no error occurs
- [ ] Fields remain empty (graceful failure)

### Test Case 3: Pre-filled Description

- [ ] Create transaction with Description already filled
- [ ] Enter PartNumber
- [ ] Verify Description NOT overwritten

### Test Case 4: Edit Existing Transaction

- [ ] Edit existing transaction
- [ ] Change PartNumber
- [ ] Verify Description updates

## Performance Considerations

### Throttling Prevention

If you have high transaction volume:

1. Add **"Delay"** action after trigger:
   - Count: 2
   - Unit: Second

2. Use caching for frequently used parts:
   - Consider a separate "Part Cache" list
   - Update cache daily

### Conditional Execution

Only run for manual entries:

- Check if Created By is not a service account
- Skip if multiple fields populated simultaneously (likely import)

## Troubleshooting

### Common Issues

1. **Flow runs too frequently**
   - Check trigger condition is correct
   - Verify PostStatus check is working
   - Consider adding delay

2. **Description not populating**
   - Verify PartNumber exact match
   - Check Parts list has Description filled
   - Ensure no extra spaces in PartNumber

3. **Overwriting existing data**
   - Add condition to check if Description empty
   - Only update empty fields

4. **Performance impact**
   - Add Top Count: 1 to Get items
   - Index PartNumber column in Parts list
   - Consider caching strategy

## Integration with Other Flows

### Timing Considerations

This flow should complete BEFORE validation flow:

1. User enters PartNumber
2. Autofill populates Description/UOM (this flow)
3. User enters remaining fields
4. User saves
5. Validation flow runs

### Preventing Conflicts

- This flow only runs when PostStatus is empty
- Validation flow sets PostStatus immediately
- No overlap or loop risk

### Additional Race Condition Prevention

To ensure the Validation flow doesn't run before Autofill completes, add a trigger condition to the Validation flow:

**In the Validation Flow Trigger Settings:**

Add Trigger Condition:
```powerautomate
@or(
  not(equals(coalesce(triggerOutputs()?['body/PostStatus'], ''), '')),
  greater(length(coalesce(triggerBody()?['Description'], '')), 0)
)
```

This ensures Validation flow only runs when:
- PostStatus is NOT empty (already processed), OR
- Description field has been populated (Autofill has run)

**Alternative Approach:**

Set both flows to:
- Concurrency Control: On
- Degree of Parallelism: 1

This ensures sequential processing of each transaction.

## Next Steps

### If Implementing

1. Test with common part numbers
2. Monitor performance impact
3. Gather user feedback on usefulness

### If Skipping

- Proceed directly to Flow 5 (Recalc)
- Consider implementing later if users request

### Enhancement Ideas

- Add vendor lookup
- Auto-populate standard quantities
- Suggest batch numbers based on patterns
- Integrate with barcode scanning
