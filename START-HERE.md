# START HERE - On-Hand Consumable System Implementation Guide

## What You're Building

A complete inventory tracking system in SharePoint + Power Automate that tracks consumable materials through three transaction types:
- **RECEIVE** - Adds inventory when materials arrive from vendors
- **ISSUE** - Removes inventory when materials are consumed by technicians
- **RETURNED** - Removes inventory when defective/damaged materials are returned to vendors for credit/replacement

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
| PartDescription | Single line of text | Required, Max length: 255 |
| DefaultUOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Required) |
| Category | Choice | Optional (e.g., Electronics, Hardware, Consumables) |
| MinStockLevel | Number | Optional, Default: 0 |
| MaxStockLevel | Number | Optional |
| IsActive | Yes/No | Required, Default: Yes |

#### Tech Transactions List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| TechName | Person or Group | Required, Allow single selection |
| TransactionType | Choice | Choices: RECEIVE, ISSUE, RETURNED (Required) |
| Part | Lookup | Required, Allow single selection, Source: Parts Master, Show: PartNumber, Additional fields: PartDescription, DefaultUOM |
| Description | Single line of text | Optional (auto-populated by Flow-04) |
| Batch | Single line of text | Required |
| UOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Optional, defaults from Part) |
| PO | Lookup | Optional (Required for ISSUE/RETURNED), Allow single selection, Source: PO List, Show: PONumber |
| Qty | Number | Required, Min: 0.01, Decimal places: 2 |
| PostStatus | Choice | Choices: New, Validated, Processing, Posted, Error (Default: New) |
| PostMessage | Multiple lines of text | Optional (status messages from flows) |
| PostedAt | Date and Time | Optional (UTC timestamp when posted) |

#### On-Hand Material List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| Part | Lookup | Required, Allow single selection, Source: Parts Master, Show: PartNumber, Additional fields: PartDescription |
| Batch | Single line of text | Required, Indexed |
| UOM | Choice | Choices: EA, BOX, CASE, PK, RL, LB, GAL, etc. (Required) |
| OnHandQty | Number | Required, Default: 0 |
| IsActive | Yes/No | Required, Default: Yes |
| LastMovementAt | Date and Time | Optional |
| LastMovementType | Single line of text | Optional |
| LastMovementRefId | Single line of text | Optional |
| CompositeKey | Single line of text | Optional (future enhancement), Indexed |

**CompositeKey Note**: This is an optional column for future performance optimization. The provided flows do NOT currently populate or use this field. If you wish to implement it:
1. Add the column as shown above
2. Modify FLOW-02 and FLOW-03 to populate it with: `concat(toUpper(trim(coalesce(variables('vPartNumber'), ''))),'|',toUpper(trim(coalesce(variables('vBatch'), ''))),'|',toUpper(trim(coalesce(variables('vUOM'), ''))))`
3. Update Get items filters to use `CompositeKey eq '@{variables('vCompositeKey')}'` instead of the multi-column filter
This optimization can improve query performance in high-volume scenarios but is not required for initial implementation.

#### Flow Error Log List

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Flow name that encountered the error |
| ErrorMessage | Multiple lines of text | Required, full error details and stack trace |
| FlowRunURL | Hyperlink | Optional, link to view flow run (see note below) |
| ItemID | Single line of text | Optional, ID of source transaction that failed |
| Timestamp | Date and Time | Required, when error occurred (UTC) |

**Note**: This is a separate list for flow errors. Tech Transactions uses PostMessage field for status updates.

**FlowRunURL Implementation**: When using the SharePoint connector to set this Hyperlink field, pass a JSON object:
```json
{
  "Url": "@{concat('https://make.powerautomate.com/environments/', workflow()?['tags']?['environmentName'], '/flows/', workflow()?['name'], '/runs/', workflow()?['run']?['name'])}",
  "Description": "View Flow Run"
}
```

#### PO List (Required for ISSUE validation)

| Column Name | Type | Settings |
|-------------|------|----------|
| Title | Single line of text | Default column |
| PONumber | Single line of text | Required, Indexed, Enforce unique values |
| OrderDate | Date and Time | Optional |
| Status | Single line of text | Optional |
| IsOpen | Yes/No | Required, Default: Yes |

