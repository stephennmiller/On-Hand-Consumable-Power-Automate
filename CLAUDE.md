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
3. **Processing** → Being handled by receive/issue/returned flows
4. **Posted** → Successfully completed
5. **Error** → Failed with logged reason

### Transaction Types

- **RECEIVE** → Adds inventory (incoming materials from vendors)
- **ISSUE** → Removes inventory (materials consumed by technicians)
- **RETURNED** → Removes inventory (defective/damaged items returned to vendor for credit/replacement)
  - Behaves identically to ISSUE in terms of inventory decrement
  - Requires valid PO reference like ISSUE transactions
  - Used for tracking defective materials sent back to suppliers

### Concurrency Control

- Uses ETag-based optimistic locking for inventory updates (primary mechanism)
- ProcessingLock column is **optional** and not used by the provided flows - ETag is the primary and sufficient locking mechanism
- Trigger concurrency settings:
  - **ISSUE and RETURNED flows**: Set to 1 (critical for preventing race conditions)
    - Flow settings path: Trigger → Settings → Concurrency Control → On, Degree of Parallelism: 1
  - **RECEIVE flows**: Can use 3-5 parallelism for better throughput
    - Flow settings path: Trigger → Settings → Concurrency Control → On, Degree of Parallelism: 3
- In the ISSUE/RETURNED flow, a Title-based sentinel is set during lock acquisition to provide visibility during retries (optional aid; ETag remains the primary lock)

### Error Recovery

- Automatic rollback on failures after inventory modification
- Circuit breaker pattern in advanced flows
- Comprehensive error logging to Flow Error Log list

## Critical Implementation Details

### SharePoint Column Names (Exact Names Required)

The flows use these specific column names:
- Tech Transactions: `Part` (lookup), `Description` (text), `Qty`, `Batch`, `UOM`, `PO` (lookup)
- On-Hand Material: `Part` (lookup), `OnHandQty`, `Batch`, `UOM`
- Additional columns: `IsActive`, `LastMovementAt`, `LastMovementType`, `LastMovementRefId`
- Parts Master: `PartNumber`, `PartDescription`, `DefaultUOM`
- Flow Error Log: `Title` (flow name), `ErrorMessage` (multi-line for full stack traces), `FlowRunURL` (run URL, optional), `ItemID` (source transaction ID), `Timestamp`

Recommended types/formatting:
- `Qty`, `OnHandQty`: Number with 2 decimals (store as number; use `round()` only when writing).
- `PONumber`: Single line of text (Enforce unique = Yes).
- `UOM`: Choice (EA, BOX, CASE, PK, RL, LB, GAL, etc.)
- `IsActive`: Yes/No.
- `LastMovementAt`: Date and Time (UTC). Use `utcNow()` when writing.
  - Note: SharePoint list views may display in the site's local timezone; configure the column as "Include time" and be mindful of regional settings when validating timestamps.
  - Example (display in Eastern time): `@{convertTimeZone(utcNow(), 'UTC', 'Eastern Standard Time')}`
- `LastMovementType`, `LastMovementRefId`: Single line of text.
- Flow Error Log → `ItemID`: Single line of text; `Timestamp`: Date and Time (UTC); `FlowRunURL`: Hyperlink (recommended); `Title`: Single line of text (flow name); `ErrorMessage`: Multiple lines of text (plain text, no HTML formatting - preserves full error payloads and stack traces).
  - **Setting Hyperlink fields (SharePoint connector):** Pass a JSON object with Url and Description:
    ```json
    {
      "Url": "@{variables('vRunUrl')}",
      "Description": "View Flow Run"
    }
    ```
  - **Setting Hyperlink fields (SharePoint REST):** Use SP.FieldUrlValue shape (nometadata payload omits `__metadata`):
    ```json
    { 
      "FlowRunURL": { 
        "Url": "@{variables('vRunUrl')}", 
        "Description": "View Flow Run" 
      } 
    }
    ```

### Required Environment Variables (Dataverse)

- `environment('SharePointSiteUrl')` - Your SharePoint site URL
- `environment('AdminEmail')` - Email address for critical alerts (can be a distribution list or shared mailbox)

**Note**: Use `environment()` for Dataverse environment variables in cloud flows. Do not use `parameters()` for environment variables; `parameters()` refers to flow-level parameters. Create the variables inside a Solution and set a per-environment "Current value" before running flows.
- Example usage in expressions:
  - Site: `@{environment('SharePointSiteUrl')}`
  - Admin: `@{environment('AdminEmail')}`
  - Note: environment() takes the environment variable's schema name (not display name).

### Required SharePoint Lists

