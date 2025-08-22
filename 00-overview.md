# Power Automate Consumables Tracking - System Overview

## Architecture Summary

This system implements a robust inventory tracking solution using Power Automate and SharePoint Lists. The solution tracks material movements (receives and issues) while maintaining accurate on-hand quantities.

## SharePoint Lists Setup

### 1. Parts List (Master Data)
Create a SharePoint list named **"Parts"** with these columns:

| Column Name | Type | Settings |
|------------|------|----------|
| PartNumber | Single line of text | Required, Indexed |
| Description | Single line of text | Optional |
| MaterialType | Choice/Single line of text | Optional |
| UOM | Single line of text | Required |
| IsActive | Yes/No | Default: Yes |

### 2. PO List (Purchase Orders)
Create a SharePoint list named **"PO List"** with these columns:

| Column Name | Type | Settings |
|------------|------|----------|
| PONumber | Single line of text | Required, Indexed |
| AssemblyNumber | Single line of text | Optional |
| AssemblyDescription | Single line of text | Optional |
| ChargingCode | Single line of text | Optional |
| IsOpen | Yes/No | Default: Yes |

### 3. Tech Transactions (Input Form)
Create a SharePoint list named **"Tech Transactions"** with these columns:

| Column Name | Type | Settings |
|------------|------|----------|
| TransactionType | Choice | Choices: Issue, Receive (Required) |
| PONumber | Single line of text | Conditionally Required |
| PartNumber | Single line of text | Required |
| Batch | Single line of text | Required |
| Qty | Number | Required, Min: 0 |
| UOM | Single line of text | Required |
| Location | Single line of text | Optional |
| PostStatus | Single line of text | Hidden from forms, Default: blank |
| PostMessage | Multiple lines of text | Hidden from forms |
| PostedAt | Date and Time | Hidden from forms |
| PostedBy | Person or Group | Hidden from forms |

### 4. On-Hand Material (Inventory)
Create a SharePoint list named **"On-Hand Material"** with these columns:

| Column Name | Type | Settings |
|------------|------|----------|
| PartNumber | Single line of text | Required, Indexed |
| Batch | Single line of text | Required, Indexed |
| Location | Single line of text | Optional |
| UOM | Single line of text | Required |
| OnHandQty | Number | Default: 0 |
| LastMovementAt | Date and Time | Optional |
| LastMovementType | Single line of text | Optional |
| LastMovementRefId | Single line of text | Optional |
| IsActive | Yes/No | Default: Yes |

## Flow Architecture

The solution consists of 5 Power Automate flows:

1. **Intake Validation Flow** - Validates all incoming transactions
2. **Receive Flow** - Processes validated receive transactions
3. **Issue Flow** - Processes validated issue transactions  
4. **Autofill Flow** - Auto-populates part details (optional)
5. **Recalc Flow** - Nightly maintenance job (optional)

## Transaction Processing Model

### Key Concepts:
- **Additive Model**: Each transaction is recorded as a movement
- **Aggregated Inventory**: On-Hand Material stores one row per Part+Batch+Location
- **Transaction Math**:
  - RECEIVE: Adds quantity to OnHandQty
  - ISSUE: Subtracts quantity from OnHandQty

### Process Flow:
1. Tech creates entry in Tech Transactions
2. Intake Validation Flow validates the entry
3. If valid, sets PostStatus = "Validated"
4. Receive/Issue flows process validated transactions
5. On-Hand Material is updated with new quantities
6. Transaction is marked as "Posted"

## Implementation Order

### Phase 1: Foundation
1. Create all 4 SharePoint lists
2. Build and test Flow 1 (Intake Validation)

### Phase 2: Core Functionality  
3. Build and test Flow 2 (Receive)
4. Build and test Flow 3 (Issue)

### Phase 3: Enhancements (Optional)
5. Build Flow 4 (Autofill)
6. Build Flow 5 (Recalc)

## Common Power Automate Expressions

### Trim and Clean
```
trim(coalesce(triggerBody()?['FieldName'], ''))
```

### Uppercase Conversion
```
toUpper(coalesce(triggerBody()?['TransactionType'], ''))
```

### Numeric Parsing
```
float(coalesce(triggerBody()?['Qty'], 0))
```

### Null Checking
```
equals(length(trim(variables('vPart'))), 0)
```

### Current Timestamp
```
utcNow()
```

## Testing Strategy

### Test Scenarios by Flow:
1. **Validation**: Missing fields, invalid PO, zero quantity
2. **Receive**: New part, existing part, multiple locations
3. **Issue**: Sufficient stock, insufficient stock, exact depletion
4. **Autofill**: Valid part, invalid part
5. **Recalc**: Data drift correction

## Best Practices

### Avoid Infinite Loops
- Use trigger conditions to filter events
- Check PostStatus before processing
- Never modify fields that trigger the same flow

### Error Handling
- Always write user-friendly error messages
- Use PostMessage field for detailed feedback
- Log timestamps for audit trail

### Performance
- Use Top Count: 1 when fetching single records
- Index frequently queried columns
- Use trigger conditions to reduce unnecessary runs

### Debugging
- Add Compose actions to log intermediate values
- Test each flow independently before integration
- Use manual triggers during development