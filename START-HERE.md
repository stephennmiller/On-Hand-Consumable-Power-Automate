# START HERE - On-Hand Consumable System Implementation Guide

## What You're Building
A complete inventory tracking system in SharePoint + Power Automate that tracks consumable materials with receive/issue transactions.

## Required Order of Implementation

### Prerequisites (Do This First!)
- [ ] Access to SharePoint site
- [ ] Power Automate license
- [ ] Create these SharePoint lists:
  - **Tech Transactions** (for incoming transactions)
  - **On-Hand Material** (for current inventory)
  - **Flow Error Log** (for error tracking)

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
1. Create a RECEIVE for Part "ABC123", Batch "LOT001", Qty 100
2. Verify On-Hand Material shows 100 units
3. Create an ISSUE for same part/batch, Qty 30
4. Verify On-Hand Material shows 70 units
5. Check Flow Error Log is empty

## Common Issues & Fixes

| Problem | Solution |
|---------|----------|
| "Flow not triggering" | Check trigger conditions match exactly |
| "PostStatus not updating" | Ensure column names match exactly |
| "Duplicate inventory rows" | Check filter expressions in Get items |
| "Negative inventory allowed" | Add validation in Issue flow Step 12 |

## File Organization

```
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