Five lists must be created with exact column names (create Parts Master first):

Master List:
1. **Parts Master** - Part catalog with descriptions and default UOM

Transaction Lists:
2. **Tech Transactions** - Incoming transaction records (uses lookup to Parts and PO)
3. **On-Hand Material** - Current inventory state (uses lookup to Parts)
4. **PO List** - Purchase order validation
5. **Flow Error Log** - Error tracking and debugging

PO List minimum columns:
- `PONumber` (Single line of text, Enforce unique = Yes)
- `IsOpen` (Yes/No)
- Optional metadata: `Vendor` (Text), `PODate` (Date), `Notes` (Multiline)
Filter example (FLOW-01 validation) (note: Yes/No is a boolean; use `true`/`false` without quotes):
- Pre-step: Initialize variable `vPONumberEscaped` = `@{replace(variables('vPONumber'), '''', '''''')}`
- Get items (Filter Query field):
  `PONumber eq '@{variables('vPONumberEscaped')}' and IsOpen eq true`
- REST (Uri querystring):
  `?$filter=PONumber eq '@{variables('vPONumberEscaped')}' and IsOpen eq true&$top=1`
Action setting (Get items): set "Top Count" = `1` (since `PONumber` is unique)

### Working with Lookup Columns

- Accessing lookup values in triggers/actions:
  - Lookup ID: `triggerBody()?['Part']?['Id']` or `int(triggerBody()?['PartId'])`
  - Lookup primary field: `triggerBody()?['Part']?['Value']`
  - Additional lookup fields: `triggerBody()?['Part_x003a_PartDescription']?['Value']`
- Filtering by lookups in OData:
  - By ID: `Part/Id eq 123`
  - By value: `Part/PartNumber eq 'ABC123'` (requires `$expand=Part`)
- Setting lookup values:
  - SharePoint connector: Use the ID directly
  - REST API: `{ "PartId": 123 }` or `{ "Part@odata.bind": "Parts(123)" }`

### Expression Syntax Notes

- Choice column access:
  - When SharePoint returns string: `triggerOutputs()?['body/PostStatus']`
  - When SharePoint returns object: `triggerOutputs()?['body/PostStatus']?['Value']`
  - Example condition (object): `@equals(triggerOutputs()?['body/PostStatus']?['Value'], 'Validated')`
  - Example condition (string): `@equals(triggerOutputs()?['body/PostStatus'], 'Validated')`
  - Tip: Use Peek code on the trigger/Get item to confirm whether the Choice value is a string or an object (with `Value`) before choosing an expression.
  - Note: `triggerBody()` and `triggerOutputs()?['body']` are equivalent—use one form consistently in a given flow to improve searchability and reduce copy-paste errors.
- When writing Choice values:
  - Single-select (SharePoint connector): pass a string (e.g., `'Validated'`).
  - Single-select (SharePoint REST, nometadata): send `{ "YourChoiceFieldInternalName": "Validated" }`.
  - Multi-select (SharePoint connector actions): pass an array of strings (e.g., `['Open','Urgent']`).
  - Multi-select (SharePoint REST via "Send an HTTP request to SharePoint"):
    `{ "YourChoiceFieldInternalName": { "results": ["Open","Urgent"] } }`.
- Always escape single quotes in filters: `replace(variables('vPartNumber'),'''','''''')`
- Float conversion required for calculations: `float(variables('vQty'))`
- Round to 2 decimals for quantities: `round(float(variables('vQty')), 2)`
- Avoid formatNumber before math operations - format only for display
- Null handling with coalesce: `coalesce(triggerBody()?['FieldName'], '')`
- SharePoint list internal name encoding: spaces become `_x0020_`, hyphens become `_x002d_`

### Indexing Recommendations

**Tech Transactions List**:
- Index: `Part`, `Batch`, `PO`, `PostStatus`, `UOM`
- These support filter queries in Get items operations
- Note: Each additional index can slow writes; prioritize columns used in $filter and lookups, then revisit indexes after observing real query patterns.

**On-Hand Material List**:
- Index: `Part`, `Batch`, `UOM`
- Prefer a dedicated text column `CompositeKey` (indexed) populated by the flow.
- Prefer normalized key (trim + upper-case):
- `concat(
  toUpper(trim(coalesce(variables('vPartNumber'), ''))),'|',
  toUpper(trim(coalesce(variables('vBatch'), ''))),'|',
  toUpper(trim(coalesce(variables('vUOM'), '')))
  )`
