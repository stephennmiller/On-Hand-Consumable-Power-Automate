# Flow 1: Tech Intake Validation

## Purpose

Validates all new Tech Transaction entries, ensuring data quality and business rules are enforced before processing.

## Flow Configuration

**Flow Name:** `TT - Intake Validate`

**Trigger:** When an item is created  
**List:** Tech Transactions

**Note:** Configure concurrency control at the trigger level and retry policies at individual action levels as described in the steps below.

## Step-by-Step Build Instructions

### Step 1: Create New Flow

1. Go to Power Automate
2. Click **"Create"** → **"Automated cloud flow"**
3. Name: `TT - Intake Validate`
4. Choose trigger: **"When an item is created - SharePoint"**
5. Configure:
   - Site Address: `@{parameters('SharePointSiteUrl')}`
   - List Name: Tech Transactions
6. **Trigger Settings:**
   - Click three dots → Settings
   - Concurrency: 1 (sequential processing)

### Step 2: Add Error Handling Scope

Add **"Scope"** action named **"Try - Main Logic"**

All subsequent steps (except error handling) go inside this scope.

### Step 3: Initialize Variables

Inside the Try scope, add 8 **"Initialize variable"** actions:

#### Variable 1: vType

- **Name:** vType
- **Type:** String
- **Value:**

```powerautomate
toUpper(coalesce(triggerBody()?['TransactionType'], ''))
```

#### Variable 2: vPO

- **Name:** vPO
- **Type:** String
- **Value:**

```powerautomate
trim(coalesce(triggerBody()?['PONumber'], ''))
```

#### Variable 3: vPart

- **Name:** vPart
- **Type:** String
- **Value:**

```powerautomate
trim(coalesce(triggerBody()?['PartNumber'], ''))
```

#### Variable 4: vBatch

- **Name:** vBatch
- **Type:** String
- **Value:**

```powerautomate
trim(coalesce(triggerBody()?['Batch'], ''))
```

#### Variable 5: vUOM

- **Name:** vUOM
- **Type:** String
- **Value:**

```powerautomate
trim(coalesce(triggerBody()?['UOM'], ''))
```

#### Variable 6: vLoc

- **Name:** vLoc
- **Type:** String
- **Value:**

```powerautomate
trim(coalesce(triggerBody()?['Location'], ''))
```

#### Variable 7: vQty

- **Name:** vQty
- **Type:** Float
- **Value:**

```powerautomate
float(coalesce(triggerBody()?['Qty'], 0))
```

#### Variable 8: vFlowRunId

- **Name:** vFlowRunId
- **Type:** String
- **Value:**

```powerautomate
workflow()?['run']?['name']
```

### Step 4: Add Duplicate Check

Add **"Get items - SharePoint"** action:

**Action Name:** "Check for Duplicate Transaction"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Filter Query:

```powerautomate
PartNumber eq '@{replace(variables('vPart'),'''','''''')}' and Batch eq '@{replace(variables('vBatch'),'''','''''')}' and Qty ge @{sub(float(variables('vQty')), 0.01)} and Qty le @{add(float(variables('vQty')), 0.01)} and Created ge '@{formatDateTime(addMinutes(utcNow(), -1), 'yyyy-MM-ddTHH:mm:ssZ')}' and ID ne @{triggerBody()?['ID']}
```

- Top Count: 1

**Add Condition:** "Is Duplicate?"

- Left: `length(body('Check_for_Duplicate_Transaction')?['value'])`
- Operand: is greater than
- Right: 0

**If Yes:**

- Update item: PostStatus='Error', PostMessage='Duplicate transaction detected within 1 minute'
- Terminate: Status=Succeeded

### Step 5: Validate Transaction Type

Add **"Condition"** action:

**Condition Name:** "Check Valid Transaction Type"

**Configure:**

- Click "Edit in advanced mode"
- Paste:

