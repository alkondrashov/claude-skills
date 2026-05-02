---
name: actual-budget
description: Operate the Actual Budget API — manage accounts, add/import transactions, set budgets, and query data. Use when the user asks to interact with their Actual Budget instance.
tools: Bash
---

# Actual Budget API Skill

Interact with the self-hosted Actual Budget instance at `https://84.8.152.88`.

## Setup

The `@actual-app/api` Node.js package is installed at `/tmp/actual-api/`.
If it's missing (e.g. after a reboot), reinstall:

```bash
mkdir -p /tmp/actual-api && cd /tmp/actual-api && npm init -y && npm install @actual-app/api
```

Always run scripts with `NODE_TLS_REJECT_UNAUTHORIZED=0` to accept the self-signed cert:

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 node /tmp/actual-api/myscript.js
```

## Authentication

Password is stored at `~/.oci/actual_budget_password` (chmod 600).

Standard boilerplate for every script:

```js
const api = require('@actual-app/api');
const fs = require('fs');

const SERVER_URL = 'https://84.8.152.88';
const PASSWORD = fs.readFileSync('/Users/kondrashov/.oci/actual_budget_password', 'utf8').trim();
const DATA_DIR = '/tmp/actual-data';

fs.mkdirSync(DATA_DIR, { recursive: true });

async function main() {
  await api.init({ dataDir: DATA_DIR, serverURL: SERVER_URL, password: PASSWORD });
  await api.downloadBudget(BUDGET_GROUP_ID); // see Known Budget below
  // ... do work ...
  await api.shutdown();
}

main().catch(err => { console.error(err); process.exit(1); });
```

## Known Budget

| Field | Value |
|---|---|
| Name | My Finances |
| groupId (syncId) | `d2cbfd7d-774d-43f5-8bf5-8d1bfae3e139` |
| cloudFileId | `bee96c58-0810-465d-8391-40678d69ace5` |

> **Important**: `api.downloadBudget()` takes the **groupId** (syncId), NOT the cloudFileId.

```js
const BUDGET_GROUP_ID = 'd2cbfd7d-774d-43f5-8bf5-8d1bfae3e139';
```

## Known Accounts

| Name | Type | ID |
|---|---|---|
| Barclays Current | checking | `3ec055ef-11ec-40f3-9b55-608ad0738387` |
| Marcus Savings | savings | `d87eead7-3054-41dc-8739-238d000b2228` |
| Amex Credit Card | credit | `281f46e6-8ba4-42d3-9d81-7d0abaef6d87` |

Look up live if unsure:

```js
const accounts = await api.getAccounts();
console.log(accounts.map(a => ({ id: a.id, name: a.name, type: a.type })));
```

## Amounts

All amounts are **integers in pence** (hundredths of the currency unit, no decimals).
- £10.00 → `1000`
- Expenses are **negative**: -£4.25 → `-425`
- Income is **positive**: £3,000.00 → `300000`

## Common Operations

### Add transactions

```js
await api.addTransactions(accountId, [
  {
    date: '2026-05-01',       // YYYY-MM-DD string
    payee_name: 'Tesco',
    amount: -4250,            // pence, negative = expense
    category: categoryId,     // optional — null to leave uncategorised
    notes: 'Weekly shop',
  },
]);
```

### Import transactions (reconcile — skips duplicates)

```js
await api.importTransactions(accountId, transactions);
```
Use `importTransactions` when syncing from a bank export — it deduplicates by date/amount/payee.
Use `addTransactions` when you know the transactions are new.

### Get categories

```js
const cats = await api.getCategories();
// cats is a flat array: { id, name, group_id, is_income, hidden }
const food = cats.find(c => /food|grocer/i.test(c.name));
```

### Get category groups

```js
const groups = await api.getCategoryGroups();
// groups[n].categories is an array of category objects
```

### Create a category

```js
const groupId = (await api.getCategoryGroups()).find(g => g.name === 'Usual Expenses').id;
await api.createCategory({ name: 'Subscriptions', group_id: groupId });
```

### Set budget amount for a category

```js
// month format: '2026-05' (YYYY-MM)
await api.setBudgetAmount('2026-05', categoryId, 50000); // £500.00
```

### Get budget for a month

```js
const month = await api.getBudgetMonth('2026-05');
// month.categoryGroups[n].categories[n].budgeted / .spent / .balance
```

### Query transactions

```js
const { data } = await api.runQuery(
  api.q('transactions')
    .filter({ 'account.id': accountId })
    .select(['date', 'amount', 'payee.name', 'category.name', 'notes'])
    .run()
);
```

### Create an account

```js
const id = await api.createAccount(
  { name: 'HSBC Current', type: 'checking' },
  0  // starting balance in pence
);
```

## HTTP API (lightweight — no Node.js needed)

For simple reads, use curl directly:

```bash
PASSWORD=$(cat ~/.oci/actual_budget_password)

# Get auth token
TOKEN=$(curl -sk -X POST https://84.8.152.88/account/login \
  -H "Content-Type: application/json" \
  -d "{\"password\":\"$PASSWORD\"}" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['token'])")

# List budget files
curl -sk https://84.8.152.88/sync/list-user-files \
  -H "X-ACTUAL-TOKEN: $TOKEN" | python3 -c "import json,sys; print(json.dumps(json.load(sys.stdin), indent=2))"
```

## Workflow Guidelines

- **Always `await api.shutdown()`** at the end — it syncs changes back to the server.
- **Clear stale local data** if you hit "Budget not found": `rm -rf /tmp/actual-data && mkdir /tmp/actual-data`
- **Write scripts to `/tmp/actual-api/`** and run them with `NODE_TLS_REJECT_UNAUTHORIZED=0 node /tmp/actual-api/<script>.js`
- **Before destructive operations** (deleting accounts, clearing transactions): show the user what will change and confirm.
- **After bulk imports**: verify with `api.getTransactions(accountId)` or check the UI.
