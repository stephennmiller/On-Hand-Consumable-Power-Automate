# Power Automate Performance Optimization Guide

## Overview

This guide addresses performance optimization for Power Automate flows handling high-volume transactions, preventing timeouts, and ensuring scalability.

## SharePoint List Limits and Solutions

### Understanding the 5000 Item Threshold

SharePoint lists have a view threshold of 5000 items. Queries returning more will fail.

#### Solutions

1. **Indexed Columns (Required)**

```
Tech Transactions:
- PartNumber (Indexed)
- PostStatus (Indexed)
- Created (Indexed)
- Compound: PartNumber + PostStatus

On-Hand Material:
- PartNumber (Indexed)
- Batch (Indexed)
- IsActive (Indexed)
- Compound: PartNumber + Batch + IsActive
```

1. **Filter First on Indexed Columns**

```
✅ Good: PartNumber eq 'ABC123' and Batch eq 'LOT001'
❌ Bad: Description eq 'Some text' and Qty gt 100
```

## Pagination Implementation

### Pattern 1: Simple Pagination (< 100k items)

```yaml
Variables:
- vSkipCount: Integer (0)
- vBatchSize: Integer (100)
- vHasMore: Boolean (true)

Do Until: equals(variables('vHasMore'), false)
  
  Get items:
    Filter: Your conditions
    Top Count: @{variables('vBatchSize')}
    Skip Count: @{variables('vSkipCount')}
    Order By: ID asc
  
  Condition: length(body('Get_items')?['value']) < vBatchSize
    If Yes: Set vHasMore = false
    If No: Increment vSkipCount by vBatchSize
  
  Process items in Apply to each
```

### Pattern 2: ID-Based Pagination (Most Reliable)

```yaml
Variables:
- vLastId: Integer (0)
- vBatchSize: Integer (100)
- vHasMore: Boolean (true)

Do Until: equals(variables('vHasMore'), false)
  
  Get items:
    Filter: ID gt @{variables('vLastId')} and [other conditions]
    Top Count: @{variables('vBatchSize')}
    Order By: ID asc
  
  Condition: length(body('Get_items')?['value']) > 0
    If Yes: 
      Set vLastId = last(body('Get_items')?['value'])?['ID']
    If No: 
      Set vHasMore = false
  
  Process items
```

### Pattern 3: Date-Based Chunking

```yaml
Variables:
- vStartDate: DateTime
- vEndDate: DateTime
- vChunkDays: Integer (7)

While vStartDate < Today:
  
  Get items:
    Filter: Created ge '@{vStartDate}' and Created lt '@{vEndDate}'
    Top Count: 5000
  
  Process items
  
  Set vStartDate = vEndDate
  Set vEndDate = addDays(vStartDate, vChunkDays)
```

## Batch Operations

### Batch Updates Using HTTP Request

Instead of updating items one by one, use batch requests:

```json
{
  "requests": [
    {
      "id": "1",
      "method": "PATCH",
      "url": "lists/getbytitle('On-Hand Material')/items(1)",
      "body": {
        "OnHandQty": 100
      },
      "headers": {
        "IF-MATCH": "*"
      }
    },
    {
      "id": "2",
      "method": "PATCH",
      "url": "lists/getbytitle('On-Hand Material')/items(2)",
      "body": {
        "OnHandQty": 200
      },
      "headers": {
        "IF-MATCH": "*"
      }
    }
  ]
}
```

**Implementation:**

1. Build array of updates
2. Create batch request JSON
3. Send single HTTP request
4. Process batch response

**Benefits:**

- 100x faster than individual updates
- Single API call for multiple operations
- Atomic transaction support

## Concurrency Control

### Configure Optimal Concurrency

```

Apply to each:
  Settings:
    Concurrency: On
    Degree of Parallelism: [1-50]

```

#### Recommended Settings by Scenario

| Scenario | Parallelism | Reason |
|----------|------------|---------|
| Inventory Updates | 1 | Prevent race conditions |
| Read Operations | 20 | Maximize throughput |
| Email Sending | 5 | Avoid throttling |
| API Calls | 10 | Balance speed/limits |
| Heavy Computation | 5 | Prevent timeouts |

