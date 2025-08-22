# Power Automate Error Handling Patterns

## Overview

This guide provides comprehensive error handling patterns for production-ready Power Automate flows, ensuring resilience, proper logging, and recovery capabilities.

## Core Error Handling Pattern: Try-Catch-Finally

### Basic Structure

Every production flow should implement this three-scope pattern:

```yaml
Scope: Try
  - Main business logic
  - All primary actions
  
Scope: Catch
  Configure run after: Failed, Skipped, TimedOut
  - Error logging
  - Notifications
  - Status updates
  
Scope: Finally
  Configure run after: Succeeded, Failed, Skipped, TimedOut
  - Cleanup actions
  - Release locks
  - Performance logging
```

## Implementation Guide

### Step 1: Try Scope Setup

Add **"Scope"** action named **"Try - Main Logic"**

Inside this scope:

- Place all main business logic
- Include validations
- Include data operations

**Best Practice:** Keep Try scope focused on happy path

### Step 2: Catch Scope Setup

Add **"Scope"** action named **"Catch - Error Handling"**

**Configure Run After:**

1. Click three dots → Settings
2. Configure run after
3. Check: has failed, is skipped, has timed out
4. Uncheck: is successful

Inside Catch scope, implement these standard actions:

#### 2a. Capture Error Details

Add **"Compose"** action:

**Action Name:** "Capture Error Info"

**Inputs:**

```json
{
  "ErrorTime": "@{utcNow()}",
  "FlowName": "@{workflow()?['name']}",
  "RunId": "@{workflow()?['run']?['name']}",
  "TryResult": "@{result('Try_-_Main_Logic')}",
  "TryBody": "@{body('Try_-_Main_Logic')}",
  "TriggerData": "@{triggerBody()}"
}
```

#### 2b. Log to SharePoint

Add **"Create item - SharePoint"** action:

**Configure:**

- Site Address: `@{parameters('SharePointSiteUrl')}`
- List Name: Flow Error Log
- Fields:
  - FlowName: `@{workflow()?['name']}`
  - ErrorMessage: `@{result('Try_-_Main_Logic')}`
  - StackTrace: `@{outputs('Capture_Error_Info')}`
  - RecordId: `@{triggerBody()?['ID']}`
  - Severity: Use condition to determine
  - Timestamp: `utcNow()`

#### 2c. Send Alert (Conditional)

Add **"Condition"** action:

**Configure:** Check severity

```powerautomate
@contains(result('Try_-_Main_Logic'), 'Critical')
```

**If Yes:**

- Send email to admin
- Create high-priority ticket
- Trigger emergency workflow

### Step 3: Finally Scope Setup

Add **"Scope"** action named **"Finally - Cleanup"**

**Configure Run After:**

1. Check all: is successful, has failed, is skipped, has timed out

Inside Finally scope:

- Release any locks
- Log performance metrics
- Clean up temporary data

## Retry Policy Configuration

### Standard Retry Settings

For all SharePoint actions:

```json
{
  "type": "exponential",
  "count": 3,
  "interval": "PT10S",
  "maximumInterval": "PT1H"
}
```text

### Action-Specific Settings

#### Critical Operations (e.g., Inventory Updates)

```json
{
  "type": "exponential",
  "count": 5,
  "interval": "PT30S",
  "maximumInterval": "PT5M"
}
```

#### Non-Critical Operations (e.g., Logging)

```json
{
  "type": "fixed",
  "count": 2,
  "interval": "PT5S"
}
```text

#### Lock Acquisition (Fail Fast)

```json
{
  "type": "none"
}
```

## Error Categories and Handling

### 1. Transient Errors (Retry)

- **429 Too Many Requests**
  - Action: Exponential backoff
  - Max retries: 5
  - Alert after: 3 failures

- **503 Service Unavailable**
  - Action: Wait and retry
  - Delay: 30 seconds minimum
  - Max retries: 3

### 2. Data Errors (Don't Retry)

- **400 Bad Request**
  - Action: Log and notify
  - Update record status
  - No retry

- **404 Not Found**
  - Action: Check if deleted
  - Create if appropriate
  - Log discrepancy

### 3. Permission Errors (Alert)

- **401 Unauthorized**
  - Action: Immediate admin alert
  - Check service account
  - No retry

- **403 Forbidden**
  - Action: Permission audit
  - Admin notification
  - Manual intervention

## Compensating Transactions Pattern

For operations that modify data:

### Compensating Transaction Implementation

```text
1. Store original state
2. Perform operation
3. On failure:
   a. Check if operation partially completed
   b. Rollback to original state
   c. Log compensation
4. Verify final state
```

### Example: Inventory Issue Rollback

```yaml
Variables:
- vOriginalQty: Store before update
- vUpdateCompleted: Track if update happened

Compensate Scope:
  If vUpdateCompleted = true:
    Update On-Hand:
      OnHandQty = vOriginalQty
      LastMovementType = "Rollback"
```

## Circuit Breaker Pattern

Prevent cascading failures:

### Circuit Breaker Implementation

```yaml
Variables:
- vFailureCount: Integer (0)
- vCircuitOpen: Boolean (false)

Before Main Operation:
  Check if vCircuitOpen:
    If true: Skip operation, return cached/default
    
After Failure:
  Increment vFailureCount
  If vFailureCount > 3:
    Set vCircuitOpen = true
    Send critical alert
```

