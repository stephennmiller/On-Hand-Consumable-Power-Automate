# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **documentation-only repository** for implementing an enterprise-grade consumables inventory tracking system using Microsoft Power Automate and SharePoint. There is no executable code - all files are Markdown documentation guides for building Power Automate flows.

**Primary Use Case**: Organizations tracking consumable materials with automated receive/issue workflows, supporting 10,000+ daily transactions when fully optimized.

## Project Structure

### Core Implementation Files (Build in Order)

1. `START-HERE.md` - Entry point with prerequisites and quick test plan
2. `FLOW-01-Intake-Validation.md` - Transaction validation flow
3. `FLOW-02-Receive-Processing.md` - Process RECEIVE transactions  
4. `FLOW-03-Issue-Processing.md` - Process ISSUE transactions with locking
5. `FLOW-04-Autofill-Optional.md` - Auto-populate descriptions (optional)
6. `FLOW-05-Daily-Recalc.md` - Daily reconciliation (optional)
7. `FLOW-06-Monitoring-Alerts.md` - System monitoring (optional)

### Reference Documentation

Located in `reference-docs/`:
- Architecture overviews, error handling patterns, performance optimization
- Advanced topics like real-time processing, migration plans, automated testing
- Power BI dashboard configuration

## Key Architecture Concepts

### Transaction Flow Pattern

1. **New** → Transaction created in Tech Transactions list
2. **Validated** → Passes business rules (triggers processing flows)
3. **Processing** → Being handled by receive/issue flows
4. **Posted** → Successfully completed
5. **Error** → Failed with logged reason

### Concurrency Control

- Uses ETag-based optimistic locking for inventory updates (primary mechanism)
- Optional: You may add a boolean `ProcessingLock` column for UI clarity/guardrails, but it is not required by the provided flows
- Trigger concurrency set to 1 for ISSUE flows
  - Flow settings path: Trigger → Settings → Concurrency Control → On, Degree of Parallelism: 1
  - Note: RECEIVE flows can use 2-5 parallelism for better throughput
  - In the provided ISSUE flow, a Title-based sentinel is set during lock acquisition to provide visibility during retries (optional aid; ETag remains the primary lock)

### Error Recovery

- Automatic rollback on failures after inventory modification
- Circuit breaker pattern in advanced flows
- Comprehensive error logging to Flow Error Log list

## Critical Implementation Details

### SharePoint Column Names (Exact Names Required)

The flows use these specific column names:
- Tech Transactions: `Qty` (not Quantity)
- On-Hand Material: `OnHandQty` (not TotalQuantity)
- Additional columns: `UOM`, `Location`, `PONumber` in Tech Transactions
- On-Hand columns: `IsActive`, `LastMovementAt`, `LastMovementType`, `LastMovementRefId`
- Flow Error Log: `Title` (flow name), `ErrorMessage`, `FlowRunURL` (run URL, optional), `ItemID` (source transaction ID), `Timestamp`

Recommended types/formatting:
- `Qty`, `OnHandQty`: Number with 2 decimals (store as number; use `round()` only when writing).
- `PONumber`: Single line of text (Enforce unique = Yes).
- `UOM`, `Location`: Single line of text (or Choice if your flows expect strict options—document your choice consistently across files).
- `IsActive`: Yes/No.
- `LastMovementAt`: Date and Time (UTC). Use `utcNow()` when writing.
- `LastMovementType`, `LastMovementRefId`: Single line of text.
- Flow Error Log → `ItemID`: Number; `FlowRunURL`: Hyperlink or Picture; others: Single line of text.
  - Writing `FlowRunURL` (SharePoint connector): supply an object
    `{ "Url": "@{variables('vRunUrl')}", "Description": "Run details" }`
  - Writing via SharePoint REST: use SP.FieldUrlValue shape (nometadata payload omits `__metadata`):
    `{ "FlowRunURL": { "Url": "@{variables('vRunUrl')}", "Description": "Run details" } }`

### Required Environment Variables (Dataverse)

- `environment('SharePointSiteUrl')` - Your SharePoint site URL
- `environment('AdminEmail')` - Email address for critical alerts (can be a distribution list or shared mailbox)

**Note**: Use `environment()` for Dataverse environment variables in cloud flows. Do not use `parameters()` for environment variables; `parameters()` refers to flow-level parameters. Create the variables inside a Solution and set a per-environment "Current value" before running flows.
- Example usage in expressions:
  - Site: `@{environment('SharePointSiteUrl')}`
  - Admin: `@{environment('AdminEmail')}`

### Required SharePoint Lists

Four lists must be created with exact column names:
1. **Tech Transactions** - Incoming transaction records
2. **On-Hand Material** - Current inventory state
3. **Flow Error Log** - Error tracking and debugging
4. **PO List** - Purchase order validation used by FLOW-01 (Intake-Validation, IsOpen check); optionally referenced by ISSUE processing