### Prevent Race Conditions

For critical sections:

```

Concurrency: 1 (Sequential processing)
OR
Implement optimistic locking with version checks

```

## Throttling Management

### SharePoint Throttling Limits

- **SharePoint Online connector:** Varies by license and tenant
- **HTTP 429 Response:** Indicates throttling - implement exponential backoff
- **Note:** For current limits and quotas, see [Microsoft Learn - Power Automate Limits](https://learn.microsoft.com/en-us/power-automate/limits-and-config)

### Throttling Prevention Strategies

#### 1. Add Strategic Delays

```

After each operation:
  Delay:
    Count: 100
    Unit: Millisecond

After batch of 10:
  Delay:
    Count: 1
    Unit: Second

```

#### 2. Implement Exponential Backoff

```

Retry Policy:
  Type: Exponential
  Count: 5
  Interval: PT10S
  Maximum: PT5M

```

#### 3. Distribute Load

```

Schedule flows at different times:

- Flow A: Runs at :00
- Flow B: Runs at :15
- Flow C: Runs at :30
- Flow D: Runs at :45

```

## Timeout Prevention

### Flow Timeout Limits

- **Standard:** 30 days maximum flow run duration
- **Default action timeout:** 5 minutes (can be extended in action settings)
- **HTTP actions:** Can be configured up to 120 seconds

### Strategies for Long-Running Operations

#### 1. Child Flow Pattern

```

Parent Flow:
  Get all items to process
  For each batch of 100:
    Call child flow with batch

Child Flow:
  Process 100 items
  Return status

```

#### 2. Continuation Token Pattern

```

Variables:

- vContinuationToken: String
- vIsComplete: Boolean (false)

Do Until vIsComplete:
  Process batch
  Save continuation token
  Check time elapsed
  If > 4 minutes:
    Save state
    Trigger new instance with token
    Terminate current

```

#### 3. Queue-Based Processing

```

Flow 1: Queue Builder
  Get items to process
  Add to SharePoint queue list
  
Flow 2: Queue Processor (runs every 5 min)
  Get top 50 from queue
  Process items
  Mark as complete
  
Flow 3: Queue Monitor
  Check for stuck items
  Retry failed items

```

## Query Optimization

### Use Select Query

Reduce data transfer by selecting only needed fields:

```

Get items:
  Select Query: ID,PartNumber,Qty,PostStatus
  // Don't retrieve all columns

```

### Use Filter Query Effectively

```

✅ Optimal:
PartNumber eq 'ABC' and PostStatus eq 'Validated' and Created gt '2024-01-01'

❌ Suboptimal:
PostStatus eq 'Validated' // Then filter in Flow

```

### Use Order By Strategically

```

Order By: Created desc
// Gets newest first, useful for recent data processing

```

## Caching Strategies

### 1. Variable Caching

```

Variables:

- vPartsCache: Object
- vPOCache: Object

// Cache frequently used lookups
If not contains(vPartsCache, PartNumber):
  Get from SharePoint
  Add to vPartsCache

```

### 2. Daily Cache List

```

Flow: Daily Cache Builder (runs at midnight)
  Get all Parts
  Get all active POs
  Store in "Daily Cache" list
  
Main Flows:
  Read from cache list (faster)
  Fallback to source if not found

```

### 3. Compose Action Caching

```

Compose: Build lookup table
{
  "PART001": { "Description": "Widget", "UOM": "EA" },
  "PART002": { "Description": "Gadget", "UOM": "BX" }
}

// Use throughout flow without repeated queries

```

## Performance Monitoring

### Add Performance Metrics

```

Variables:

- vStartTime: utcNow()
- vRecordCount: 0

At End of Flow:
Compose: Performance Metrics
{
  "DurationSeconds": @{div(sub(ticks(utcNow()), ticks(variables('vStartTime'))), 10000000)},
  "RecordsProcessed": variables('vRecordCount'),
  "RecordsPerSecond": @{div(variables('vRecordCount'), max(1, div(sub(ticks(utcNow()), ticks(variables('vStartTime'))), 10000000)))},
  "FlowRun": workflow()?['run']?['name']
}

Log to Performance Tracking list

```

### Key Metrics to Track

- Execution time per record
- Queue depth
- Error rate
- Retry count
- Throttling occurrences

## Optimization Checklist

### Before Deployment

- [ ] All SharePoint columns indexed
- [ ] Pagination implemented for >100 items
- [ ] Retry policies configured
- [ ] Concurrency settings optimized
- [ ] Select queries limit fields
- [ ] Filter queries use indexed columns
- [ ] Batch operations for bulk updates
- [ ] Throttle protection delays added

### During Testing

- [ ] Test with 10x expected volume
- [ ] Monitor execution times
- [ ] Check for 429 errors
- [ ] Verify no timeouts occur
- [ ] Test concurrent executions
- [ ] Measure throughput
- [ ] Check memory usage

### After Deployment

- [ ] Monitor performance dashboard
- [ ] Review error logs weekly
- [ ] Optimize slow queries
- [ ] Adjust concurrency as needed
- [ ] Update caching strategy
- [ ] Document bottlenecks

## Common Performance Issues and Solutions

### Issue 1: Flow Times Out

**Solution:**

- Implement pagination
- Use child flows
- Add continuation token
- Reduce batch size

### Issue 2: SharePoint Throttling

**Solution:**

- Add delays between operations
- Reduce concurrency
- Use service account
- Implement exponential backoff

### Issue 3: Slow Queries

**Solution:**

- Add indexes
- Use Select Query
- Filter on indexed columns first
- Reduce result set size

### Issue 4: Memory Errors

**Solution:**

- Process in smaller batches
- Clear variables after use
- Avoid storing large arrays
- Use streaming where possible

### Issue 5: Concurrent Update Conflicts

**Solution:**

- Set concurrency to 1
- Implement optimistic locking
- Use queue-based processing
- Add retry logic

## Advanced Techniques

### 1. Parallel Branch Processing

```

Parallel Branch 1: Process Part A items
Parallel Branch 2: Process Part B items
Parallel Branch 3: Process Part C items

Join: Wait for all branches
Consolidate results

```

### 2. Lazy Loading Pattern

```

Get minimal data first
Only fetch details when needed
Cache fetched details

```

### 3. Pre-aggregation

```

Nightly Job:
  Calculate daily totals
  Store in summary table
  
Daily Flows:
  Read from summary (fast)
  Only calculate deltas

```

### 4. Smart Scheduling

```

Heavy flows: Run during off-hours
Light flows: Run during business hours
Distribute start times to avoid peaks
Use trigger conditions to skip when not needed

```

## Recommended Architecture for Scale

### For <1000 transactions/day

- Simple flows with basic error handling
- Direct SharePoint operations
- Standard retry policies

### For 1000-10,000 transactions/day

- Implement pagination
- Add caching layer
- Use batch operations
- Monitor performance

### For >10,000 transactions/day

- Queue-based architecture
- Multiple processor flows
- Dedicated service accounts
- Consider Azure Functions for heavy processing
- Implement data warehouse for reporting

## Testing Performance

### Load Testing Script

```javascript
// Generate test data
for(i = 0; i < 1000; i++) {
  Create Tech Transaction:
    PartNumber: "TEST-" + i
    Qty: Random(1, 100)
    TransactionType: Random("Issue", "Receive")
}

// Measure processing time
Start Timer
Wait for all Posted
End Timer
Calculate: Records/Second
```

### Stress Testing Scenarios

1. Burst load: 1000 records in 1 minute
2. Sustained load: 100 records/minute for 1 hour
3. Peak load: 5000 records in 10 minutes
4. Concurrent users: 10 flows running simultaneously

## Conclusion

Performance optimization is crucial for production Power Automate flows. Implement these patterns based on your volume requirements and continuously monitor performance metrics to identify optimization opportunities.