## Error Notification Strategy

### Severity Levels

#### Critical (Immediate Alert)

- Data corruption risk
- Complete flow failure
- Security issues
- Rollback failures

**Actions:**

- Email admin immediately
- Create P1 ticket
- SMS/Teams alert
- Pause related flows

#### High (Alert within 15 min)

- Partial failures
- Performance degradation
- Repeated errors

**Actions:**

- Email to support team
- Create P2 ticket
- Dashboard update

#### Medium (Batch Alert)

- Single record failures
- Retry succeeded
- Non-critical validations

**Actions:**

- Log to list
- Daily summary email
- Dashboard metrics

#### Low (Log Only)

- Expected errors
- User input errors
- Duplicate attempts

**Actions:**

- Log to list
- Weekly report
- No immediate action

## Monitoring and Alerting Rules

### Alert Aggregation

```yaml
Condition: 3+ errors in 5 minutes
Action: Escalate to Critical
Reason: Indicates systemic issue
```

### Performance Degradation

```yaml
Condition: Execution time > 3x average
Action: Warning alert
Reason: Possible throttling or data issue
```

### Failure Rate

```yaml
Condition: >10% failure rate in hour
Action: Pause flow, alert admin
Reason: Prevent data corruption
```

## Testing Error Handlers

### Test Scenarios

1. **Force SharePoint Throttling**
   - Create loop with 1000 rapid calls
   - Verify exponential backoff works
   - Check error logging

2. **Force Controlled Failure**
   - Add a Condition → Terminate (Failed) when testing flag is set
   - Verify Catch scope runs and logs correctly
   - Optionally call an HTTP action to a non-routable IP (e.g., 10.255.255.1) to simulate timeout

3. **Test Partial Failure**
   - Update record then force error
   - Verify rollback executes
   - Check data consistency

4. **Test Lock Contention**
   - Run parallel instances
   - Verify lock handling
   - Check queue behavior

## Error Recovery Procedures

### Manual Recovery Steps

1. **Identify Scope**

```text
   Query: PostStatus eq 'Error' and Created gt '[Date]'
   Action: Export to Excel for analysis
   ```

1. **Categorize Errors**
   - Group by error message
   - Identify patterns
   - Prioritize by impact

1. **Bulk Retry**

```text
   For each error record:
     Reset PostStatus to ''
     Clear PostMessage
     Flow will reprocess
   ```

1. **Verify Recovery**
   - Check all retried records
   - Validate data consistency
   - Update documentation

## Best Practices

### DO

- ✅ Always use Try-Catch-Finally pattern
- ✅ Set appropriate retry policies
- ✅ Log all errors with context
- ✅ Implement compensating transactions
- ✅ Test error paths thoroughly
- ✅ Monitor error rates continuously
- ✅ Document recovery procedures
- ✅ Use correlation IDs for tracing

### DON'T

- ❌ Swallow errors silently
- ❌ Retry non-idempotent operations
- ❌ Use generic error messages
- ❌ Ignore throttling warnings
- ❌ Mix error handling with business logic
- ❌ Forget to release locks on error
- ❌ Alert for every single error
- ❌ Assume retries will always succeed

## Performance Impact

### Error Handling Overhead

- Try-Catch: ~100ms per flow run
- Logging: ~200ms per error
- Compensation: ~500ms when triggered
- Email alerts: ~1000ms per send

### Optimization Tips

- Batch error logs (every 5 min)
- Use async patterns for notifications
- Cache frequently used error data
- Implement smart retry delays

## Troubleshooting Guide

### Common Issues

1. **Catch Block Not Triggering**
   - Check run after configuration
   - Verify scope names match
   - Test with forced error

2. **Infinite Retry Loop**
   - Add retry counter
   - Set maximum retry limit
   - Implement circuit breaker

3. **Compensation Failing**
   - Increase retry count
   - Add manual intervention flag
   - Log compensation attempts

4. **Error Logs Missing**
   - Check permissions on log list
   - Verify connection references
   - Test logging separately

## Integration with Azure Monitor

### Application Insights Integration

```json
{
  "action": "Send telemetry",
  "properties": {
    "EventName": "FlowError",
    "FlowName": "@{workflow()?['name']}",
    "ErrorType": "@{result('Try_-_Main_Logic')}",
    "Severity": "Error",
    "CustomDimensions": {
      "RunId": "@{workflow()?['run']?['name']}",
      "RecordId": "@{triggerBody()?['ID']}"
    }
  }
}
```text

### Log Analytics Queries

```kusto
// Flows with high error rates
PowerAutomateErrors
| where TimeGenerated > ago(1h)
| summarize ErrorCount = count() by FlowName
| where ErrorCount > 10
| order by ErrorCount desc

// Error patterns
PowerAutomateErrors
| where TimeGenerated > ago(24h)
| summarize Count = count() by ErrorMessage, FlowName
| order by Count desc
```

## Conclusion

Implementing robust error handling is crucial for production Power Automate flows. Follow these patterns to ensure your flows are resilient, maintainable, and provide clear visibility into issues when they occur.
