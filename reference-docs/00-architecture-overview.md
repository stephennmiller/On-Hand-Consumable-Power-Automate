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
| UOM | Single line of text | Required |
| OnHandQty | Number | Default: 0 |
| LastMovementAt | Date and Time | Optional |
| LastMovementType | Single line of text | Optional |
| LastMovementRefId | Single line of text | Optional |
| IsActive | Yes/No | Default: Yes |
| OHKey | Single line of text | Enforce unique values, Indexed |

**Important:** The OHKey field must be populated by flows using this formula:
```powerautomate
concat(
  toUpper(trim(coalesce(variables('vPartNumber'), ''))),
  '|',
  toUpper(trim(coalesce(variables('vBatch'), ''))),
  '|',
  toUpper(trim(coalesce(variables('vUOM'), '')))
)
```
This ensures composite uniqueness for Part+Batch+UOM combinations. All flows MUST populate OHKey when creating/updating On-Hand records.

### 5. Flow Error Log (Monitoring)

Create a SharePoint list named **"Flow Error Log"** with these columns:

| Column Name | Type | Settings |
|------------|------|----------|
| FlowName | Single line of text | Required |
| ErrorMessage | Multiple lines of text | Required |
| StackTrace | Multiple lines of text | Optional |
| RecordId | Single line of text | Optional |
| Severity | Choice | Critical, Warning, Info |
| Timestamp | Date and Time | Required |
| ResolvedAt | Date and Time | Optional |
| ResolvedBy | Person or Group | Optional |

## Flow Architecture

The solution consists of 5 core Power Automate flows:

1. **Intake Validation Flow** - Validates all incoming transactions
2. **Receive Flow** - Processes validated receive transactions
3. **Issue Flow** - Processes validated issue transactions  
4. **Autofill Flow** - Auto-populates part details (optional)
5. **Recalc Flow** - Nightly maintenance job (optional)

### Required Compound Indexes

For optimal performance, create these compound indexes in SharePoint:

- **Tech Transactions**: PartNumber + PostStatus
- **On-Hand Material**: PartNumber + Batch + IsActive
- **PO List**: PONumber + IsOpen

### Connection References

All flows should use these connection references:

- **SharePoint Connection**: Service account with contribute permissions
- **Office 365 Outlook**: For error notifications
- **Office 365 Users**: For user information retrieval

## Transaction Processing Model

### Key Concepts

- **Additive Model**: Each transaction is recorded as a movement
- **Aggregated Inventory**: On-Hand Material stores one row per Part+Batch+UOM
- **Transaction Math**:
  - RECEIVE: Adds quantity to OnHandQty
  - ISSUE: Subtracts quantity from OnHandQty

### Process Flow

1. Tech creates entry in Tech Transactions
2. Intake Validation Flow validates the entry
3. If valid, sets PostStatus = "Validated"
4. Receive/Issue flows process validated transactions
5. On-Hand Material is updated with new quantities
6. Transaction is marked as "Posted"

## Implementation Order

### Phase 0: Environment Setup

1. Configure service accounts and permissions
2. Create all SharePoint lists with proper indexes
3. Set up Flow Error Log list
4. Configure environment variables:
   - `SharePointSiteUrl`: Your SharePoint site URL
   - `AdminEmail`: Email for critical alerts
   - `MaxRetryCount`: Default 3
   - `ThrottleDelay`: Default 100ms

### Phase 1: Foundation

1. Import solution with connection references
2. Build and test Flow 1 (Intake Validation)
3. Verify error logging works

### Phase 2: Core Functionality  

1. Build and test Flow 2 (Receive)
2. Build and test Flow 3 (Issue)
3. Test with 100+ transactions for performance

### Phase 3: Enhancements (Optional)

1. Build Flow 4 (Autofill)
2. Build Flow 5 (Recalc)
3. Add monitoring dashboard

### Phase 4: Production Readiness

1. Load test with 1000+ transactions
2. Implement automated testing flows
3. Document rollback procedures

## Common Power Automate Expressions

### Trim and Clean

```powerautomate
trim(coalesce(triggerBody()?['FieldName'], ''))
```

### Uppercase Conversion

```powerautomate
toUpper(coalesce(triggerBody()?['TransactionType'], ''))
```

### Numeric Parsing

```powerautomate
float(coalesce(triggerBody()?['Qty'], 0))
```

### Null Checking

```powerautomate
equals(length(trim(coalesce(triggerBody()?['PartNumber'], ''))), 0)
```

### Current Timestamp

```powerautomate
utcNow()
```

## Testing Strategy

### Test Scenarios by Flow

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
- Set concurrency control to 1 for transaction flows

### Error Handling

- Implement Try-Catch-Finally pattern in all flows
- Log all errors to Flow Error Log list
- Configure retry policies on all SharePoint actions
- Send email alerts for critical failures
- Use compensating transactions for rollback

### Performance

- Implement pagination for >100 items
- Use Select Query to limit returned fields
- Add 100ms delay between bulk operations
- Use batch operations for multiple updates
- Set appropriate concurrency limits (1-5)

### Throttling Protection

- SharePoint: 600 calls/minute per user
- Add exponential backoff retry: PT10S, PT30S, PT90S
- Use service accounts to increase limits
- Monitor 429 (throttling) errors

### Security

- Use app-only authentication for service accounts
- Implement row-level security on lists
- Never log sensitive data (PO details, user info)
- Use environment variables for configuration
- Regular security audits of permissions

### Debugging

- Add Compose actions to log intermediate values
- Test each flow independently before integration
- Use manual triggers during development
- Enable run history for 30 days
- Use version control for flow definitions

## Monitoring and Alerting

### Key Metrics to Track

- Flow run success/failure rates
- Average execution time per flow
- Error frequency and types
- Transaction volume trends
- Stock level alerts

### Alert Thresholds

- 3+ errors in 5 minutes: Email admin
- Flow execution >3 minutes: Warning
- 10+ consecutive failures: Critical alert
- Stock discrepancy >10%: Investigation required

## Deployment Strategy

### Solution Packaging

1. Export as managed solution
2. Include all flows, connections, and environment variables
3. Document dependencies and prerequisites
4. Version control in Git

### Deployment Process

1. Deploy to DEV environment
2. Run automated tests
3. Deploy to UAT with limited users
4. Monitor for 1 week
5. Deploy to PROD with rollback plan

### Rollback Procedures

1. Keep previous solution version
2. Document configuration changes
3. Have manual process ready
4. Test rollback in UAT first
