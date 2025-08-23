# START HERE - On-Hand Consumable System Implementation Guide

## What You're Building
A complete inventory tracking system in SharePoint + Power Automate that tracks consumable materials with receive/issue transactions.

## Required Order of Implementation

### Prerequisites (Do This First!)
- [ ] Access to SharePoint site
- [ ] Power Automate license
- [ ] Create these SharePoint lists with exact column names and types:

#### Tech Transactions List
| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| TechName | Single line of text | Required |
| TransactionType | Choice | Choices: RECEIVE, ISSUE (Required) |
| PartNumber | Single line of text | Required |
| PartDescription | Single line of text | Optional |
| Batch | Single line of text | Required |
| Bin | Single line of text | Optional |
| Quantity | Number | Required, Min: 0.01 |
| PostStatus | Choice | Choices: New, Validated, Processing, Posted, Error (Default: New) |
| ProcessingLock | Single line of text | Optional (for concurrency) |
| ErrorMessage | Multiple lines of text | Optional |

#### On-Hand Material List
| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| PartNumber | Single line of text | Required |
| PartDescription | Single line of text | Optional |
| Batch | Single line of text | Required |
| TotalQuantity | Number | Required, Default: 0 |
| LastTransactionID | Single line of text | Optional |
| LastUpdated | Date and Time | Optional |
| ProcessingLock | Single line of text | Optional (for concurrency) |

#### Flow Error Log List
| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Flow name |
| ErrorMessage | Multiple lines of text | Required |
| FlowRunURL | Hyperlink | Optional |
| ItemID | Single line of text | Optional |
| Timestamp | Date and Time | Required |

### Phase 1: Core System (Build These First)
**Build in this exact order:**

1. **[FLOW-01-Intake-Validation.md](FLOW-01-Intake-Validation.md)** ⭐ START HERE
   - Validates all incoming transactions
   - Time to build: 15 minutes
   - Test before moving on: Create a test transaction, verify it gets "Validated" status

2. **[FLOW-02-Receive-Processing.md](FLOW-02-Receive-Processing.md)** 
   - Processes RECEIVE transactions to add inventory
   - Time to build: 20 minutes
   - Test: Create a RECEIVE transaction, verify inventory increases

3. **[FLOW-03-Issue-Processing.md](FLOW-03-Issue-Processing.md)**
   - Processes ISSUE transactions to remove inventory
   - Time to build: 30 minutes (more complex due to locking)
   - Test: Create an ISSUE transaction, verify inventory decreases

**✅ MILESTONE: You now have a working inventory system!**

### Phase 2: Enhancements (Optional but Recommended)

4. **[FLOW-04-Autofill-Optional.md](FLOW-04-Autofill-Optional.md)**
   - Auto-populates item descriptions
   - Time to build: 10 minutes
   - Makes data entry easier

5. **[FLOW-05-Daily-Recalc.md](FLOW-05-Daily-Recalc.md)**
   - Daily reconciliation of all inventory
   - Time to build: 45 minutes
   - Ensures data accuracy

6. **[FLOW-06-Monitoring-Alerts.md](FLOW-06-Monitoring-Alerts.md)**
   - Alerts for errors and performance issues
   - Time to build: 30 minutes
   - Keeps system healthy

## Quick Test Plan

After building Phase 1, test with this sequence:

### Basic Functionality Tests
1. Create a RECEIVE for Part "ABC123", Batch "B001" (batch number), Qty 100
2. Verify On-Hand Material shows 100 units
3. Create an Issue for same part/batch, Qty 30
4. Verify On-Hand Material shows 70 units
5. Check Flow Error Log is empty

### Edge Case Tests
6. **Test Idempotency**: Create duplicate RECEIVE transaction
   - Verify no duplicate inventory rows created
   - Confirm total quantity remains correct

7. **Test Insufficient Stock**: Create Issue for Qty 200 (more than available)
   - Verify transaction marked as Error
   - Check error message in Flow Error Log
   - Confirm inventory unchanged

8. **Test Concurrent Processing**: Create 5 transactions rapidly
   - Verify all process correctly without conflicts
   - Check ProcessingLock prevents race conditions

9. **Test Invalid Data**: Create transaction with negative quantity
   - Verify validation catches error
   - Check appropriate error logged

## Common Issues & Fixes

| Problem | Solution |
|---------|----------|
| "Flow not triggering" | Verify trigger condition: `@equals(triggerOutputs()?['body/PostStatus']?['Value'], 'Validated')` |
| "PostStatus not updating" | Check exact column name spelling and that it's a Choice column with correct values |
| "Duplicate inventory rows" | Fix OData filter: `PartNumber eq '@{variables('PartNumber')}' and Batch eq '@{variables('Batch')}'` |
| "Negative inventory allowed" | Add condition in FLOW-03 Step 12: `greater(variables('CurrentQuantity'), variables('TransactionQuantity'))` |
| "ETag mismatch errors" | Ensure using `outputs('Get_inventory_record')?['body/@odata.etag']` in Update item |

## File Organization

```text
Main Flow Documentation (Build These):
├── FLOW-01-Intake-Validation.md    ← Start here
├── FLOW-02-Receive-Processing.md   ← Build second
├── FLOW-03-Issue-Processing.md     ← Build third
├── FLOW-04-Autofill-Optional.md    ← Optional enhancement
├── FLOW-05-Daily-Recalc.md         ← Daily reconciliation
└── FLOW-06-Monitoring-Alerts.md    ← System monitoring

Reference Documentation (Read if Needed):
└── reference-docs/
    ├── 00-architecture-overview.md
    ├── 01-error-handling-patterns.md
    ├── 02-performance-optimization.md
    └── (other advanced topics)
```

## Success Criteria

You know your system is working when:
- ✅ Transactions automatically get validated
- ✅ RECEIVE transactions increase inventory
- ✅ ISSUE transactions decrease inventory
- ✅ No duplicate inventory rows created
- ✅ Error log stays empty during normal operation

## Next Steps

1. Start with **[FLOW-01-Intake-Validation.md](FLOW-01-Intake-Validation.md)**
2. Build and test completely before moving to next flow
3. Each flow file has exact click-by-click instructions
4. Copy/paste the expressions exactly as shown

---

**Questions?** Each flow file has a Troubleshooting section at the bottom.