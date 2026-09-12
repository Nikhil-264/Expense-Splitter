# Expense Splitter

A clean, object-oriented, Splitwise-style expense manager written in modern C++ as a Low-Level Design (LLD) exercise. It tracks shared expenses across groups and one-off individual expenses between friends, computes who owes whom, and can minimize the number of payments needed to settle everyone up.

The entire system lives in a single file — [expense-manager.cpp](expense-manager.cpp) — so it's easy to read end-to-end in one sitting, from data model to `main()`.

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture & Design Patterns](#architecture--design-patterns)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [What the Demo Does](#what-the-demo-does)
- [Sample Output](#sample-output)
- [Core Concepts](#core-concepts)
- [Documentation](#documentation)
- [Known Limitations](#known-limitations)
- [Roadmap / Ideas for Extension](#roadmap--ideas-for-extension)
- [Contributing](#contributing)

## Overview

Splitting shared costs — rent, groceries, a trip, a dinner — always comes down to the same problem: someone pays, several people owe a share, and over time a tangled web of "who owes whom" builds up. This project models that problem the way a real expense-sharing app would:

- **Groups** (e.g. "Hostel Expenses", "Goa Trip") have members and their own running balance sheet.
- **Expenses** are recorded once, split among the involved people using a pluggable rule, and immediately reflected in the group's balances.
- **Individual expenses** let two people settle up outside of any group (e.g. "I bought you coffee").
- **Debt simplification** collapses a group's tangled balance graph into the smallest possible number of actual payments.
- **Notifications** are pushed to every group member whenever an expense or settlement happens.

## Key Features

- **Three split types**, chosen per expense:
  - **Equal** — the amount is divided evenly among everyone involved.
  - **Exact** — the payer specifies exactly how much each person owes.
  - **Percentage** — the payer specifies what percentage of the total each person owes.
- **Per-group balance tracking** — each group maintains its own pairwise balance matrix (who owes whom, and how much), independent of other groups.
- **Combined per-user balance view** — `showUserBalance` aggregates a user's balances across every group they belong to *and* their individual (non-group) transactions into a single "total you owe / total others owe you" summary.
- **Minimal settlements (debt simplification)** — a greedy netting algorithm computes each member's net position and produces the smallest set of payments that settles the group, instead of leaving every historical pairwise debt in place.
- **Group membership rules** — a member cannot leave a group while they still have an outstanding balance with anyone else in it.
- **Observer-based notifications** — every group member is notified (console output, in this implementation) whenever a new expense is added or a settlement is made.
- **Extensible by design** — new split types, notification channels, or persistence layers can be added without touching existing business logic (see [Architecture & Design Patterns](#architecture--design-patterns)).

## Architecture & Design Patterns

The codebase is organized around four classic design patterns, each mapped directly onto a requirement:

| Pattern | Purpose in this project | Where |
|---|---|---|
| **Strategy** | Pluggable split calculation — `EqualSplit`, `ExactSplit`, `PercentageSplit` all implement the same `SplitStrategy` interface | `SplitStrategy` and subclasses |
| **Factory** | Picks the right `SplitStrategy` implementation from a `SplitType` enum, so callers never branch on split type themselves | `SplitFactory::getSplitStrategy` |
| **Observer** | Broadcasts balance-changing events (new expense, settlement) to every group member | `Observer` interface, implemented by `User`; published via `Group::notifyMembers` |
| **Singleton + Facade** | `Splitwise` is the single, process-wide entry point exposing simple methods (`createUser`, `addExpenseToGroup`, `showUserBalance`, ...) that hide the internal `User`/`Group`/`Expense` object graph | `Splitwise` class |

On top of these, a **greedy min-cash-flow algorithm** (`DebtSimplifier`) nets out a group's balance matrix by repeatedly matching the largest remaining creditor against the largest remaining debtor — the same technique real settle-up features use to reduce transaction count.

For a full, line-referenced walkthrough of how these pieces fit together (including an end-to-end trace of adding an expense, module-by-module breakdowns, and design trade-offs), see [SYSTEM_EXPLANATION.md](SYSTEM_EXPLANATION.md).

## Project Structure

```
Expense-Splitter/
├── expense-manager.cpp          # Entire application: models, patterns, and main() demo
├── README.md                    # You are here
├── CODE_AUDIT.md                # Line-referenced engineering audit (bugs, risks, roadmap)
├── SYSTEM_EXPLANATION.md        # Deep-dive walkthrough of architecture & data flow
├── TROUBLESHOOTING_AND_FIXES.md # Log of problems found and fixes actually applied
└── .gitignore
```

## Getting Started

### Prerequisites

- A C++ compiler with C++17 support. The code has been built and verified with **GCC (MinGW)** on Windows.
- Note: [expense-manager.cpp](expense-manager.cpp) includes `<bits/stdc++.h>`, a GCC-specific umbrella header. If you're using MSVC or a strict Clang setup, either switch to GCC or replace that include with the specific standard headers already listed above it (`<iostream>`, `<string>`, `<vector>`, `<map>`, `<algorithm>`, `<iomanip>`).

### Build

```bash
g++ -std=c++17 expense-manager.cpp -o expense-manager
```

On Windows this produces `expense-manager.exe`.

### Run

```bash
./expense-manager
```

No arguments, no configuration, no external dependencies — it runs a scripted demo scenario and prints everything to the console.

## What the Demo Does

`main()` walks through a realistic end-to-end scenario:

1. Creates 4 users: Aditya, Rohit, Manish, and Saurav.
2. Creates a group, "Hostel Expenses", and adds all 4 users to it.
3. Adds a **Lunch** expense (Rs 800, equal split among all 4).
4. Adds a **Dinner** expense (Rs 700, exact split among 3 of them).
5. Prints the group's balances.
6. Runs **debt simplification** on the group and prints the reduced balances.
7. Adds an **individual** expense ("Coffee") between two users, outside the group.
8. Prints each user's **combined balance** (individual + every group they're in).
9. Attempts to remove a member with an outstanding balance (rejected), settles their debt, then successfully removes them.
10. Prints the group's final balances.

## Sample Output

A trimmed excerpt from running the demo (see [TROUBLESHOOTING_AND_FIXES.md](TROUBLESHOOTING_AND_FIXES.md) for the full before/after comparison that motivated the current balance logic):

```
=========== Balance for Rohit ====================
Total you owe: Rs 200.00
Total others owe you: Rs 20.00
Detailed balances:
  You owe Manish: Rs 200.00
  Saurav owes you: Rs 20.00
```

This reflects Rohit's Rs 200 group debt (from the Dinner expense) *and* his Rs 20 individual credit (from splitting a Rs 40 coffee with Saurav) in one unified view.

## Core Concepts

- **`Split`** — a single `{userId, amount}` pair describing one person's share of an expense.
- **`SplitStrategy`** — an interface with three implementations (`EqualSplit`, `ExactSplit`, `PercentageSplit`) that turn a total amount and a list of participants into a list of `Split`s.
- **`Expense`** — an immutable record of one spend event: description, amount, payer, computed splits, and (optionally) the group it belongs to.
- **`User`** — a person, who also acts as an `Observer` to receive notifications, and who tracks their own individual (non-group) balances.
- **`Group`** — owns membership, its own expense history, and its own pairwise balance matrix; enforces that members can't leave with an outstanding balance.
- **`DebtSimplifier`** — a stateless utility that nets a group's balance matrix down to the minimum number of payments.
- **`Splitwise`** — the singleton facade that ties everything together and is the only class `main()` talks to directly.

## Documentation

This repository includes deeper written documentation alongside the code itself:

- **[CODE_AUDIT.md](CODE_AUDIT.md)** — a critical, line-referenced engineering audit covering architecture review, code quality, a detailed issue log (bugs, security, performance, scalability), test coverage gaps, and a prioritized fix roadmap.
- **[SYSTEM_EXPLANATION.md](SYSTEM_EXPLANATION.md)** — a teaching-style deep dive: system purpose, end-to-end request flow, module-by-module breakdown, data flow diagrams, design-decision trade-offs, and how to safely extend the system.
- **[TROUBLESHOOTING_AND_FIXES.md](TROUBLESHOOTING_AND_FIXES.md)** — a running log of specific problems found in the code, why each one mattered, and exactly what was changed to fix it, with before/after snippets and verification notes.

## Known Limitations

This is an LLD learning project, not a production system — some limitations are intentional scoping, others are open issues tracked in [CODE_AUDIT.md](CODE_AUDIT.md):

- Everything is in-memory; there is no persistence, so all data is lost when the program exits.
- No test suite exists yet — `main()` serves as a manual, print-and-eyeball demo rather than an automated regression check.
- `ExactSplit` and `PercentageSplit` don't yet validate that their inputs sum correctly, and a few other input-validation gaps remain open (see the audit's issue log).
- No network layer, authentication, or multi-user concurrency — it's a single-process console application.

## Roadmap / Ideas for Extension

Documented in more depth in [SYSTEM_EXPLANATION.md](SYSTEM_EXPLANATION.md) §11 and [CODE_AUDIT.md](CODE_AUDIT.md) §10:

- Introduce a single unified `Ledger` class that both group and individual balance flows write through directly.
- Add input validation for split amounts/percentages and negative expense values.
- Replace raw owning pointers with smart pointers (`unique_ptr`) to eliminate memory leaks.
- Add a build system (`Makefile`/`CMakeLists.txt`) and an assertion-based test suite.
- Add a persistence layer (e.g. JSON serialization) so data survives across runs.
- Add real notification channels (email/push) by implementing new `Observer`s.

## Contributing

Found a bug or have an improvement idea? Open an issue or a pull request. If you fix something described in [CODE_AUDIT.md](CODE_AUDIT.md), please also add an entry to [TROUBLESHOOTING_AND_FIXES.md](TROUBLESHOOTING_AND_FIXES.md) describing the problem, why it mattered, and what changed — that log is meant to stay in sync with what's actually been fixed versus what's still open.