PO List minimum columns:
- `PONumber` (Single line of text, Enforce unique = Yes)
- `IsOpen` (Yes/No)
- Optional metadata: `Vendor` (Text), `PODate` (Date), `Notes` (Multiline)
Filter example (FLOW-01 validation):
`$filter=PONumber eq '@{variables('vPONumberEscaped')}' and IsOpen eq true`
Action setting: set "Top Count" = `1` (since `PONumber` is unique)

### Expression Syntax Notes

- Choice column access:
  - When SharePoint returns string: `triggerOutputs()?['body/PostStatus']`
  - When SharePoint returns object: `triggerOutputs()?['body/PostStatus']?['Value']`
  - Example condition (object): `@equals(triggerOutputs()?['body/PostStatus']?['Value'], 'Validated')`
  - Example condition (string): `@equals(triggerOutputs()?['body/PostStatus'], 'Validated')`
  - Tip: Use Peek code on the trigger/Get item to confirm whether the Choice value is a string or an object (with `Value`) before choosing an expression.
  - Note: `triggerBody()` and `triggerOutputs()?['body']` are equivalent—use one form consistently in a given flow to improve searchability and reduce copy-paste errors.
- When writing Choice values:
  - Single-select: pass a string (e.g., `'Validated'`).
  - Multi-select (SharePoint connector actions): pass an array of strings (e.g., `['Open','Urgent']`).
  - Multi-select (SharePoint REST via "Send an HTTP request to SharePoint"): 
    send `{ "YourChoiceFieldInternalName": { "results": ["Open","Urgent"] } }`.
- Always escape single quotes in filters: `replace(variables('vPart'),'''','''''')`
- Float conversion required for calculations: `float(variables('vQty'))`
- Round to 2 decimals for quantities: `round(float(variables('vQty')), 2)`
- Avoid formatNumber before math operations - format only for display
- Null handling with coalesce: `coalesce(triggerBody()?['FieldName'], '')`
- SharePoint list internal name encoding: spaces become `_x0020_`, hyphens become `_x002d_`

### Indexing Recommendations

**Tech Transactions List**:
- Index: `PartNumber`, `Batch`, `PONumber`, `PostStatus`, `UOM`, `Location`
- These support filter queries in Get items operations
- Note: Each additional index can slow writes; prioritize columns used in $filter and lookups

**On-Hand Material List**:
- Index: `PartNumber`, `Batch`, `UOM`, `Location`
- Prefer a dedicated text column `CompositeKey` (indexed) populated by the flow.
- Prefer normalized key (trim + upper-case):
- `concat(toUpper(trim(variables('vPartNumber'))),'|',coalesce(toUpper(trim(variables('vBatch'))),''),'|',toUpper(trim(variables('vUOM'))),'|',toUpper(trim(variables('vLocation'))))`
- Use a delimiter like `|` that won't appear in data. This key works reliably in $filter and supports lookups.

**Flow Error Log**:
- Index: `ItemID` (recommended), `Timestamp` or `Created` (recommended), `Title` (optional)
- Rationale: Speeds up incident triage, time-window filtering, and cross-referencing source transactions.

**PO List**:
- Index: `PONumber` (required for FLOW-01 intake-validation lookups)
- Enforce unique values on `PONumber` (List settings → select the `PONumber` column → set "Enforce unique values" = Yes). SharePoint will auto-index the column

## Documentation Maintenance

When updating documentation:
1. Maintain consistency between START-HERE.md schemas and flow implementations
2. Use exact Power Automate expression syntax (no pseudocode)
3. Include step numbers for clear build instructions
4. Document both happy path and error scenarios
5. Provide complete troubleshooting sections

## Known Complexities

- **FLOW-03 Locking Pattern**: Uses HTTP MERGE with ETag for optimistic locking, requires careful error handling
  - Send `If-Match: <etag>` with MERGE/Update; success returns 204 No Content
  - Required headers (SharePoint REST):
    - `Accept: application/json;odata=nometadata`
    - `Content-Type: application/json;odata=nometadata`
    - `If-Match: "<etag>"` (pass exactly as returned by the API; avoid adding/removing quotes)
  - Use either:
    - POST + header `X-HTTP-Method: MERGE`, or
    - PATCH (if your HTTP action/tenant supports it)
  - Example (Send an HTTP request to SharePoint — preferred):
    - Site Address: `@{environment('SharePointSiteUrl')}`
    - Method: POST
    - Uri: `/_api/web/lists/getbytitle('On-Hand Material')/items(@{variables('vOnHandId')})`
    - Headers: as above (plus `X-HTTP-Method: MERGE` for POST)
  - Alternative (generic HTTP connector with Entra ID):
    - Add `Authorization: Bearer <token>` and any tenant-required headers (e.g., request digest in classic contexts).
    - This path is more complex and generally not recommended when the SharePoint action is available.
  - On 412 (Precondition Failed), re-read the item to get latest ETag, reapply changes, retry with bounded exponential backoff (e.g., 1s, 2s, 4s; max 3 attempts)
