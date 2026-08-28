# Expense Splitter

A clean, object-oriented expense manager that supports Equal, Exact, and Percentage splits with a focus on scalable Low-Level Design (LLD).

## Overview

The app tracks group expenses, computes who owes whom, and minimizes settlements. It’s designed to be easy to extend with new split types and notification channels.

## Key Features

Split types: Equal · Exact · Percentage (plugged via strategies)

Minimal settlements: Custom greedy netting reduced inter-user payments by ~90% in test scenarios

Real-time updates: Observer-based notifications to 100% of group members on balance changes/transactions

Extensible design: Clear boundaries for adding new splits, payment methods, or UIs

## Architecture & Patterns

Strategy – pluggable SplitStrategy for Equal/Exact/Percent

Observer – NotificationCenter pushes balance updates to users/channels

Factory – SplitFactory builds the right split from input

Singleton – Ledger (source of truth for balances)

Facade – ExpenseService exposes simple APIs to the UI/CLI
