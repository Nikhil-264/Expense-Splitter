# Codebase Audit Report

## 1. Overview

**Purpose.** The system is a single-file C++ console application that models a Splitwise-style shared-expense manager: it creates users, organizes them into groups, records expenses split three ways (equal, exact, percentage), maintains pairwise debt balances, notifies group members of changes, and can "simplify" a group's debt graph into a minimal set of settlements. All logic (models, strategies, singleton facade, and a `main()` demo scenario) lives in [expense-manager.cpp](expense-manager.cpp).

**Tech stack.** C++ (single translation unit), STL containers (`std::map`, `std::vector`), no build system, no package manager, no external dependencies. `#include <bits/stdc++.h>` (expense-manager.cpp:7) is used as a catch-all header — this is a GCC-only extension, not portable ISO C++.

**Architecture summary.** This is a monolithic, in-memory, single-process console app with no persistence, no networking, and no concurrency. It is organized as a set of plain classes in one file rather than as separate compilation units/headers, so there is no real "layering" (no separation between domain model, service layer, and presentation — `cout` calls are interleaved directly inside domain methods, e.g. `Group::addExpense` at expense-manager.cpp:390-443).

## 2. Architecture & Design Review

**High-level architecture:** Single monolith, procedural `main()` driving a small object graph. There is one entry point (`main`, expense-manager.cpp:682) that both sets up the "application" (via the `Splitwise` singleton) and acts as an ad-hoc integration test / demo script.

**Design patterns used (as claimed in README.md vs. as actually implemented):**

| Pattern | README claim | Actual implementation | File/line |
|---|---|---|---|
| Strategy | `SplitStrategy` for Equal/Exact/Percent | ✅ Implemented: `SplitStrategy` interface with `EqualSplit`, `ExactSplit`, `PercentageSplit` | expense-manager.cpp:39-88 |
| Factory | `SplitFactory` builds the right split | ✅ Implemented, but returns raw owning pointers with no lifetime management (`new EqualSplit()` never freed) | expense-manager.cpp:91-105 |
| Observer | "NotificationCenter pushes balance updates to users/channels" | ⚠️ Partially implemented. `Observer` interface exists (expense-manager.cpp:33-36) and `User` implements `update()` (expense-manager.cpp:122-124), and `Group::notifyMembers` iterates `members` as `Observer*` (expense-manager.cpp:341-345). There is **no `NotificationCenter` class** anywhere in the codebase — the README describes a component that does not exist. |
| Singleton | "Ledger (source of truth for balances)" | ⚠️ Implemented, but as `Splitwise` (a Facade), not a class named `Ledger`. There is no dedicated ledger class — balances are scattered across `User::balances` (expense-manager.cpp:114) and `Group::groupBalances` (expense-manager.cpp:294), which are two independent, unsynchronized sources of truth (see Issue A-1 below). |
| Facade | "ExpenseService exposes simple APIs" | ⚠️ Implemented as `Splitwise` (expense-manager.cpp:507-678), not `ExpenseService`. Functionally a facade over `User`/`Group`, but it also duplicates group logic (individual expenses/settlements re-implement balance math instead of delegating to a shared component). |

The README's architecture section is materially inaccurate against the actual code (wrong class names, a nonexistent class, and a duplicated-rather-than-unified ledger). This should be corrected or the code should be refactored to match the documented design.

**Module interactions:**
- `Splitwise` (the facade/singleton, expense-manager.cpp:507) owns `map<string,User*> users`, `map<string,Group*> groups`, and `map<string,Expense*> expenses`, and delegates group-scoped operations to `Group` (e.g. `addExpenseToGroup` → `Group::addExpense`, expense-manager.cpp:583-594).
- `Group` (expense-manager.cpp:275-503) owns its members (`vector<User*>`), its own expense book (`map<string,Expense*> groupExpenses`), and its own balance matrix (`groupBalances`). It uses `SplitFactory` (expense-manager.cpp:406) to compute splits and `DebtSimplifier` (expense-manager.cpp:180) to minimize transactions.
- `User` (expense-manager.cpp:108-154) holds its own **separate** balance map used only for non-group ("individual") expenses (expense-manager.cpp:622-640), creating two parallel, non-interacting accounting systems.
- `Expense` (expense-manager.cpp:158-178) is a passive data holder referenced by both `Group.groupExpenses` and `Splitwise.expenses`, but the two maps are never kept in sync (group expenses are never added to `Splitwise::expenses`).

