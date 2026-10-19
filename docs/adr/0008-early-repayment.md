# ADR-0008: Early repayment discount

- **Status:** Proposed
- **Date:** 2026-10-19

## Context

Interest is fixed when a loan is disbursed. `quote_terms` sets
`total_repayment = principal + principal * rate / BASIS_POINTS` and the
loan's term is always `policy.max_loan_duration`. The rate is a flat charge
for the whole term, not an annualised one: a borrower who repays the day
after disbursement owes exactly as much interest as one who repays on the
due date.

That has two effects:

- **No reason to repay early.** Capital sits with the borrower until the
  due date even when they could return it, so the treasury has less to
  lend to the next member.
- **The rate overstates the cost of short use.** The DAO has priced a
  full term of risk and opportunity cost, but it carries far less of
  either on a loan returned in a week.

The constraints:

- **Members' yield.** Interest is the members' return (it is distributed
  through `distribute_interest`). A discount must not let a loan cost
  almost nothing, or a member could borrow the treasury repeatedly for a
  few ledgers at no cost.
- **Interest-first repayments.** `repay_loan_internal` applies each payment
  to interest before principal, and interest that has been paid has
  already been distributed as yield. It can't be clawed back.
- **Storage compatibility.** `Loan` and `LoanPolicy` are stored structs;
  adding fields would break decoding of existing entries (see ADR-0004).
  `Loan` already carries `start_time`, `due_time`, `principal`,
  `total_repayment` and `amount_repaid`, which is enough.
- **Late penalty.** Past `due_time` a penalty is added to `principal` and
  `total_repayment`. A discount must never apply to an overdue loan.

## Decision

On a **full payoff before `due_time`**, the borrower owes only the interest
**earned so far, pro rata by time, with a floor of half the loan's
interest**. Interest not yet earned is waived.

### Formula

With `interest_total = total_repayment - principal` and
`term = due_time - start_time`, evaluated at `now`:

```text
elapsed         = now - start_time
earned_bps      = max(MIN_EARNED_BPS, ceil(elapsed * BASIS_POINTS / term))   (capped at BASIS_POINTS)
earned_interest = ceil(interest_total * earned_bps / BASIS_POINTS)
interest_paid   = min(amount_repaid, interest_total)
rebate          = interest_total - max(earned_interest, interest_paid)
payoff_amount   = total_repayment - amount_repaid - rebate
```

with `MIN_EARNED_BPS = 5_000`. Both roundings go up, so rounding favours
members, and `rebate` is never negative.

- **The floor.** In the first half of the term a payoff still owes half the
  interest. This is the members' minimum return for having committed the
  capital and voted on the loan, and it removes any gain from cycling short
  loans. It also keeps the yield distributed to members predictable.
- **Paid interest stays paid.** Because repayments are interest-first, a
  borrower who has already paid more interest than `earned_interest` gets
  no refund; the rebate just shrinks to zero. No transfer from the treasury
  is ever made.

Worked example: principal 1,000, rate 10% (interest 100), term 30 days.

| Payoff on day | 1 | 10 | 15 | 20 | 27 | 30 |
| --- | --- | --- | --- | --- | --- | --- |
| Interest owed | 50 | 50 | 50 | 67 | 90 | 100 |
| Payoff amount | 1,050 | 1,050 | 1,050 | 1,067 | 1,090 | 1,100 |

### Where it applies

- **Only a full payoff.** `repay_loan` / `repay_loan_in_asset` (amount
  omitted) and `repay_loan_partial` / `repay_partial` when the amount equals
  the discounted `payoff_amount`. A partial payment that leaves a balance
  gets no discount and is split interest-first as today. When the payoff
  closes the loan, `total_repayment` is lowered by `rebate` before the
  usual `remaining == 0` check, so the stored loan records what was really
  paid and `Repaid` is set as normal.
- **Only before `due_time`.** From `due_time` on, `elapsed >= term` gives
  `earned_bps = BASIS_POINTS` and no rebate. The grace period and the late
  penalty are unchanged.
- **Not for extended loans.** A loan with an approved `LoanExtension`
  (ADR-0004) gets no rebate: the borrower asked for, and paid for, more
  time, and its `due_time` no longer matches the original term. This is
  one storage read on the payoff path.
- **Yield.** Only the interest actually collected is passed to
  `distribute_interest`, through the existing interest-first split. Waived
  interest was never in the treasury, so nothing else changes.
- **Composes with ADR-0005.** The reputation discount lowers the quoted
  rate and so `interest_total`; this discount then waives part of that
  smaller amount. They can both apply to one loan.

### ABI additions

| Item | Purpose |
| --- | --- |
| `get_payoff_amount(loan_id) -> Result<i128, Error>` | Exact amount that closes the loan now, including the rebate or any late penalty. |
| Event `loan_erly`: `(loan_id, borrower, rebate)` | Emitted on payoff when `rebate > 0`; `loan_rpy` is unchanged. |

No storage, `Loan`, `LoanPolicy` or `Member` changes, and no new errors.
`MIN_EARNED_BPS` is a compiled-in constant; making it governable later would
use a separate policy key with this value as the default, as ADR-0005
suggests for its own constants.

## Alternatives considered

- **No discount (status quo).** Simplest, and the flat rate is what
  members voted on. Rejected because it gives borrowers no reason to free
  capital early and makes short-term use of a loan look expensive.
- **Pure pro-rata interest, no floor.** Fairest to the borrower, but a
  loan repaid in the same ledger costs nothing, which invites free
  borrowing of the treasury and leaves voters' work unpaid. Rejected.
- **Fixed prepayment fee instead of a floor** (e.g. pro-rata plus 1% of
  principal). Behaves badly at both ends: too high on small loans, too low
  on large ones. The 50% floor scales with the interest the DAO priced.
- **Rebate paid back after repayment.** Same economics but needs a second
  transfer out of the treasury and interacts badly with interest that has
  already been distributed as yield. Rejected.
- **Discount on partial payments too** (re-amortising the schedule).
  Needs a per-loan accrual record, which means a storage change, and makes
  the interest-first split much harder to follow. Deferred.
- **A governable, per-loan `min_earned_bps` in `LoanPolicy`.** Breaks the
  stored struct. Deferred until policy fields are added through a
  migration or a separate key.

## Consequences

- Borrowers who can repay early save up to half the interest, and capital
  comes back to the treasury sooner for the next loan.
- Members give up the unearned part of the interest on early payoffs, but
  never less than half of what they voted for, and nothing already
  distributed is affected.
- The amount that closes a loan now depends on time. Clients must call
  `get_payoff_amount` rather than computing `total_repayment -
  amount_repaid`. A borrower who sends the old full amount through
  `repay_loan_partial` before `due_time` would overpay and be rejected with
  `InvalidAmount`, so wallets should switch to `repay_loan` or the new view.
- Off-chain indexers summing `total_repayment` will see the lowered value
  for early-repaid loans, which is the amount actually received.
- **Implementation tests:** the worked example above; the floor in the
  first half of the term; no rebate at or after `due_time` and with the
  late penalty; no rebate on a partial payment that leaves a balance; no
  refund when paid interest already exceeds `earned_interest`; no rebate on
  an extended loan; rounding in members' favour; `distribute_interest`
  receives only collected interest; `get_payoff_amount` matches what
  `repay_loan` charges; and `loan_erly` appears only when `rebate > 0`.
