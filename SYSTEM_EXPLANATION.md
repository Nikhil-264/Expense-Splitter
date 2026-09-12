# System Deep Dive & Explanation

## 1. System Purpose

This project models the core low-level design (LLD) of a shared-expense tracker like Splitwise. The real-world problem it solves: a group of people (roommates, a trip group, friends) incurs shared costs, and each expense needs to be divided among some subset of the group according to a rule (split equally, by exact amounts, or by percentage). Over time, the group accumulates a web of "who owes whom" relationships, and the system needs to (a) record new expenses and update that web correctly, (b) let a user or a group see the current state of balances, (c) let people settle debts, and (d) minimize the number of actual payments needed to zero everyone out (debt simplification), which is the same "minimum cash flow" problem used in real Splitwise-style products.

The entire implementation lives in one file, [expense-manager.cpp](expense-manager.cpp), and is meant to be read as a worked LLD exercise: notice how each design pattern maps directly onto a requirement (pluggable split rules → Strategy; broadcasting balance changes → Observer; one shared registry of users/groups → Singleton; a simple external API → Facade).

## 2. High-Level Architecture

Think of this the way you'd describe it in a system design interview, but scoped to a single process with no persistence or network layer:

```
                          +---------------------+
                          |  main() (driver)    |
                          +----------+-----------+
                                     |
                                     v
                     +-------------------------------+
                     |   Splitwise (Singleton/Facade) |
                     |  - users: map<id, User*>       |
                     |  - groups: map<id, Group*>     |
                     |  - expenses: map<id, Expense*> |
                     +---------------+-----------------+
                                     |
              +----------------------+----------------------+
              v                                              v
      +---------------+                              +----------------+
      |     Group     |<-----owns members------------|      User      |
      | groupBalances |                              |   balances     |
      | groupExpenses |                              | (Observer impl)|
      +-------+-------+                              +----------------+
              |
              | uses
              v
      +----------------+        +--------------------+
      | SplitFactory   |------->|  SplitStrategy     |
      | (Factory)      |        |  Equal/Exact/%     |
      +----------------+        +--------------------+
              |
              v
      +----------------+
      | DebtSimplifier |  (greedy min-cash-flow algorithm)
      +----------------+
```

**Components and roles:**
- **`Splitwise`** (expense-manager.cpp:507-678): the single entry point for all operations — a Singleton (one instance per process) acting as a Facade (hides `User`/`Group` internals behind simple method calls like `createUser`, `addExpenseToGroup`, `showUserBalance`).
- **`Group`** (expense-manager.cpp:275-503): the unit that actually owns shared-expense bookkeeping — membership, group-scoped expenses, and the group's pairwise balance matrix.
- **`User`** (expense-manager.cpp:108-154): a person, who also happens to implement `Observer` so groups can push notifications to them, and who separately tracks non-group ("individual") balances.
- **`Expense`** (expense-manager.cpp:158-178): an immutable record of one spend event and how it was split.
- **`SplitStrategy` family + `SplitFactory`** (expense-manager.cpp:39-105): pluggable algorithms for turning "$X among these people" into a list of per-person `Split` amounts.
- **`DebtSimplifier`** (expense-manager.cpp:180-272): a stateless utility that takes a group's raw balance matrix and returns a minimized version using a greedy netting algorithm.

## 3. End-to-End Flow

Walking through the flow the demo in `main()` (expense-manager.cpp:682-739) actually exercises, for "add a group expense":

