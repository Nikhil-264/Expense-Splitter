# Troubleshooting & Fixes

A running log of problems found in [expense-manager.cpp](expense-manager.cpp), why they mattered, and exactly what was changed to fix them. Each entry corresponds to a real commit — see the linked commit hash for the exact diff.

For the full audit this log was drawn from, see [CODE_AUDIT.md](CODE_AUDIT.md) and [SYSTEM_EXPLANATION.md](SYSTEM_EXPLANATION.md).

---

## 1. Group debts were invisible in a user's balance summary

**Status:** Fixed — commit `d50c649`

### The problem

The system tracked money owed in **two separate, unsynchronized places**:

- `User::balances` ([expense-manager.cpp:114](expense-manager.cpp)) — updated only by individual (non-group) expenses and settlements, via `Splitwise::addIndividualExpense` / `Splitwise::settleIndividualPayment`.
- `Group::groupBalances` ([expense-manager.cpp:294](expense-manager.cpp)) — updated only by group expenses and settlements, via `Group::addExpense` / `Group::settlePayment`.

`Splitwise::showUserBalance` computed "Total you owe" / "Total others owe you" purely from `User::getTotalOwed()` / `User::getTotalOwing()`, both of which only read `User::balances`. Group debts never flowed into that map at all.

**Why it mattered:** for a shared-expense tracker, a user's balance is the entire point of the app. In the demo scenario, Rohit owed Manish Rs 200 from a group "Dinner" expense, but calling `showUserBalance` for Rohit reported that debt as nonexistent — it only surfaced his unrelated individual "Coffee" expense. Any feature built on top of `showUserBalance` (or `User::getTotalOwed`/`getTotalOwing`) would present an incomplete, misleading financial picture.

### The fix

`Splitwise::showUserBalance` (expense-manager.cpp) now builds a **combined balance map** per call: it starts from the user's individual `balances`, then iterates every group the user belongs to (`group->isMember(userId)`) and folds in that group's per-counterparty balances via the previously-unused `Group::getUserGroupBalances`. Totals ("Total you owe" / "Total others owe you") and the detailed per-person breakdown are now computed from this combined map instead of `User::balances` alone.

```cpp
map<string, double> combinedBalances = user->balances;
for (auto& groupPair : groups) {
    Group* group = groupPair.second;
    if (group->isMember(userId)) {
        for (auto& groupBalance : group->getUserGroupBalances(userId)) {
            combinedBalances[groupBalance.first] += groupBalance.second;
        }
    }
}
```

**Verification:** rebuilt with `g++ -std=c++17 expense-manager.cpp -o expense-manager.exe` and ran the existing `main()` demo scenario. Before the fix, Rohit's balance summary showed only his Rs 20 individual debt. After the fix, it correctly shows both the Rs 200 group debt to Manish *and* the Rs 20 individual debt from Saurav in one combined view.

**Known limitation left in place:** this fix resolves the *read* path (`showUserBalance`). The two underlying stores (`User::balances` and `Group::groupBalances`) are still separate data structures internally — a proper long-term fix (tracked in [CODE_AUDIT.md](CODE_AUDIT.md), Issue A-1) is to introduce one unified `Ledger` class that both flows write through directly, rather than aggregating at read time.

---

## 2. `addIndividualExpense` credited/debited the full amount instead of the actual split share

**Status:** Fixed — commit `d50c649` (same commit as #1, found while fixing it)

### The problem

`Splitwise::addIndividualExpense` correctly computed a `vector<Split>` via the chosen `SplitStrategy` (e.g. `EqualSplit` divides the amount between the payer and the other user), but then ignored that computed split entirely:

```cpp
// Before
paidByUser->updateBalance(toUserId, amount);      // full amount, not the split share
toUser->updateBalance(paidByUserId, -amount);
```

**Why it mattered:** for an equal-split individual expense (e.g. a Rs 40 coffee split between two people), the correct debt is Rs 20 — the *other person's share* — not the full Rs 40. Using the raw `amount` overstated every individual debt by 2x for an equal split, and would be wrong in a different, non-obvious way for exact/percentage splits too, since it bypassed the strategy's output altogether.

### The fix

The method now looks up the actual computed split amount belonging to `toUserId` and uses that instead of the raw `amount` parameter:

```cpp
// After
double owedAmount = 0;
for (const Split& split : splits) {
    if (split.userId == toUserId) {
        owedAmount = split.amount;
        break;
    }
}

paidByUser->updateBalance(toUserId, owedAmount);
toUser->updateBalance(paidByUserId, -owedAmount);
```

**Verification:** same test run as above — the "Coffee" expense (Rs 40, equal split, paid by Rohit for Saurav) now correctly shows Saurav owing Rohit Rs 20, not Rs 40.

---

## Notes on scope

This log only includes changes that were actually implemented and committed. The full audit ([CODE_AUDIT.md](CODE_AUDIT.md)) documents several additional known issues (missing input validation in `ExactSplit`/`PercentageSplit`, memory leaks from unmanaged `new` allocations, division-by-zero risk in `EqualSplit`, inconsistent error handling, thread-safety gaps in the `Splitwise` singleton, etc.) that have **not** been fixed yet. As those get addressed, add a new numbered entry here following the same "problem → why it mattered → fix → verification" format.