- Use a delimiter like `|` that won't appear in data. If a component may contain `|`, sanitize:
  `replace(toUpper(trim(coalesce(variables('vPartNumber'), ''))), '|', '¦')`
  (repeat for other components). This keeps CompositeKey stable in $filter and lookups.

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
    - Example (variable): Initialize `vETag` = the item's `@odata.etag`, then set header:
      - If-Match: `@{variables('vETag')}`
      - Anti-pattern to avoid: `If-Match: "@{variables('vETag')}"` (adds extra quotes and causes 400/412)
  - Use either:
    - POST + header `X-HTTP-Method: MERGE`, or
    - PATCH (if your HTTP action/tenant supports it)
  - Example (Send an HTTP request to SharePoint — preferred):
    - Site Address: `@{environment('SharePointSiteUrl')}`
    - Method: POST
    - Uri (resilient): `/_api/web/lists(guid'{YourListGuid}')/items(@{int(variables('vOnHandId'))})`
      - Alternative (title-based): `/_api/web/lists/getbytitle('On-Hand Material')/items(@{int(variables('vOnHandId'))})`
      - If your list title contains a single quote, escape it as two single quotes inside the literal:
        `getbytitle('ACME''s Materials')`
      - Tip (find List GUID): Site contents → open the list → Settings; the URL contains `List=%7B<GUID>%7D`. Copy the GUID portion including braces.
    - Headers: as above (plus `X-HTTP-Method: MERGE` for POST)
  - Alternative (generic HTTP connector with Entra ID):
    - Add `Authorization: Bearer <token>` and any tenant-required headers (e.g., request digest in classic contexts).
    - This path is more complex and generally not recommended when the SharePoint action is available.
  - On 412 (Precondition Failed), re-read the item to get latest ETag, reapply changes, retry with bounded exponential backoff (e.g., 1s, 2s, 4s; max 3 attempts). See "HTTP Status Codes" section for exact expressions.
- **Rollback Mechanism**: FLOW-03 implements compensating transactions (see "Rollback Mechanism" section in that doc)
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

### Transaction Processing Rates

- **Basic Implementation**: 500-800 transactions/hour
  - Single flow instances, no parallelism
  - Standard SharePoint connector actions
- **Optimized Implementation**: 2,000-3,000 transactions/hour
  - RECEIVE flows with Degree of Parallelism = 3
  - Proper indexing on all filter columns
  - Batch processing where applicable
- **Daily Recalc**: Process 10,000-15,000 items
  - Using SharePoint batch APIs
  - 500-item pages with deterministic ordering
- **Error Rate Target**: <0.1% in production

### Performance Baselines

- **SharePoint Connector Limits**:
  - 600 API calls per minute per flow
  - 2000 API calls per minute per connection
  - Response time: 200-500ms per operation (typical)
- **Recommended Batch Sizes**:
  - Get items: 500 items per page
  - Batch updates: 100 items per batch
  - Daily recalc: Process in 1000-item chunks

**Assumptions**: Standard connectors with E3/E5 licensing, well-maintained list indexes, low contention (<10% concurrent updates), RECEIVE flows running with Degree of Parallelism = 3, and tenant throttling within normal limits.

## Retry Policy Guidelines

### When to Use Fixed Interval

Use **Fixed Interval** retry for:
- Simple read operations (Get items, Get item)
- Non-critical updates that can tolerate delays
- Operations unlikely to face contention
- Settings: Count = 3, Interval = PT2S to PT5S

### When to Use Exponential Backoff

Use **Exponential Backoff** retry for:
- Critical inventory updates
- Operations that might face throttling
- Create operations that might face unique constraints
- Rollback operations (must succeed)
- Settings: Count = 3-5, Initial Interval = PT10S, Max Interval = PT1M

### When to Disable Retry (None)

Set Retry Policy to **None** for:
- Operations inside Do Until loops (to prevent double retry)
- ETag-based update attempts (handle 412 manually)
- Operations where you implement custom retry logic

### HTTP Status Code Handling

- **2xx**: Success, no retry needed
- **400-404**: Client errors, do not retry (except 408, 429)
- **408**: Request timeout, retry with exponential backoff
- **412**: Precondition failed (ETag mismatch), refresh and retry
- **429**: Too many requests, honor Retry-After header
- **5xx**: Server errors, retry with exponential backoff

## Common Troubleshooting Areas

1. **Trigger Conditions**: Exact syntax and Choice column value access

2. **ETag Handling**:
   - Get items → `first(body('Get_OnHand_for_Part_Batch')?['value'])?['@odata.etag']` (after `greater(length(...?['value']), 0)`).
   - Get item → `body('Get_OnHand')?['@odata.etag']`.
   If zero/not found, branch to a not-found handler.