```powerautomate
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

### Step 6: Validate Quantity

Add **"Condition"** action (outside previous condition):

**Condition Name:** "Check Qty > 0"

**Configure:**

- vQty | is greater than | 0

**If No:**

- Add **"Update item - SharePoint"** action
- Configure same as Step 5 but with:
  - PostMessage: `Quantity must be greater than 0`
- Add **"Terminate"** action

### Step 7: Validate Part Number

Add **"Condition"** action:

**Condition Name:** "Check Part Required"

**Configure:**

- Click "Edit in advanced mode"
- Paste:

```powerautomate
@greater(length(variables('vPart')), 0)
```

**If No:**

- Update item with PostMessage: `PartNumber is required`
- Terminate

### Step 8: Validate Batch

Add **"Condition"** action:

**Condition Name:** "Check Batch Required"

**Configure:**

```powerautomate
@greater(length(variables('vBatch')), 0)
```

**If No:**

- Update item with PostMessage: `Batch is required`
- Terminate

### Step 9: Validate UOM

Add **"Condition"** action:

**Condition Name:** "Check UOM Required"

**Configure:**

```powerautomate
@greater(length(variables('vUOM')), 0)
```

**If No:**

- Update item with PostMessage: `UOM is required`
- Terminate

### Step 10: Validate PO for Issues

Add **"Condition"** action:

**Condition Name:** "Check if Issue Transaction"

**Configure:**

- vType | is equal to | ISSUE

**If Yes:**

#### Step 10a: Check PO Provided

Add **"Condition"** inside Yes branch:

**Configure:**

```powerautomate
@greater(length(variables('vPO')), 0)
```

**If No:**

- Update item with PostMessage: `PONumber is required for Issue transactions`
- Terminate

#### Step 10b: Fetch PO Record with Retry

Add **"Get items - SharePoint"** action (in Yes of 10a):

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: PO List
- Filter Query: `PONumber eq '@{replace(variables('vPO'), '''', '''''')}'`
- Top Count: 1
- **Settings:**
  - Retry Policy: Exponential
  - Count: 3
  - Interval: PT10S

#### Step 10c: Validate PO Exists and Open

Add **"Condition"** action:

**Configure advanced mode:**

```powerautomate
@and(
  greater(length(body('Get_items')?['value']), 0),
  equals(first(body('Get_items')?['value'])?['IsOpen'], true)
)
```

**If No:**

- Update item with PostMessage: `PO not found or is closed`
- Terminate

### Step 11: Mark as Validated

Add **"Update item - SharePoint"** action (at the end of Try scope):

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `triggerBody()?['ID']`
- Fields:
  - PostStatus: `Validated`
  - PostMessage: ` ` (empty)
  - PostedAt: `utcNow()`
- **Settings:**
  - Retry Policy: Fixed Interval
  - Count: 3
  - Interval: PT5S

### Step 12: Add Error Handling Scope

Add **"Scope"** action named **"Catch - Error Handling"**

**Configure Run After:**

- Click three dots → Settings → Configure run after
- Check: has failed, is skipped, has timed out

Inside Catch scope:

#### Step 12a: Log Error

Add **"Create item - SharePoint"** action:

**Action Name:** "Log Error to List"

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - FlowName: `TT - Intake Validate`
  - ErrorMessage: `@{concat('Error validating transaction ID: ', triggerBody()?['ID'], ' - ', substring(string(result('Try_-_Main_Logic')),0,min(4000,length(string(result('Try_-_Main_Logic'))))))}`
  - StackTrace: `@{variables('vFlowRunId')}`
  - RecordId: `@{triggerBody()?['ID']}`
  - Severity: `Critical`
  - Timestamp: `utcNow()`

#### Step 12b: Update Transaction with Error

Add **"Update item - SharePoint"** action:

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Tech Transactions
- Id: `triggerBody()?['ID']`
- Fields:
  - PostStatus: `Error`
  - PostMessage: `System error during validation. Admin notified. Run ID: @{variables('vFlowRunId')}`
  - PostedAt: `utcNow()`

#### Step 12c: Send Alert Email

Add **"Send an email (V2)"** action:

**Configure:**

- To: `@{parameters('AdminEmail')}`
- Subject: `CRITICAL: Intake Validation Flow Error`
- Body:

```text
Flow: TT - Intake Validate
Run ID: @{variables('vFlowRunId')}
Transaction ID: @{triggerBody()?['ID']}
Error: @{result('Try_-_Main_Logic')}
Time: @{utcNow()}

Please investigate immediately.
```

### Step 13: Add Finally Scope (Optional)

Add **"Scope"** action named **"Finally - Cleanup"**

**Configure Run After:**

- Check all: is successful, has failed, is skipped, has timed out

Inside Finally scope:

- Add any cleanup actions if needed
- Log completion metrics

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

### Common Issues

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
