# Chapter 03 — Prompt Test Suite
**Concept: Custom Tools (SQLite + CSV) + Penalty Calculation Logic**

Run any prompt with:
```bash
uv run python main.py "YOUR PROMPT HERE"
```

---

## Category 1: Single Tool Invocation
*Purpose: Verify agent can call individual tools and interpret results.*

```
What invoices do we have from Gujarat Steel Corp?
```
**Expected:**
- Agent calls `query_ledger` to fetch invoices for VEN-1000
- Returns: Invoice ID, Amount, Status
- Example: INV-2000, Amount: 500000.0, Status: Paid

```
When did Gujarat Steel Corp deliver goods last?
```
**Expected:**
- Agent calls `check_delivery_log` with vendor ID
- Returns delivery records with Expected_Delivery, Actual_Delivery, days_late
- Example: Expected 2025-12-15, Actual 2025-12-29, days_late: 14

**Analysis question:** What data structure does each tool return? (JSON, dict, raw text?) How does the agent parse and interpret it?

---

## Category 2: Multi-Tool Orchestration
*Purpose: Verify agent chains tools together to answer complex questions.*

```
Audit the account for Gujarat Steel Corp and report any discrepancies.
```
**Expected sequence:**
1. Plan: "Check contract, fetch invoices, check deliveries, calculate penalties"
2. Read contract: finds "5% penalty if delivery > 7 days"
3. Query ledger: finds INV-2000 for 500000.0
4. Check delivery: finds 14 days late
5. Calculate penalty: 500000.0 × 0.05 = 25000.0
6. Report: "Discrepancy found: INR 25,000 penalty applies"

```
Compare two vendors: which one has more late deliveries?
```
**Expected:**
- Agent calls `check_delivery_log` for both vendors
- Counts/compares days_late across shipments
- Reports which vendor has worse delivery performance

**Analysis question:** In what order does the agent call the tools? Why is that order logical?

---

## Category 3: Penalty Calculation Verification
*Purpose: Test the core business logic: penalty = invoice_amount × 0.05 if days_late > 7.*

```
What penalty should we charge Gujarat Steel Corp?
```
**Expected:**
- Agent identifies: 14 days late (> 7 days threshold)
- Fetches invoice amount: 500000.0
- Contract says: "5% penalty"
- Calculation: 500000.0 × 0.05 = 25000.0
- Output: "Penalty: INR 25,000"

```
Is there a penalty for vendor VEN-1002?
```
**Expected:**
- Agent checks delivery days_late
- If ≤ 7 days: "No penalty (within acceptable range)"
- If > 7 days: calculates penalty amount
- References contract clause explaining the rule

```
Calculate penalties for all vendors with late deliveries.
```
**Expected:**
- Agent iterates through delivery log
- For each vendor with days_late > 7, fetches invoice and contract
- Calculates: amount × 0.05
- Summarizes: list of penalties by vendor

**Analysis question:** What would break if the rule were "2 days late" instead of "7 days late"? Find where that number lives in the code.

---

## Category 4: Edge Cases & Robustness
*Purpose: Test handling of unusual scenarios.*

```
What if a vendor delivered on time but their invoice amount is zero?
```
**Expected:**
- Agent checks: days_late ≤ 7? Yes, on time.
- Penalty: INR 0 (no late delivery)
- Reports: "No penalty — delivery met terms"

```
We have an invoice for INR 1,000,000 from a vendor that delivered 10 days late. What is the penalty?
```
**Expected:**
- Days late: 10 > 7, triggers penalty
- Amount: 1,000,000 × 0.05 = 50,000
- Output: "Penalty: INR 50,000"

```
What if a vendor's contract doesn't mention a penalty percentage?
```
**Expected:**
- Agent reads contract
- If no "5% penalty" clause found: reports "No penalty clause in contract"
- Handles gracefully (does not assume or calculate penalty)

**Observation:** Does the agent require the contract to explicitly say "5% penalty", or does it infer it?

---

## Category 5: Data Cross-Reference Validation
*Purpose: Verify agent correctly correlates vendor ID across ledger and CSV.*

```
For invoice INV-2000, check if the delivery was on time.
```
**Expected:**
1. Query ledger to find which vendor issued INV-2000 → VEN-1000
2. Look up VEN-1000 in delivery log
3. Compare Expected vs Actual delivery dates
4. Report: "14 days late" or "On time"

```
Which invoices have corresponding late deliveries?
```
**Expected:**
- Agent queries all invoices
- For each, finds corresponding vendor in delivery log
- Cross-references expected vs actual dates
- Lists invoices where delivery was late

**Analysis question:** How does the agent know to correlate Vendor_ID in the ledger with Vendor_ID in the CSV? Is this documented in the system prompt?

---

## Category 6: Discrepancy Detection
*Purpose: Test agent's ability to identify audit findings.*

```
Report all discrepancies for the last 30 days.
```
**Expected:**
- Queries invoices from last 30 days
- Checks deliveries for those vendors
- Calculates penalties
- Reports: "Vendor X: INR Y penalty applies"

```
Is there any invoice we should not have paid?
```
**Expected:**
- Checks: are there late deliveries that should trigger penalties?
- Compares against what was actually paid
- Flags: "Invoice INV-2000 should have a INR 25,000 deduction"

**Analysis question:** What is the difference between "discrepancy detected" and "audit complete"? When does the agent know it has enough data to conclude?

---

## Category 7: Comparison with Chapter 02
*Purpose: Understand what changed between chapters.*

```
Audit Gujarat Steel Corp.
```

Run this prompt in **both** Chapter 02 and Chapter 03. Compare:
- How many steps does the agent take?
- Does it calculate a penalty amount in Ch02? In Ch03?
- Where is the penalty math happening? (prompt-based in Ch02 vs backend logic in Ch03?)

**Analysis question:** Which chapter's approach is better for an auditor? Why?

---

## Self-Check (no LLM needed)
```bash
uv run python main.py --self-check
```
**Expected output:** JSON containing:
```json
{
  "vendor_id": "VEN-1000",
  "vendor_name": "Gujarat Steel Corp",
  "invoice_amount": 500000.0,
  "days_late": 14,
  "penalty_amount_inr": 25000.0,
  "action_required": "Recover Funds"
}
```

**Verify:** penalty = invoice_amount × 0.05 = 500000.0 × 0.05 = 25000.0 ✓