3. **Filter Queries**: Single quote escaping and case sensitivity

4. **Float Conversions**: Required for all numeric operations

5. **Metadata Type Names**: When using `application/json;odata=nometadata`, omit `__metadata` entirely (preferred).
   - If you must use verbose/minimalmetadata, fetch the exact `ListItemEntityTypeFullName` dynamically and use that value.
   - Note: SharePoint encodes spaces (`_x0020_`) and hyphens (`_x002d_`) in the type name (e.g., `SP.Data.On_x002d_Hand_x0020_MaterialListItem`). Do not guess—always use the live value.

6. **Concurrency Issues**: Set trigger concurrency to 1 for ISSUE flows to prevent race conditions

7. **Deterministic Pagination**:
   - Use `$orderby=ID asc` (or another stable key).
   - Set `$top=<pageSize>` (e.g., 500) and enable Pagination in action settings.
   - Set the Pagination threshold high enough to cover the total expected items (it does not need to equal `$top`).
   - The connector manages `$skiptoken` internally when Pagination is On; stable ordering plus idempotent filters helps avoid duplicates on retries.

8. **HTTP Status Codes** (example using FLOW-03's "Lock_OnHand_with_ETag_Attempt" action):
   - Lock success: any 2xx
     `@and(greaterOrEquals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 200), less(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 300))`
   - Retry Policy: set the MERGE/Update HTTP action Retry Policy to `None` (Action → Settings) when using this Do Until pattern to prevent double retries.
   - 412 Precondition Failed → ETag mismatch: re-read the item to refresh ETag, then retry (max 3 attempts)  
     `@equals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 412)`
   - 429 Too Many Requests → throttling: retry up to 3 attempts with bounded exponential backoff + jitter  
     `@equals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 429)`
   - 409 Conflict → unique/index constraint violation or conflicting state:
     - Treat as non-retryable validation/business error (log and exit), unless your scenario intentionally retries idempotent creates.
     `@equals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 409)`
   - 408 Request Timeout → transient: retry with bounded exponential backoff  
     `@equals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 408)`
   - 5xx Server Errors → transient faults: retry up to 3 attempts with bounded exponential backoff  
     `@and(greaterOrEquals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 500), less(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 600))`
   - vAttempt management:
     - Initialize before the loop: Initialize variable `vAttempt` (Integer) = `1`
     - At end of each iteration: Set variable `vAttempt` → `@{add(variables('vAttempt'), 1)}`
   - Do Until (Change limits): set `Count` = `3` and a conservative `Timeout` (e.g., `PT1M`) to avoid runaway loops if variables are mis-set.
   - Delay duration per attempt (vAttempt = 1..3). Prefer `Retry-After` (secs). If only `x-ms-retry-after-ms` is present, convert ms→s. Else use exponential + jitter:
     `PT@{int(coalesce(
        outputs('Lock_OnHand_with_ETag_Attempt')?['headers']?['Retry-After'],
        outputs('Lock_OnHand_with_ETag_Attempt')?['headers']?['retry-after'],
        if(
          or(
            equals(outputs('Lock_OnHand_with_ETag_Attempt')?['headers']?['x-ms-retry-after-ms'], null),
            empty(outputs('Lock_OnHand_with_ETag_Attempt')?['headers']?['x-ms-retry-after-ms'])
          ),
          null,
          string(ceiling(div(float(outputs('Lock_OnHand_with_ETag_Attempt')?['headers']?['x-ms-retry-after-ms']), 1000)))
        ),
        string(add(int(pow(2, sub(variables('vAttempt'), 1))), rand(0,2)))
     ))}S`
   - Do Until termination (complete when success OR attempts exhausted):
     `@or(
       and(greaterOrEquals(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 200), less(outputs('Lock_OnHand_with_ETag_Attempt')?['statusCode'], 300)),
       greaterOrEquals(variables('vAttempt'), 3)
     )`

## Important Files for Reference

- **START-HERE.md**: Prerequisites, SharePoint schemas, implementation order, quick test plan
- **FLOW-03-Issue-Processing.md**: Most complex flow with ETag locking, rollback patterns, and compensating transactions
- **FLOW-05-Daily-Recalc.md**: Advanced patterns including checkpoints, circuit breakers, and batch processing
- **reference-docs/01-error-handling-patterns.md**: Comprehensive error handling strategies

## Implementation Timeline

- **Minimum Viable (3 days)**: FLOW-01, 02, 03 + basic monitoring
- **Standard (1 week)**: Add FLOW-04, 05, 06 + testing
- **Enterprise (1 month)**: Add real-time processing, Power BI dashboard, full automation
