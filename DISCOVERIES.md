# Credit Karma GraphQL API Discoveries

Last updated: January 2026

## Overview

Credit Karma uses a GraphQL API at `https://api.creditkarma.com/graphql`. This document captures discoveries made while exploring the API for data export capabilities.

## Authentication

- **Token**: Bearer token from `Authorization` header in browser requests
- **Token lifetime**: ~15 minutes before expiration
- **Required headers**:
  ```
  Authorization: Bearer <token>
  Content-Type: application/json
  Origin: https://www.creditkarma.com
  Referer: https://www.creditkarma.com/
  ck-client-name: navigation-web
  ck-client-version: 3.25.1
  ```

## API Limitations

- **Introspection disabled**: Returns 403 "Invalid Introspection Request"
- **Complex query structure**: Requires specific KPL (Karma Presentation Layer) fragments
- **Rate limiting**: Recommended 1.5s delay between requests

## Verified Working Operations

### Top-Level Operations

| Operation | TypeName | Description |
|-----------|----------|-------------|
| `dashboardV2` | DashboardV2 | Credit scores, account summaries for Equifax & TransUnion |
| `profileOverview` | ProfileOverview | User profile (name, address, contact info) |
| `myLoans` | MyLoansQuery | Personal loan view |
| `allLoans` | AllLoansQuery | Complete loans data |
| `mortgage` | MortgageQuery | Mortgage/home loan information |
| `savings` | Savings | Savings accounts and products |
| `insurance` | Insurance_Query | Insurance products and policies |
| `auto` | AutoQuery | Auto loans and vehicle financing |
| `home` | HomeQuery | Property and home equity data |
| `recsysFeed` | RecsysFeed | Personalized recommendations feed |

### Operations Under `prime.*`

| Operation | TypeName | Description |
|-----------|----------|-------------|
| `prime.networth` | Prime_NetworthLayout | Net worth overview with balance history charts |
| `prime.networthByAccountType` | Prime_NetworthLayout | Balance history by category (Cash, Investments, etc.) |
| `prime.transactionsHub` | Prime_Transactions | Transaction history with pagination |

### Operations That Did NOT Work

These may require specific input parameters or different query structure:
- creditScore, creditFactors, creditReport
- alerts, identity, spending
- linkedAccounts, settings, notifications
- member, tax, cards, fraud

## Historical Data Available

### Net Worth / Balance History

- **Date range**: January 2012 to present
- **Granularity**: Monthly data points
- **Data source**: `prime.networth` via captured JSON responses
- **Export file**: `creditkarma_balance_history.csv`

Sample data structure:
```csv
"Date","Account Type","Account Name","Balance"
"2012-01-31","Net Worth","Total","8202.25"
"2020-01-31","Net Worth","Total","123456.78"
```

### Transactions

- **Date range**: Back to at least 2014 (account creation)
- **Pagination**: Cursor-based, ~100 transactions per page
- **Export file**: `creditkarma_transactions.csv`
- **Script**: `fetch_credit_karma_transactions`

### Per-Account Balance History

Available but requires manual JSON capture from UI:
- Cash accounts
- Investment accounts
- Loan accounts
- Credit card accounts
- Real estate

## Scripts

### fetch_credit_karma_transactions

Exports all transactions to CSV with:
- Automatic retry with exponential backoff
- Rate limiting (1.5s between requests)
- Auto-resume via cursor file
- Interactive token refresh on expiration

```bash
export MY_ACCESS_TOKEN="your_token"
ruby fetch_credit_karma_transactions
```

### parse_balance_history

Parses captured GraphQL JSON responses for balance history:

```bash
ruby parse_balance_history [json_files...]
```

Output: `creditkarma_balance_history.csv` and `.json`

### explore_graphql

Discovers and tests GraphQL operations:

```bash
ruby explore_graphql --list        # Show known operations
ruby explore_graphql --analyze     # Analyze captured JSON files
ruby explore_graphql --introspect  # Try schema introspection
ruby explore_graphql --probe       # Probe for operations
ruby explore_graphql --query "..." # Run custom query
```

## GraphQL Types Discovered

From analyzing captured JSON responses (170+ types):

### Query Types
- AllLoansQuery
- MyLoansQuery
- DashboardV2
- ProfileOverview
- Savings
- Insurance_Query
- MortgageQuery
- AutoQuery
- HomeQuery

### Key Data Types
- Prime_NetworthLayout
- Prime_Transactions
- Prime_Transaction
- Prime_TransactionPage
- Prime_AmountOfUsd

### UI Component Types (KPL)
- KPLLineGraphV2DataPoint (contains balance history points)
- KPLViewGroup
- KPLBarChart
- KPLCardView
- KPLButtonView
- FabricCardAny
- FabricDataVisualizationGroup

## Data Structure Notes

### Balance History Location

Balance history is nested deep in the response:
```
data.prime.networth.cards[].item.views[].dataVisualizationGroupDataSets[]
  .dataVisualizationDataSet.lines[].points[]
```

Each point contains:
- `xValueLabel.spans[0].text` - Date string (e.g., "Jan 31, 2012")
- `yValue` - Balance amount

### Transaction Structure

```
data.prime.transactionsHub.transactionPage.transactions[]
```

Each transaction contains:
- id, date, description, status
- amount.value, amount.asCurrencyString
- account (id, name, type, providerName)
- category (id, name, type)
- merchant (id, name)

### Pagination

Transactions use cursor-based pagination:
```
transactionPage.pageInfo {
  startCursor
  endCursor
  hasNextPage
  hasPreviousPage
}
```

## Token Scopes

From JWT payload, available scopes include:
- mortgage, assets, auth, engagement
- tax, credit, autos, cards
- personal-loans, encryption, fraud
- net-worth, identity

## Future Exploration

Potential areas to investigate:
1. Credit score history (may be in dashboardV2 sub-fields)
2. Account-level transaction filtering
3. Spending analytics/budgets
4. Alert and notification history
5. Credit report details

## Files in Repository

```
fetch_credit_karma_transactions  # Main transaction export script
parse_balance_history            # Balance history JSON parser
explore_graphql                  # API exploration tool
DISCOVERIES.md                   # This file
.creditkarma_cursor              # Resume cursor (gitignored)
creditkarma_transactions.csv     # Transaction export (gitignored)
creditkarma_balance_history.csv  # Balance export (gitignored)
```