1. **Entry point:** `main()` calls `manager->addExpenseToGroup(hostelGroup->groupId, "Lunch", 800.0, user1->userId, groupMembers, SplitType::EQUAL)` (expense-manager.cpp:701).
2. **Facade delegation:** `Splitwise::addExpenseToGroup` (expense-manager.cpp:583-594) looks up the `Group*` by ID via `getGroup()` (expense-manager.cpp:545-548); if not found, it prints an error and returns early — otherwise it delegates to `group->addExpense(...)`.
3. **Validation:** `Group::addExpense` (expense-manager.cpp:390-443) checks that the payer (`paidByUserId`) is a group member via `isMember()` (expense-manager.cpp:347-349), and that every entry in `involvedUsers` is also a member — throwing `runtime_error` if not (expense-manager.cpp:394-403).
4. **Split calculation (Strategy + Factory):** `SplitFactory::getSplitStrategy(SplitType::EQUAL)` (expense-manager.cpp:93-104) returns a new `EqualSplit` instance; its `calculateSplit(amount, involvedUsers)` (expense-manager.cpp:47-56) divides `800.0` evenly across the 4 members and returns a `vector<Split>`, one `{userId, amount}` pair per person.
5. **Record creation:** A new `Expense` object is constructed (expense-manager.cpp:410) with a unique auto-incrementing ID (`Expense::nextExpenseId`, expense-manager.cpp:160/178) and stored in `groupExpenses[expense->expenseId]` (expense-manager.cpp:411) — note: **not** in `Splitwise::expenses`, which is only used by the individual-expense path (see Section 6).
6. **Ledger update:** For every split entry where the split recipient isn't the payer, `updateGroupBalance(paidByUserId, split.userId, split.amount)` (expense-manager.cpp:414-419) is called, which mutates `groupBalances[fromId][toId]` and its mirror `groupBalances[toId][fromId]` (expense-manager.cpp:352-363), auto-removing entries that net to ~0.
7. **Notification (Observer):** `notifyMembers(...)` (expense-manager.cpp:341-345, called at expense-manager.cpp:423) iterates every `User*` in `members` as an `Observer*` and calls `update(message)`, which `User::update` (expense-manager.cpp:122-124) implements by printing `"[NOTIFICATION to <name>]: <message>"` to `cout`.
8. **Output:** The method itself also prints a human-readable summary of the expense and its split (expense-manager.cpp:426-440) before returning `true`.
9. **Read path:** Later, `manager->showGroupBalances(hostelGroup->groupId)` (expense-manager.cpp:709) delegates to `Group::showGroupBalances()` (expense-manager.cpp:468-495), which walks `groupBalances` and prints, per member, who owes them and whom they owe, using `getUserByuserId` (expense-manager.cpp:277-286) to resolve display names.

## 4. Module-by-Module Breakdown

### `Split` / `SplitType` / `SplitStrategy` hierarchy (expense-manager.cpp:15-105)
- **What it does:** Represents "how much does person X owe for this expense" (`Split`, expense-manager.cpp:21-30) and the pluggable algorithms that compute that (`SplitStrategy` and its three subclasses).
- **Why it exists:** So that `Group::addExpense` doesn't need an `if/else` on split type — it just asks the factory for the right strategy object (classic Strategy + Factory pairing).
- **How it works internally:**
  - `EqualSplit` (expense-manager.cpp:45-57): `totalAmount / userIds.size()`, same amount to everyone.
  - `ExactSplit` (expense-manager.cpp:59-72): takes caller-supplied `values[i]` verbatim as each person's share — no normalization to `totalAmount`.
  - `PercentageSplit` (expense-manager.cpp:74-88): `totalAmount * values[i] / 100.0` per person.
  - `SplitFactory::getSplitStrategy(SplitType)` (expense-manager.cpp:91-105): a `switch` that `new`s the matching subclass, defaulting to `EqualSplit` for unrecognized types.
- **Key files/functions:** all in expense-manager.cpp:15-105.

### `Observer` / `User` (expense-manager.cpp:32-154)
- **What it does:** `Observer` is a one-method interface (`update(message)`) that lets any "listener" react to a broadcast. `User` is both a domain entity (name, email, per-counterparty balances) and a concrete `Observer`.
- **Why it exists:** So `Group` can broadcast events (new expense, settlement) to all members uniformly, without knowing they're specifically `User` objects — this is the textbook Observer pattern (subject = `Group`, observers = `User`s).
- **How it works internally:** `User::update` (expense-manager.cpp:122-124) just prints to `cout`; `User::updateBalance(otherUserId, amount)` (expense-manager.cpp:126-133) adjusts a running per-counterparty balance and prunes near-zero entries; `getTotalOwed`/`getTotalOwing` (expense-manager.cpp:135-153) sum negative/positive entries respectively — but **only entries from individual (non-group) transactions**, since group flows update `Group.groupBalances` instead (see Section 7, "Design Decisions").
- **Key files/functions:** expense-manager.cpp:32-154.

### `Expense` (expense-manager.cpp:157-178)
- **What it does:** An immutable value object recording one spend event: description, total amount, who paid, the computed `Split`s, and an optional `groupId` (empty string for individual expenses).
- **Why it exists:** Acts as the historical record / audit trail — separate from the "current balance" state, which is derived and mutable.
- **How it works internally:** Just a constructor that stamps a unique auto-incrementing `expenseId` (via the static counter `nextExpenseId`, expense-manager.cpp:160 and 178) and copies its arguments into fields. No behavior beyond construction.