## 3. Code Quality Analysis

- **Naming conventions:** Mostly consistent camelCase for methods/variables and PascalCase for classes, matching typical C++ style. One inconsistency: `getUserByuserId` (expense-manager.cpp:277) has incorrect internal capitalization (should be `getUserByUserId`).
- **Modularity:** None — everything is in one `.cpp` file with no header separation (no `.h`/`.hpp`), so there are no independent compilation units, no clear public/private API boundary at the file level, and the file cannot be unit-tested without pulling in `main()`.
- **Reusability:** Low. `Splitwise::addIndividualExpense` (expense-manager.cpp:622-640) reimplements balance-update logic that already exists in `Group::updateGroupBalance` (expense-manager.cpp:352-363) instead of reusing it — see Issue B-1.
- **Separation of concerns:** Poor. Domain classes (`Group`, `Splitwise`) mix business logic with presentation (`cout <<` formatting) directly inside methods like `Group::addExpense` (expense-manager.cpp:420-440), `Group::showGroupBalances` (expense-manager.cpp:468-495), and `Splitwise::showUserBalance` (expense-manager.cpp:643-662). There is no separation between "compute" and "display."
- **Code duplication:**
  - Balance-update logic is duplicated between `Group::updateGroupBalance` (expense-manager.cpp:352-363) and `User::updateBalance` (expense-manager.cpp:126-133) / `Splitwise::settleIndividualPayment` (expense-manager.cpp:610-620) — both implement "add amount, remove if near zero" independently.
  - The "look up user in group members" loop is duplicated as `Group::getUserByuserId` (expense-manager.cpp:277-286) and again inline pattern-matched in `Splitwise::getUser` (expense-manager.cpp:532-535), though the latter is map-based; still, two different lookup idioms for conceptually the same operation exist in the file.
  - Null-checking after `getUser`/`getGroup` lookups is repeated ad-hoc in nearly every `Splitwise` method (expense-manager.cpp:550-620) without a shared helper or consistent error-handling strategy (some return silently, some print and return, some throw).

## 4. Detailed Issue Log