- **Rollback Mechanism**: Step 16 in FLOW-03 implements compensating transactions
- **Batch Processing**: FLOW-05 uses SharePoint batch APIs for significant performance improvement (results vary by tenant and list size)
- **Circuit Breaker**: Advanced flows implement circuit breaker pattern to prevent cascade failures

## Testing Approach

Each flow includes specific test cases:
- Normal operations (happy path)
- Edge cases (zero quantity, exact depletion)
- Error conditions (insufficient stock, missing data)
- Concurrency scenarios (parallel transactions)
- Throttling scenarios (HTTP 429 with Retry-After handling)
- Transient server faults (HTTP 5xx with bounded exponential backoff)

## Performance Targets

- Basic Implementation: 500 transactions/hour
- Optimized Implementation: 2,500 transactions/hour  
- Daily Recalc: Process 10,000+ items
- Error Rate: <0.1% target

**Assumptions**: Standard connectors, well-maintained list indexes, low contention, RECEIVE flows running with Degree of Parallelism > 1, and E3/E5 tenant throttling within norms.

## Common Troubleshooting Areas

1. **Trigger Conditions**: Exact syntax and Choice column value access

2. **ETag Handling**: Proper capture and usage for locking - use `first(body('Get_OnHand_for_Part_Batch')?['value'])?['@odata.etag']` after checking `greater(length(body('Get_OnHand_for_Part_Batch')?['value']), 0)`. If zero, branch to a not-found handler.

3. **Filter Queries**: Single quote escaping and case sensitivity

4. **Float Conversions**: Required for all numeric operations

5. **Metadata Type Names**: SharePoint list internal names - e.g., `SP.Data.On_x002d_Hand_x0020_MaterialListItem`

6. **Concurrency Issues**: Set trigger concurrency to 1 for ISSUE flows to prevent race conditions

7. **Deterministic Pagination**:
   - Use `$orderby=ID asc` (or another stable key).
   - Set `$top=<pageSize>` (e.g., 500) and enable Pagination in action settings.
   - Set the Pagination threshold high enough to cover the total expected items (it does not need to equal `$top`).
   - The connector manages `$skiptoken` internally when Pagination is On; stable ordering plus idempotent filters helps avoid duplicates on retries.

8. **HTTP Status Codes**:
   - Lock success = 204 (and in some cases 200 for PATCH)
     `@or(equals(outputs('Lock_Action')?['statusCode'], 204), equals(outputs('Lock_Action')?['statusCode'], 200))`
     (replace `Lock_Action` with your actual action name)
   - 412 Precondition Failed → ETag mismatch: re-read the item to refresh ETag, then retry (max 3 attempts)  
     `@equals(outputs('Your_Http_Action_Name')?['statusCode'], 412)`
   - 429 Too Many Requests → throttling: retry up to 3 attempts with bounded exponential backoff + jitter  
     `@equals(outputs('Your_Http_Action_Name')?['statusCode'], 429)`
   - 5xx Server Errors → transient faults: retry up to 3 attempts with bounded exponential backoff  
     `@and(greaterOrEquals(outputs('Your_Http_Action_Name')?['statusCode'], 500), less(outputs('Your_Http_Action_Name')?['statusCode'], 600))`
   - Delay duration per attempt (vAttempt = 1..3). Prefer `Retry-After` (secs). If only `x-ms-retry-after-ms` is present, convert ms→s. Else use exponential + jitter:
     `PT@{coalesce(
        int(outputs('Your_Http_Action_Name')?['headers']?['Retry-After']),
        int(outputs('Your_Http_Action_Name')?['headers']?['retry-after']),
        ceiling(div(float(outputs('Your_Http_Action_Name')?['headers']?['x-ms-retry-after-ms']), 1000)),
        add(int(pow(2, sub(variables('vAttempt'), 1))), rand(0,2))
     )}S`
   - Do Until termination (complete when success OR attempts exhausted):
     `@or(equals(outputs('Lock_Action')?['statusCode'], 204), greaterOrEquals(variables('vAttempt'), 3))`

## Important Files for Reference

- **START-HERE.md**: Prerequisites, SharePoint schemas, implementation order, quick test plan
- **FLOW-03-Issue-Processing.md**: Most complex flow with ETag locking, rollback patterns, and compensating transactions
- **FLOW-05-Daily-Recalc.md**: Advanced patterns including checkpoints, circuit breakers, and batch processing
- **reference-docs/01-error-handling-patterns.md**: Comprehensive error handling strategies

## Implementation Timeline

- **Minimum Viable (3 days)**: FLOW-01, 02, 03 + basic monitoring
- **Standard (1 week)**: Add FLOW-04, 05, 06 + testing
- **Enterprise (1 month)**: Add real-time processing, Power BI dashboard, full automation