### `DebtSimplifier` (expense-manager.cpp:180-272)
- **What it does:** Implements the classic "minimum cash flow" / debt-netting algorithm: given a full pairwise balance matrix for a group, produce an equivalent matrix with fewer total non-zero entries (i.e., fewer required payments to settle everyone up).
- **Why it exists:** Without it, if A owes B, B owes C, and C owes A different amounts, settling requires up to 3 payments; simplification can often reduce this to 1-2 payments by netting everyone's overall position first.
- **How it works internally (step by step, expense-manager.cpp:182-271):**
  1. Compute each person's **net position** by summing all their outbound credits and inbound debts (expense-manager.cpp:186-209) — note the comment at expense-manager.cpp:194-196 explaining that only *positive* entries are processed to avoid double-counting the mirrored matrix.
  2. Split people into `creditors` (net > +0.01, i.e., owed money overall) and `debtors` (net < -0.01, i.e., owe money overall) (expense-manager.cpp:211-221).
  3. Sort both lists descending by amount (expense-manager.cpp:223-231) — a greedy heuristic that tends to minimize the number of resulting transactions (largest-first matching), though it is not a formally optimal minimum-transaction solver for all cases.
  4. Use a two-pointer greedy sweep (expense-manager.cpp:242-268): repeatedly match the largest remaining creditor with the largest remaining debtor, settle the smaller of the two amounts, record that single new debt, and advance whichever side got fully settled.
  5. Return the new, smaller balance matrix, which `Group::simplifyGroupDebts` (expense-manager.cpp:497-502) then assigns back onto `groupBalances`, **replacing** the original detailed history.
- **Key files/functions:** `DebtSimplifier::simplifyDebts` (expense-manager.cpp:182), called from `Group::simplifyGroupDebts` (expense-manager.cpp:497).

### `Group` (expense-manager.cpp:275-503)
- **What it does:** The central bookkeeping unit — owns membership, the group's own expense ledger, and the pairwise balance matrix (`groupBalances`).
- **Why it exists:** Real-world sharing happens in groups (a household, a trip), and balances are naturally scoped there rather than globally.
- **How it works internally:**
  - Membership: `addMember`/`removeMember` (expense-manager.cpp:308-339), where `removeMember` refuses to remove someone with an outstanding balance (`canUserLeaveGroup`, expense-manager.cpp:366-379) — enforcing a real-world invariant ("you can't leave until you're settled up").
  - Balances: `updateGroupBalance` (expense-manager.cpp:352-363) is the single mutation primitive for this class, always writing both directions of the pairwise relationship and pruning near-zero entries.
  - Expenses: `addExpense` (expense-manager.cpp:390-443) is the main write path (see Section 3 walkthrough).
  - Settlement: `settlePayment` (expense-manager.cpp:445-466) is just `updateGroupBalance` plus a notification and console message.
  - Reporting: `showGroupBalances` (expense-manager.cpp:468-495) and `simplifyGroupDebts` (expense-manager.cpp:497-502).
- **Key files/functions:** expense-manager.cpp:275-503.

### `Splitwise` (expense-manager.cpp:507-678)
- **What it does:** The single, process-wide access point ("God object" facade) for everything: user/group creation and lookup, wiring users into groups, delegating expense/settlement calls into the right `Group`, and a second, **parallel** code path for individual (non-group) expenses and settlements that bypasses `Group` entirely.
- **Why it exists:** Gives `main()` (or, in a real app, a CLI/API layer) one simple object to call instead of manipulating `User`/`Group` internals directly — the Facade pattern.
- **How it works internally:** Mostly thin delegation (`addExpenseToGroup` → `group->addExpense`, expense-manager.cpp:583-594), except for `addIndividualExpense`/`settleIndividualPayment` (expense-manager.cpp:610-640), which duplicate the strategy-call + balance-update logic independently against `User.balances` instead of a `Group`'s `groupBalances`.
- **Key files/functions:** expense-manager.cpp:507-678; singleton accessor `getInstance()` at expense-manager.cpp:517-522.

## 5. Data Flow

```
main()
  │
  ▼
Splitwise (facade)  ──lookup──▶  users{}, groups{} (map<string, T*>)
  │
  ▼
Group.addExpense(desc, amount, payer, involvedUsers, splitType, splitValues)
  │
  ├─▶ SplitFactory.getSplitStrategy(splitType)  ─▶ SplitStrategy subclass
  │        │
  │        ▼
  │   calculateSplit(amount, involvedUsers, splitValues) ─▶ vector<Split>
  │
  ├─▶ new Expense(desc, amount, payer, splits, groupId) ─▶ stored in groupExpenses{}
  │
  ├─▶ for each Split: Group.updateGroupBalance(payer, split.userId, split.amount)
  │        │
  │        ▼
  │   groupBalances[payer][splitUser] += amount
  │   groupBalances[splitUser][payer] -= amount   (mirrored, pruned at |x|<0.01)
  │
  └─▶ notifyMembers(message) ─▶ each User.update(message) ─▶ cout
```