### Issue A-1: Two disjoint, unsynchronized balance systems
- **Severity:** Critical
- **File/line:** expense-manager.cpp:114 (`User::balances`) vs. expense-manager.cpp:294 (`Group::groupBalances`)
- **Problem:** A user's total debt exists in two independent places: `User.balances` (populated only by `settleIndividualPayment`/`addIndividualExpense`) and `Group.groupBalances` (populated only by group expense/settlement methods). `User::getTotalOwed()`/`getTotalOwing()` (expense-manager.cpp:135-153) only reflect individual expenses and settlements — group balances never flow into a user's own balance map.
- **Why it's a problem:** `Splitwise::showUserBalance` (expense-manager.cpp:643-662) claims to show "Balance for X" but only shows the individual-expense subset, silently omitting all group debts. Any consumer relying on `User.balances` (or `getTotalOwed`/`getTotalOwing`) gets an incomplete, misleading financial picture — a critical correctness bug for a financial application.
- **Suggested fix:** Introduce a single ledger abstraction (matching the README's stated "Singleton — Ledger" design) that both group and individual expense flows write through, e.g. `Ledger::recordDebt(fromUserId, toUserId, amount)` used by both `Group::updateGroupBalance` and `Splitwise::addIndividualExpense`/`settleIndividualPayment`, and have `showUserBalance` query that ledger across all groups plus individual expenses.

### Issue A-2: Memory leaks — no cleanup of raw `new`-allocated objects
- **Severity:** High
- **File/line:** expense-manager.cpp:96-102 (`SplitFactory::getSplitStrategy`), expense-manager.cpp:526 (`createUser`), expense-manager.cpp:539 (`createGroup`), expense-manager.cpp:410/629 (`Expense`)
- **Problem:** `SplitFactory::getSplitStrategy` returns a `new`-allocated `SplitStrategy*` that is used once in `Group::addExpense` (expense-manager.cpp:406-407) and `Splitwise::addIndividualExpense` (expense-manager.cpp:626-627) and never deleted — leaked on every expense creation. `Splitwise::createUser`/`createGroup` allocate `User`/`Group` with `new` and store raw pointers in `map`s, but `Splitwise` has no destructor to free `users`/`groups`/`expenses` (only `Group`'s destructor frees `groupExpenses`, expense-manager.cpp:301-306, and even that never runs because `Splitwise` is a leaked singleton that's never destroyed).
- **Why it's a problem:** Every expense recorded leaks a `SplitStrategy` object; the whole `Splitwise` singleton and its `User`/`Group`/`Expense` graph is intentionally never freed (`Splitwise::instance` is a bare pointer, expense-manager.cpp:513/680). In a long-running process (as opposed to a short demo `main`), this is unbounded memory growth.
- **Suggested fix:** Use `std::unique_ptr<SplitStrategy>` returned by value from the factory; store `unique_ptr<User>`/`unique_ptr<Group>`/`unique_ptr<Expense>` in the manager maps; add explicit ownership semantics instead of raw owning pointers throughout.

### Issue A-3: `ExactSplit`/`PercentageSplit` have no validation and will crash or silently miscompute
- **Severity:** Critical
- **File/line:** expense-manager.cpp:59-72 (`ExactSplit::calculateSplit`), expense-manager.cpp:74-88 (`PercentageSplit::calculateSplit`)
- **Problem:** Both methods have a `//validations` comment (expense-manager.cpp:65, expense-manager.cpp:80) but no actual validation. `ExactSplit` indexes `values[i]` for `i` up to `userIds.size()` with no check that `values.size() == userIds.size()`, and never verifies that the exact amounts sum to `totalAmount`. `PercentageSplit` never verifies percentages sum to 100.
- **Why it's a problem:** If `values` is shorter than `userIds` (e.g. caller passes a mismatched vector, or omits it and the default empty `{}` is used with `SplitType::EXACT`), `values[i]` is an out-of-bounds read on a `std::vector` — undefined behavior (likely a crash or garbage data) rather than a caught error. Even when sizes match, mismatched sums silently corrupt the ledger (e.g. exact splits totaling more or less than the actual expense amount) with no warning.
- **Suggested fix:** Validate `values.size() == userIds.size()` and throw a clear exception if not; for `ExactSplit`, assert `sum(values) == totalAmount` (within epsilon); for `PercentageSplit`, assert `sum(values) == 100.0` (within epsilon). Fail fast with a descriptive error rather than allowing UB or silent corruption.

### Issue A-4: Division by zero in `EqualSplit`
- **Severity:** High
- **File/line:** expense-manager.cpp:50 (`double amountPerUser = totalAmount / userIds.size();`)
- **Problem:** No check that `userIds` is non-empty before dividing.
- **Why it's a problem:** `Group::addExpense` (expense-manager.cpp:390-443) does not validate that `involvedUsers` is non-empty before calling the split strategy. An empty `involvedUsers` vector produces a division by zero (yields `inf`/`NaN` for `double`, which then propagates silently into `groupBalances`), corrupting the group's ledger without any error surfaced to the caller.
- **Suggested fix:** Validate `!involvedUsers.empty()` in `Group::addExpense` before invoking the strategy, and/or guard in `EqualSplit::calculateSplit` itself and throw `invalid_argument` on empty input.

### Issue B-1: Duplicated balance-update logic between individual and group flows
- **Severity:** Medium
- **File/line:** expense-manager.cpp:352-363 (`Group::updateGroupBalance`) vs. expense-manager.cpp:610-620 (`Splitwise::settleIndividualPayment`) and expense-manager.cpp:635-636 (inline in `addIndividualExpense`)
- **Problem:** The pattern "increment balance, decrement the mirror balance, drop entries under 0.01" is implemented three separate times with subtly different code paths (map-of-map vs. per-user map).
- **Why it's a problem:** A future bug fix (e.g. changing the zero-threshold, or fixing rounding) has to be applied in three places; it's easy to fix one and miss another (as Issue A-1 shows, they're not even using the same data structure).
- **Suggested fix:** Extract a single `Ledger` class (per README's intended design) with one `applyTransfer(fromId, toId, amount)` method used everywhere.

### Issue B-2: `getUserByuserId` returns the *last* matching pointer silently, no not-found signal beyond `nullptr`
- **Severity:** Medium
- **File/line:** expense-manager.cpp:277-286
- **Problem:** The loop doesn't `break`/`return` on match — it keeps iterating and overwrites `user` on each match (harmless here since `userId` should be unique, but wasteful and misleading), and callers dereference the result without null-checks (e.g. expense-manager.cpp:422 `getUserByuserId(paidByUserId)->name`, expense-manager.cpp:456-457, expense-manager.cpp:473, expense-manager.cpp:484).
- **Why it's a problem:** If `paidByUserId` (or any looked-up ID) isn't actually a member for any reason, this throws a null-pointer dereference/segfault instead of a clean error, despite `isMember()` checks existing elsewhere in the same file (inconsistently applied).
- **Suggested fix:** Add `break` after match for efficiency/clarity, and either throw a descriptive exception on `nullptr` before use, or check-and-report at each call site consistently.

### Issue B-3: Inconsistent error-handling strategy (throw vs. print-and-return vs. print-and-continue)
- **Severity:** Medium
- **File/line:** expense-manager.cpp:367-369 (`canUserLeaveGroup` throws), expense-manager.cpp:394-403 (`addExpense` throws), expense-manager.cpp:447-450 (`settlePayment` prints and returns `false`), expense-manager.cpp:563-572 (`removeUserFromGroup` prints and returns `false`)
- **Problem:** Some invalid-input paths throw `runtime_error`, others print to `cout` (not even `cerr`) and return a boolean, with no consistent convention documented or enforced.
- **Why it's a problem:** Callers cannot write uniform error handling; `main()` never wraps anything in try/catch (expense-manager.cpp:682-739), so any of the `throw runtime_error(...)` paths (e.g. adding an expense with a non-member payer) will crash the whole program instead of being handled gracefully — inconsistent with the print-and-return paths that degrade gracefully.
- **Suggested fix:** Pick one strategy (prefer exceptions for programming/contract errors, return values for expected/recoverable conditions) and apply uniformly; wrap `main()`'s scenario in try/catch to prevent crashes from becoming the default failure mode.

### Issue B-4: `Splitwise` singleton is never destroyed; not thread-safe
- **Severity:** Medium
- **File/line:** expense-manager.cpp:513-522, expense-manager.cpp:680
- **Problem:** Classic raw-pointer, lazily-initialized Meyers-style-but-not-actually-Meyers singleton. `getInstance()` is not synchronized.
- **Why it's a problem:** If ever used from multiple threads, `if (instance == nullptr) instance = new Splitwise();` is a data race that can construct two instances or return a partially-constructed pointer. Currently single-threaded so latent, but a common defect once concurrency is introduced.
- **Suggested fix:** Use a function-local `static Splitwise instance;` (Meyers singleton, thread-safe under C++11's static-init guarantees) instead of a heap-allocated raw pointer.

### Issue B-5: Dead/unused code and stray include
- **Severity:** Low
- **File/line:** expense-manager.cpp:7 (`#include <bits/stdc++.h>`), expense-manager.cpp:511 (`map<string, Expense*> expenses` in `Splitwise`), expense-manager.cpp:381-387 (`getUserGroupBalances`, never called anywhere)
- **Problem:** `bits/stdc++.h` is redundant given the explicit includes above it (lines 1-6) and is non-portable (GCC-specific, slows compilation). `Splitwise::expenses` is populated only by `addIndividualExpense` (expense-manager.cpp:630) and never read anywhere. `Group::getUserGroupBalances` (expense-manager.cpp:382-387) is defined but never invoked from `main()` or anywhere else in the file.
- **Why it's a problem:** Increases compile time and gives a false impression of a feature (a queryable per-user, per-group balance API) that's actually dead.
- **Suggested fix:** Remove the umbrella include and keep only the specific headers used; remove `Splitwise::expenses` or wire it into `showUserBalance`/reporting; either call `getUserGroupBalances` from a real feature (e.g. a "show my balance in this group" command) or delete it.

### Issue B-6: Forward declarations for classes never referenced afterward as pointers/incomplete types
- **Severity:** Low
- **File/line:** expense-manager.cpp:10-13
- **Problem:** `class User; class Group; class ExpenseManager;` are forward-declared, but `ExpenseManager` is never defined anywhere (the real facade class is `Splitwise`, expense-manager.cpp:507), and `User`/`Group` are fully defined later in the same file before being used, making the forward declarations unnecessary in a single-TU program.
- **Why it's a problem:** `ExpenseManager` is a vestigial forward declaration for a class that doesn't exist — a leftover from an earlier design/rename that was never cleaned up, and is confusing alongside the README's own inconsistent naming (see Section 2 table).
- **Suggested fix:** Remove `class ExpenseManager;` or actually rename `Splitwise` to `ExpenseManager` for consistency with the forward declaration and align naming across code/README.

### Issue C-1: Currency formatting inconsistency
- **Severity:** Low
- **File/line:** expense-manager.cpp:488/490 (`fixed << setprecision(2)`) vs. expense-manager.cpp:656/658 (`balance.second` printed without `fixed`/`setprecision`), vs. expense-manager.cpp:423/460 (`to_string(amount)` — always 6 decimal digits since `to_string(double)` doesn't respect stream formatting)
- **Problem:** Some money output uses 2-decimal fixed formatting, some uses raw `<<` (locale/precision-dependent), and some uses `to_string()` on a `double`, which always renders 6 decimal places (e.g. "800.000000").
- **Why it's a problem:** User-facing monetary output is inconsistent within the same program run — unprofessional and error-prone for a financial tool; `to_string(800.0)` renders `"800.000000"` while other paths render `"800.00"`.
- **Suggested fix:** Centralize a `formatCurrency(double)` helper using `fixed << setprecision(2)` and use it at every output site instead of ad hoc formatting/`to_string`.

## 5. Performance Analysis

- **Bottlenecks:** None significant at current scale — the program processes a handful of users/expenses in `main()`. However, `Group::getUserByuserId` (expense-manager.cpp:277-286) is O(n) linear scan over `members` invoked on every expense addition, settlement, and balance display (expense-manager.cpp:422, 456-457, 473, 484), rather than O(1) map lookup against `Splitwise::users`. At real-world group sizes this is still cheap, but it's an unnecessary linear scan when a `map<string,User*>` (as already used in `Splitwise`) would give O(log n).
- **DB/query inefficiencies:** N/A — no database; all data is in-memory `std::map`/`std::vector`.
- **Unnecessary recomputation:** `DebtSimplifier::simplifyDebts` (expense-manager.cpp:182-271) is only invoked explicitly (`simplifyGroupDebts`), which is appropriate, but it recomputes net amounts from scratch each call rather than maintaining a running net balance — fine at this scale, not fine if groups grow large and this is called frequently.
- **I/O inefficiencies:** All output goes directly to `cout` synchronously inside business logic (e.g. expense-manager.cpp:421-440), which couples core computation to console I/O and would need to be refactored entirely to support any other output channel (file, network, UI) or to be performant under high output volume.

## 6. Security Audit

This is a local, offline console application with no network exposure, no persistence layer, and no authentication surface, so classical web/API vulnerabilities (SQL injection, XSS, CSRF) do not apply. Relevant findings:

- **Input validation issues:** None of the public methods validate numeric input for sanity (e.g. negative `amount` in `addExpense`/`addIndividualExpense`/`settlePayment` is never rejected — expense-manager.cpp:390, 445, 622, 610). A negative expense amount would flow straight into the balance ledger and invert debts silently.
- **Injection risks:** Not applicable (no SQL, no shell exec, no deserialization).
- **Auth/authz flaws:** There is no authentication or authorization model at all — any caller can call `settlePayment`, `addExpense`, or `removeMember` for any user/group with no notion of "who is making this call." This is acceptable for a design-exercise/demo but would need an authz layer (e.g. verifying the caller is a group member) before being usable as a real service.
- **Secrets handling:** N/A — no secrets, credentials, or config files are present in the repo.
- **API vulnerabilities:** N/A — no exposed API surface (no HTTP server, no CLI argument parsing of untrusted input).

## 7. Scalability Concerns

- **What breaks at scale and why:** `Group::getUserByuserId`'s O(n) scan (Section 5) degrades linearly with group size. More critically, Issue A-1 (two disjoint balance stores) means the system cannot scale to "a user's balance across many groups plus individual debts" without a fundamental redesign — the current per-group and per-user maps have no unifying query layer.
- **State management issues:** All state lives in raw pointers inside static/singleton maps (`Splitwise::users/groups/expenses`, expense-manager.cpp:509-511) with no persistence; a process restart loses all data. There's no serialization, no database, no snapshotting.
- **Concurrency problems:** None of the data structures (`std::map`, `std::vector` on `User`/`Group`/`Splitwise`) are protected by any locking. The `Splitwise::getInstance()` double-checked-without-locking pattern (Issue B-4) and every mutation method (`addExpense`, `updateGroupBalance`, `updateBalance`, etc.) would race under concurrent access. The code is fundamentally single-threaded and would require a full concurrency-control pass (mutexes around ledger mutations, or moving to an actor/message-queue model) before being used in any multi-user server context.

## 8. Testing & Reliability

- **Test coverage analysis:** There are no automated tests in the repository (no test framework, no `test/` directory, no CI config). `main()` (expense-manager.cpp:682-739) functions as a manual, print-and-eyeball smoke test/demo script, not an assertion-based test suite.
- **Missing test cases:** No coverage exists for: empty `involvedUsers` (Issue A-4), mismatched `splitValues` length (Issue A-3), negative amounts, removing a non-existent user/group, settling more than the outstanding balance, `PercentageSplit` values not summing to 100, concurrent access, or the `User.balances` vs. `Group.groupBalances` divergence (Issue A-1) itself.
- **Flaky logic:** The floating-point epsilon comparisons (`abs(x) < 0.01`, used at expense-manager.cpp:130, 216, 218, 262, 265, 357, 360, 374) are a reasonable but ad hoc convention repeated as a magic number in 7+ places rather than a named constant — a future edit to one occurrence without the others would introduce inconsistent rounding behavior.

## 9. Dependency & DevOps Review

- **Dependency risks:** No external dependencies — the project relies solely on the C++ standard library plus the GCC-specific `<bits/stdc++.h>` umbrella header (expense-manager.cpp:7), which will fail to compile on non-GCC toolchains (e.g. MSVC, and by default Clang without `-fgnu-keywords`-style shims). This limits portability without adding any real value, since all needed headers (`<iostream>`, `<string>`, `<vector>`, `<map>`, `<algorithm>`, `<iomanip>`) are already explicitly included above it.
- **Docker/config issues:** No Dockerfile, no build scripts (no `Makefile`/`CMakeLists.txt`), and no `.gitignore` entries relevant to C++ build artifacts were inspected beyond the repo's [.gitignore](.gitignore). There is no reproducible build process — a contributor must manually know to run something like `g++ -std=c++17 expense-manager.cpp -o expense-manager`.
- **Environment handling:** N/A — no environment variables, no configuration files, no secrets management needed at this scale.

## 10. Prioritized Fix Roadmap

1. **Fix Issue A-3** (missing validation in `ExactSplit`/`PercentageSplit`) — prevents out-of-bounds UB and silent ledger corruption; highest-risk correctness bug.
2. **Fix Issue A-1** (unify `User.balances` and `Group.groupBalances` into one ledger) — the core "does the app compute correct debts" guarantee depends on this.
3. **Fix Issue A-4** (guard against empty `involvedUsers` / division by zero in `EqualSplit`).
4. **Fix Issue A-2** (memory leaks: adopt smart pointers for `SplitStrategy`, `User`, `Group`, `Expense`).
5. **Standardize error handling** (Issue B-3) and wrap `main()` in try/catch so invalid input degrades gracefully instead of crashing.
6. **Deduplicate balance-update logic** (Issue B-1) into a single `Ledger`/settlement helper, consistent with the README's documented (but unimplemented) design.
7. **Add a `Makefile`/`CMakeLists.txt`** and a basic assertion-based test suite covering the missing cases in Section 8, so regressions in items 1-6 are caught automatically.
8. **Clean up dead code and naming inconsistencies** (Issues B-5, B-6, C-1) and reconcile README.md's architecture section with the actual class names (`Splitwise` vs. `ExpenseService`, no `NotificationCenter`, no `Ledger`) or implement the missing pieces to match the documented design.
9. **Add basic input validation** (negative amounts, non-existent IDs surfaced consistently) across all public `Splitwise`/`Group` methods, closing the security-relevant gaps noted in Section 6.
10. **Address thread-safety** (Issue B-4) only if/when the project moves toward any concurrent or server-based usage — not urgent for the current single-threaded console demo.
