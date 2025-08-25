# START HERE - On-Hand Consumable System Implementation Guide

## What You're Building

A complete inventory tracking system in SharePoint + Power Automate that tracks consumable materials with receive/issue transactions.

## Required Order of Implementation

### Prerequisites (Do This First!)

- [ ] Access to SharePoint site
- [ ] Power Automate license
- [ ] Set up Dataverse environment variables:
  - **SharePointSiteUrl**: Your SharePoint site URL
    - Usage in flows: `environment('SharePointSiteUrl')`
  - **AdminEmail**: Email address for critical alerts
    - Usage in flows: `environment('AdminEmail')`
- [ ] Create these SharePoint lists with exact column names and types (create master lists first):

#### Master Lists (Create These First)

##### Parts Master List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column (can be used for display name) |
| PartNumber | Single line of text | Required, Indexed, Enforce unique values |
| PartDescription | Single line of text | Required |
| DefaultUOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Required) |
| Category | Choice | Optional (e.g., Electronics, Hardware, Consumables) |
| MinStockLevel | Number | Optional, Default: 0 |
| MaxStockLevel | Number | Optional |
| IsActive | Yes/No | Required, Default: Yes |

##### Locations Master List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Location code (e.g., A1, B2, W01) |
| LocationName | Single line of text | Required (e.g., "Warehouse 1 Shelf A1") |
| LocationType | Choice | Choices: Warehouse, Store, Transit (Required) |
| IsActive | Yes/No | Required, Default: Yes |

##### Vendors Master List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Vendor code |
| VendorName | Single line of text | Required, Enforce unique values |
| ContactEmail | Single line of text | Optional |
| ContactPhone | Single line of text | Optional |
| IsActive | Yes/No | Required, Default: Yes |

#### Tech Transactions List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| TechName | Person or Group | Required, Allow single selection |
| TransactionType | Choice | Choices: RECEIVE, ISSUE, RETURNED (Required) |
| Part | Lookup | Required, Allow single selection, Source: Parts Master, Show: PartNumber, Additional fields: PartDescription, DefaultUOM |
| Batch | Single line of text | Required |
| Location | Lookup | Optional, Allow single selection, Source: Locations Master, Show: Title |
| UOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Optional, defaults from Part) |
| PO | Lookup | Optional (Required for ISSUE/RETURNED), Allow single selection, Source: PO List, Show: PONumber |
| Qty | Number | Required, Min: 0.01, Decimal places: 2 |
| PostStatus | Choice | Choices: New, Validated, Processing, Posted, Error (Default: New) |
| ProcessingLock | Single line of text | Optional (for concurrency) |
| PostMessage | Multiple lines of text | Optional (status messages) |
| ErrorMessage | Multiple lines of text | Optional (legacy/compatibility) |

#### On-Hand Material List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| Part | Lookup | Required, Allow single selection, Source: Parts Master, Show: PartNumber, Additional fields: PartDescription |
| Batch | Single line of text | Required, Indexed |
| Location | Lookup | Optional, Allow single selection, Source: Locations Master, Show: Title |
| UOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Required) |
| OnHandQty | Number | Required, Default: 0 |
| IsActive | Yes/No | Required, Default: Yes |
| LastMovementAt | Date and Time | Optional |
| LastMovementType | Single line of text | Optional |
| LastMovementRefId | Single line of text | Optional |
| ProcessingLock | Single line of text | Optional (for concurrency) |

**Performance Note**: Consider adding a computed column `Key` with formula `=[Part:PartNumber]&"|"&[Batch]&"|"&[Location:Title]&"|"&[UOM]` and indexing it for faster lookups. This creates a unique composite key for each inventory record.

#### Flow Error Log List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Flow name |
| ErrorMessage | Multiple lines of text | Required |
| FlowRunURL | Hyperlink | Optional |
| ItemID | Single line of text | Optional |
| Timestamp | Date and Time | Required |

#### PO List (Required for ISSUE validation)

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| PONumber | Single line of text | Required, Indexed, Enforce unique values |
| Vendor | Lookup | Required, Allow single selection, Source: Vendors Master, Show: VendorName |
| OrderDate | Date and Time | Optional |
| Status | Single line of text | Optional |
| IsOpen | Yes/No | Required, Default: Yes |