**Transformations at each step:** raw request parameters (strings/doubles) → `vector<Split>` (strategy output) → an `Expense` record (persisted history) + mutations to `groupBalances` (current-state cache). The system deliberately keeps history (`Expense` objects) separate from current state (`groupBalances`), though nothing ever reconstructs current state from history — if `groupBalances` were ever corrupted, there's no replay mechanism from `groupExpenses` to rebuild it.

## 6. Core Logic Explanation

**Algorithms used:**
- **Equal/Exact/Percentage splitting** — simple arithmetic, O(n) in number of involved users (expense-manager.cpp:45-88).
- **Debt simplification** — a greedy two-pointer netting algorithm (expense-manager.cpp:242-268) that runs in O(n log n) (dominated by the sort) for n = number of group members with nonzero net balance. This is a well-known heuristic for the "minimum cash flow" problem; it is *not* guaranteed to find the true minimum number of transactions in all cases (that variant of the problem is NP-hard in general), but the greedy largest-first match is a strong, commonly used approximation.

**Business logic / important decision points:**
- A user **cannot leave a group while they have any nonzero balance** with any other member (`canUserLeaveGroup`, expense-manager.cpp:366-379) — this is the one hard business invariant enforced in the codebase, demonstrated in `main()` at expense-manager.cpp:726-733 (first removal attempt fails, a settlement is made, second attempt succeeds).
- Balances that net to within `0.01` of zero are treated as settled and removed from the map (used consistently for float-precision tolerance in currency math, e.g. expense-manager.cpp:130, 216-218, 357-360, 374).
- Individual (2-person, no group) expenses and settlements are tracked in a data structure completely separate from group expenses — see Section 7 for why this is a meaningful (and risky) design decision.

## 7. Design Decisions

- **Splitting `User.balances` (individual) from `Group.groupBalances` (group-scoped):** likely chosen because it mirrors real Splitwise UX, where "non-group" (friend-to-friend) expenses and "group" expenses are conceptually different features. **Trade-off:** it's simple to implement each path independently, but it means there is no single query that answers "what does this user owe in total, everywhere" (as covered in [CODE_AUDIT.md](CODE_AUDIT.md) Issue A-1) — `showUserBalance` (expense-manager.cpp:643-662) only reflects individual balances. **Better alternative:** a single `Ledger` keyed by `(fromUserId, toUserId, context)` where `context` can be `"individual"` or a `groupId`, letting one query aggregate across all contexts for a given user — this is also what the README's mention of a "Ledger" singleton implies was intended.
- **Strategy + Factory for splits:** a textbook, appropriate choice — it isolates the "how do we divide this amount" concern so a new split type (e.g. "by shares/weights") can be added by writing one new subclass and one new `SplitType` enum value, without touching `Group::addExpense`. **Trade-off:** the factory returns raw owning pointers with no cleanup (leak, see Audit Issue A-2); a value-returning factory (`unique_ptr<SplitStrategy>`) would keep the same design benefit without the leak.
- **Observer pattern for notifications:** decouples `Group` from knowing exactly how a "notification" is delivered — today it's `cout`, but the interface would allow swapping in email/push/SMS observers without changing `Group`. **Trade-off:** currently only `User` implements `Observer`, and the notification message is always a hardcoded string built inside `Group::addExpense`/`settlePayment` rather than a structured event object, so it's harder to route different messages to different channels differently.
- **Singleton for `Splitwise`:** a natural first modeling choice for "there's one app-wide registry," but it's also a well-known anti-pattern for testability (global mutable state makes isolated unit tests hard, since `Splitwise::getInstance()` persists between test cases). **Better alternative:** construct one `Splitwise`/`ExpenseManager` instance explicitly in `main()` and pass it by reference to whatever needs it (dependency injection) — same "single instance in practice" behavior without the global-state downsides.
- **Debt simplification replaces the group's live balance matrix in place** (`groupBalances = simplifiedBalances`, expense-manager.cpp:499): a deliberate simplification-first design, but it means the "detailed, who-actually-owes-whom-from-which-expense" history is lost from the live balance view after simplification runs — only `groupExpenses` (the immutable `Expense` records) retains the original detail. This is a reasonable trade-off for a settlement feature (you generally want fewer, larger payments) but is worth being explicit about, since it's a destructive operation on `groupBalances`.