### Important Notes on Lookup Columns in Power Automate

When using lookup columns in Power Automate flows:
- To get the lookup ID: `triggerBody()?['Part']?['Id']` or `int(triggerBody()?['PartId'])`
- To get the lookup value (PartNumber): `triggerBody()?['Part']?['Value']` or `triggerBody()?['Part_x003a_PartNumber']?['Value']`
- To get additional fields (PartDescription): `triggerBody()?['Part_x003a_PartDescription']?['Value']`
- For filtering by lookup: Use `Part/Id eq <id>` or `Part/PartNumber eq '<value>'` with `$expand=Part`

### Setting Up Environment Variables (Required)

All flows use Dataverse environment variables for configuration. Here's how to create them:

1. **Navigate to Power Platform Admin Center**
   - Go to https://admin.powerplatform.microsoft.com
   - Select your environment

2. **Create a Solution (if not already created)**
   - In Power Apps, go to Solutions
   - Click "New solution"
   - Name: "Inventory Management System"
   - Publisher: Select or create a publisher

3. **Add Environment Variables to Solution**
   - Inside your solution, click "New" → "More" → "Environment variable"
   - Create the following variables:

   **Variable 1: SharePointSiteUrl** (Required)
   - Display name: SharePoint Site URL
   - Schema name: new_SharePointSiteUrl (auto-generated, note your actual prefix)
   - Data type: Text
   - Current value: Your SharePoint site URL (e.g., https://yourcompany.sharepoint.com/sites/Inventory)
   
   **Variable 2: AdminEmail** (Required)
   - Display name: Admin Email
   - Schema name: new_AdminEmail (auto-generated, note your actual prefix)
   - Data type: Text
   - Current value: Your admin email or distribution list (e.g., inventory-alerts@yourcompany.com)

   **Variable 3: SignalRConnection** (Optional - only for real-time dashboard)
   - Display name: SignalR Connection String
   - Schema name: new_SignalRConnection (auto-generated, note your actual prefix)
   - Data type: Text (or Secret for production)
   - Current value: Your Azure SignalR connection string (if using real-time features)
   - Note: Only needed if implementing real-time dashboard updates in FLOW-06

4. **Using Environment Variables in Flows**
   - In expressions, use the schema name with environment() function
   - **Important**: Schema names include auto-generated prefixes based on your solution publisher
   - Common patterns:
     - Default publisher: `environment('new_SharePointSiteUrl')`
     - Custom publisher: `environment('cr123_SharePointSiteUrl')` 
     - With underscores: `environment('contoso_SharePointSiteUrl')`
   
   **Schema Name Mapping Guide** (replace with your actual schema names):
   | Documentation Reference | Your Schema Name (Example) | Expression to Use |
   |------------------------|---------------------------|-------------------|
   | SharePointSiteUrl | new_SharePointSiteUrl | `environment('new_SharePointSiteUrl')` |
   | AdminEmail | new_AdminEmail | `environment('new_AdminEmail')` |
   | SignalRConnection | new_SignalRConnection | `environment('new_SignalRConnection')` |

**Note**: Environment variables are environment-specific. When moving between Dev/Test/Prod, update the "Current value" for each environment without changing the flows.

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
- Create a PO in PO List (PONumber: "PO001", IsOpen: Yes)

1. Create a RECEIVE for Part "ABC123", Batch "B001", Qty 100
2. Verify On-Hand Material shows 100 units
3. Create an ISSUE for same part/batch, PO "PO001", Qty 30
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
   - Check ETag-based optimistic locking prevents race conditions

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
| "Flow not triggering" | Verify trigger condition. All flows assume PostStatus returns a string value:<br/>`@equals(triggerBody()?['PostStatus'], 'Validated')`<br/>Note: If your SharePoint returns PostStatus as an object, update all trigger conditions to use:<br/>`@equals(triggerBody()?['PostStatus']?['Value'], 'Validated')` |
| "PostStatus not updating" | Check exact column name spelling and that it's a Choice column with correct values |
| "Duplicate inventory rows" | Fix OData filter: `Part/Id eq @{variables('vPartId')} and Batch eq '@{variables('vBatch')}' and UOM eq '@{variables('vUOM')}'` |
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