### Important Notes on Lookup Columns in Power Automate

When using lookup columns in Power Automate flows:
- To get the lookup ID: `triggerBody()?['Part']?['Id']` or `triggerBody()?['PartId']?['Value']`
- To get the lookup value (PartNumber): `triggerBody()?['Part']?['Value']` or `triggerBody()?['Part_x003a_PartNumber']?['Value']`
- To get additional fields (PartDescription): `triggerBody()?['Part_x003a_PartDescription']?['Value']`
- For filtering by lookup: Use `Part/Id eq <id>` or `Part/PartNumber eq '<value>'` with `$expand=Part`

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

1. **[FLOW-04-Autofill-Optional.md](FLOW-04-Autofill-Optional.md)**
   - Auto-populates item descriptions
   - Time to build: 10 minutes
   - Makes data entry easier

2. **[FLOW-05-Daily-Recalc.md](FLOW-05-Daily-Recalc.md)**
   - Daily reconciliation of all inventory
   - Time to build: 45 minutes
   - Ensures data accuracy

3. **[FLOW-06-Monitoring-Alerts.md](FLOW-06-Monitoring-Alerts.md)**
   - Alerts for errors and performance issues
   - Time to build: 30 minutes
   - Keeps system healthy

## Quick Test Plan

After building Phase 1, test with this sequence:

### Basic Functionality Tests

**Prerequisites**: 
- Create a part in Parts Master (PartNumber: "ABC123", Description: "Test Part", DefaultUOM: "EA")
- Create a location in Locations Master (Title: "A1", LocationName: "Warehouse Shelf A1")
- Create a vendor in Vendors Master (VendorName: "Test Vendor")
- Create a PO in PO List (PONumber: "PO001", Vendor: "Test Vendor", IsOpen: Yes)

1. Create a RECEIVE for Part "ABC123", Batch "B001", Location "A1", Qty 100
2. Verify On-Hand Material shows 100 units
3. Create an ISSUE for same part/batch/location, PO "PO001", Qty 30
4. Verify On-Hand Material shows 70 units
5. Check Flow Error Log is empty

### Edge Case Tests

1. **Test Idempotency**: Create duplicate RECEIVE transaction
   - Verify no duplicate inventory rows created
   - Confirm total quantity remains correct

2. **Test Insufficient Stock**: Create Issue for Qty 200 (more than available)
   - Verify transaction marked as Error
   - Check error message in Flow Error Log
   - Confirm inventory unchanged

3. **Test Concurrent Processing**: Create 5 transactions rapidly
   - Verify all process correctly without conflicts
   - Check ProcessingLock prevents race conditions

4. **Test Invalid Data**: Create transaction with negative quantity
   - Verify validation catches error
   - Check appropriate error logged

5. **Test RETURNED Transaction**: Create RETURNED for Part "ABC123", Batch "B001", PO "PO001", Qty 10
   - Verify On-Hand Material decreases by 10 units (should show 60 if following from test #3)
   - Confirm transaction posts successfully
   - Check that RETURNED behaves like ISSUE (decrements inventory)

## Common Issues & Fixes

| Problem | Solution |
|---------|----------|
| "Flow not triggering" | Verify trigger condition (use the one that matches your trigger output shape):<br/>`@equals(triggerOutputs()?['body/PostStatus'], 'Validated')`<br/>or (if PostStatus is an object):<br/>`@equals(triggerOutputs()?['body/PostStatus']?['Value'], 'Validated')` |
| "PostStatus not updating" | Check exact column name spelling and that it's a Choice column with correct values |
| "Duplicate inventory rows" | Fix OData filter: `Part/PartNumber eq '@{variables('PartNumber')}' and Batch eq '@{variables('Batch')}'` (use $expand=Part) |
| "Negative inventory allowed" | Add condition in FLOW-03 Step 12: `greaterOrEquals(outputs('Compute_New_Qty'), 0)` to allow exact depletion |
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