## 8. External Integrations

None. This is a fully self-contained, offline console application with:
- No database (SQL or NoSQL)
- No HTTP/REST or RPC API
- No third-party SDKs
- No file I/O (no persistence — all state lives in process memory and is lost on exit)

The only "integration point" is `cout`/console output, which stands in for what would be a real notification channel (email/SMS/push) or a UI layer in a production system.

## 9. Configuration & Environment

There are no environment variables, configuration files, or build-time flags used anywhere in the codebase. Everything (users, groups, expenses) is hardcoded directly into the `main()` demo scenario (expense-manager.cpp:682-739). There is no `.env`, no `config.json`, and the [.gitignore](.gitignore) contains only standard ignore rules (not inspected for project-specific entries beyond what's typical for a C++ project).

## 10. How to Run the System

**Dependencies:** a C++ compiler supporting at least C++11 (the code uses `enum class`, range-based `for`, lambdas — expense-manager.cpp:224-231) and, specifically, a GCC-compatible toolchain, because of `#include <bits/stdc++.h>` (expense-manager.cpp:7), which is not available on MSVC or vanilla Clang without extra configuration.

**Setup steps (no build system exists, so compile directly):**

```bash
g++ -std=c++17 expense-manager.cpp -o expense-manager
```

**Execution:**

```bash
./expense-manager
```

**Execution flow:** running the binary executes `main()` (expense-manager.cpp:682), which runs the entire scripted demo scenario end-to-end — creating 4 users, forming a group, adding two group expenses (one equal split, one exact split), printing balances, simplifying debts, adding an individual expense, showing per-user balances, attempting (and then succeeding at) removing a member, and settling a payment — all output goes to stdout with no user interaction required.

## 11. How to Extend the System

- **Add a new split type** (e.g., "by shares/weights"): add a value to `SplitType` (expense-manager.cpp:15-19), create a new `SplitStrategy` subclass implementing `calculateSplit` (pattern after expense-manager.cpp:45-88), and add a `case` in `SplitFactory::getSplitStrategy` (expense-manager.cpp:93-104). No changes needed to `Group::addExpense` — this is the safe extension point the Strategy pattern was built for.
- **Add a new notification channel** (e.g., email): create a new class implementing `Observer` (expense-manager.cpp:33-36) with its own `update()` method, and register instances of it wherever `Group.members` (or a new, separate observer list) is populated. Currently `Group::notifyMembers` only iterates `members` (which are all `User*`), so a real extension would require adding a separate `vector<Observer*> subscribers` to `Group` for non-`User` observers.
- **Add persistence:** introduce a serialization layer (e.g., JSON via a library, or simple CSV) that can dump/reload `Splitwise::users/groups/expenses` — this is currently entirely absent, so it's a clean, additive extension point that wouldn't require restructuring existing classes, just adding save/load methods to `Splitwise`.
- **Unify balances (recommended before other extensions):** introduce a single `Ledger` class as described in Section 7 and [CODE_AUDIT.md](CODE_AUDIT.md) Issue A-1, and have both `Group::updateGroupBalance` and `Splitwise::addIndividualExpense`/`settleIndividualPayment` write through it. This is the most valuable and least risky place to invest extension effort, since nearly every future feature (multi-group balance views, exports, notifications on total-balance thresholds) depends on having one correct source of truth.
- **Safe extension points, summarized:** `SplitStrategy` subclasses (safe, isolated), `Observer` implementations (safe, isolated), new `Splitwise` facade methods (safe, as long as they reuse `Group`'s existing mutation primitives rather than duplicating them). **Unsafe without refactor first:** anything that needs a cross-group or individual+group combined view of a user's balance, since that data doesn't exist in one place yet.

## 12. Mental Model Summary

Picture two independent notebooks per person: one shared notebook per group they're in (`Group.groupBalances`), and one personal notebook for one-off IOUs with friends outside any group (`User.balances`). Every time money changes hands, the relevant notebook gets a matching pair of entries (one negative, one positive) so the books always balance internally — but the two notebook types are never cross-referenced, so nobody has a single page listing "everything I owe or am owed, everywhere." Expenses are split via pluggable formulas (equal share, my-exact-share, or my-percentage) chosen by a small factory, recorded as permanent historical entries, and then folded into the relevant notebook's running totals. Periodically, a group can ask a specialist (`DebtSimplifier`) to look at everyone's net position in the shared notebook and rewrite it as the smallest possible set of payments that achieves the same net effect — trading historical detail for fewer future transactions.
