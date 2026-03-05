# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Finance Tracker is a self-hosted personal finance application with a FastAPI backend and React/TypeScript frontend. The project uses Docker for PostgreSQL and supports importing CSV files from multiple banks/credit cards.

## Development Commands

### Backend

```bash
cd finance-tracker-backend

# Run locally (requires Python 3.11+)
python3 -m uvicorn app.main:app --reload

# Run tests (when implemented)
pytest

# Check syntax
python3 -m py_compile app/services/csv_import/parsers/my_parser.py
```

### Frontend

```bash
cd finance-tracker-frontend

# Development server
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# TypeScript type check
npx tsc --noEmit
```

### Docker

```bash
# Start all services (backend, frontend, PostgreSQL)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down

# Reset database (WARNING: deletes all data)
docker-compose down -v
```

### Database Migrations

**Migrations run automatically when you start Docker Compose!** No manual steps needed.

```bash
# View migration status
docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "SELECT * FROM schema_migrations;"

# View migration logs
docker logs finance-tracker-backend | grep -i migration

# Manual migration (if needed)
docker exec -i finance-tracker-db psql -U financeuser -d financedb < finance-tracker-backend/migrations/005_add_splitwise_split_flag.sql

# Backup / restore database
docker exec finance-tracker-db pg_dump -U financeuser financedb > backup.sql
docker exec -i finance-tracker-db psql -U financeuser -d financedb < backup.sql
```

**Migration System Details:**
- Automated migration runner in `scripts/run_migrations.sh`
- Tracks applied migrations in `schema_migrations` table — only has a `version` column (no `description` column)
- Runs on container startup via `scripts/entrypoint.sh`
- Idempotent - safe to restart containers
- INSERT format: `INSERT INTO schema_migrations (version) VALUES ('005') ON CONFLICT (version) DO NOTHING;`

**Applied Migrations:**
- `001_add_import_tracking.sql` - import_sessions, category_rules, bank_parser_templates tables; adds import_session_id, raw_data to transactions; adds bank_name, account_number_last4 to accounts
- `005_add_splitwise_split_flag.sql` - adds `splitwise_split BOOLEAN NOT NULL DEFAULT FALSE` to transactions

## Architecture

### Backend: CSV Import System (Strategy Pattern)

**Location**: `finance-tracker-backend/app/services/csv_import/`

**Key Components**:

1. **Base Parser** (`base_parser.py`):
   - `CSVParser` abstract base class
   - `ParsedTransaction` dataclass for standardized output
   - All parsers implement: `get_name()`, `get_display_name()`, `detect(csv_content)` → float 0.0-1.0, `parse(csv_content)` → List[ParsedTransaction], `get_required_headers()`

2. **Parser Registry** (`parser_registry.py`):
   - Singleton pattern for parser discovery
   - `registry.register(parser)`, `registry.detect_parser(csv_content)`, `registry.get_parser(name)`

3. **Individual Parsers** (`parsers/` directory):
   - `amex_parser.py` - American Express Credit Card (`Trans. Date`, `Description`, `Amount`; positive=expense)
   - `discover_bank_parser.py` - Discover Card (`Trans. Date`, `Post Date`, `Description`, `Amount`, `Category`; positive=expense)
   - `discover_savings_parser.py` - Discover Savings Account (`Transaction Date`, `Transaction Description`, `Transaction Type`, `Debit`, `Credit`, `Balance`; separate debit/credit columns; Credit=income, Debit=expense/transfer)
   - `chase_bank_parser.py` - Chase Bank
   - `chase_bank_pdf_parser.py` - Chase Bank PDF
   - `capital_one_parser.py`, `citi_parser.py`, `boa_parser.py`, `wells_fargo_parser.py` (registered in parsers/__init__.py)

4. **Import Service** (`import_service.py`): orchestrates preview + confirm flow
5. **Category Suggester** (`category_suggester.py`): keyword/regex auto-categorization
6. **Transfer Detector** (`transfer_detector.py`): detects transfers between accounts

**Parser Registration**: Both `parsers/__init__.py` (export) AND `csv_import/__init__.py` `initialize_parsers()` must be updated when adding a new parser.

### Backend: API Structure

**Location**: `finance-tracker-backend/app/api/`

**Routers** (all prefixed with `/api/v1`):
- `auth.py` - `/auth/*` - Registration, login, user info
- `accounts.py` - `/accounts/*` - Account CRUD
- `transactions.py` - `/transactions/*` - Transaction CRUD + CSV import
- `parsers.py` - `/parsers/*` - List parsers, auto-detect
- `categories.py` - `/categories/*` - Categories + rules CRUD
- `settings.py` - `/splitwise/*` - Splitwise integration (credentials, friends, groups, create expense)
- `analytics.py` - `/analytics/*` - Sankey diagram data

**Important Endpoints**:
- `GET /transactions` - List with filters (start_date, end_date, account_id, category_id, transaction_type)
- `PATCH /transactions/{id}` - Update mutable fields (currently: category_id)
- `DELETE /transactions/{id}` - Delete a transaction
- `POST /transactions/import-csv/preview` - Preview CSV without saving
- `POST /transactions/import-csv/confirm` - Execute import with category mappings
- `POST /parsers/detect` - Auto-detect CSV format
- `GET /parsers` - List all available parsers
- `GET /splitwise/credentials` - Check if Splitwise API key is configured
- `GET /splitwise/friends` - List Splitwise friends
- `GET /splitwise/groups` - List Splitwise groups (skips id=0 "Non-group expenses")
- `POST /splitwise/create-expense` - Create split expense on Splitwise
- `GET /analytics/sankey` - Sankey data (params: start_date, end_date, include_transfers)

### Backend: Database Models

**Location**: `finance-tracker-backend/app/models/`

**Core Models**:
- `user.py` - User accounts with JWT auth
- `account.py` - Financial accounts; fields: `bank_name`, `account_number_last4` (for transfer matching)
- `transaction.py` - Transactions; fields: `import_session_id`, `raw_data` (JSONB), `splitwise_split` (Boolean, default False)
- `category.py` - Hierarchical categories with colors/icons

**Import Tracking Models**:
- `import_session.py` - Tracks each CSV import
- `category_rule.py` - Auto-categorization rules
- `bank_parser_template.py` - Custom parser configurations

### Backend: Splitwise Integration

**Location**: `app/api/settings.py` + `app/services/splitwise_service.py`

**Key implementation details**:
- Uses `splitwise` Python SDK with API key auth (no OAuth)
- `client.createExpense()` returns `(Expense, Errors)` tuple in newer SDK versions — always check `isinstance(result, tuple)`
- When creating an expense, the current user MUST be included as a participant (Splitwise rejects expenses not involving yourself)
- `paid_share = txn_amount` for current user, `owed_share = txn_amount - sum(friends_owed)` for current user
- Friend shares are scaled per-transaction proportionally (friend_ratio × txn_amount)
- Skip current user from participant list when iterating group members (avoid "included multiple times" error)
- `group_id` is optional — sent when a group is selected in frontend
- After successful expense creation, set `txn.splitwise_split = True` and `db.commit()`

### Backend: Sankey Service

**Location**: `app/services/sankey_service.py`

**Critical gotcha**: Income and expense category nodes must never share the same name — this creates a cycle (NodeX → Cash Flow → NodeX) that causes recharts Sankey to recurse infinitely and crash with "Maximum call stack size exceeded".
- Income uncategorized → `"Uncategorized Income"`
- Expense nodes that collide with income node names → append `" (Expenses)"`
- Reserved names `"Cash Flow"` and `"Surplus"` also protected from collision
- The SQLAlchemy join to Account uses explicit `onclause`: `.join(Account, Transaction.account_id == Account.id)` (two FK paths exist)

### Frontend: Pages

**Location**: `finance-tracker-frontend/src/pages/`

- `DashboardPage.tsx` - Main dashboard
- `TransactionsPage.tsx` - Transaction list with filters, inline category editing, bulk delete, Splitwise split
- `AnalyticsPage.tsx` - Sankey money flow diagram; month picker (not date range) defaulting to current month

### Frontend: Multi-Step Import Wizard

**Main Component**: `CSVUploadModalNew.tsx`
- 4-step wizard; Step 1: Upload & detect; Step 2-3: Preview & categorize; Step 4: Confirmation
- Transaction descriptions are editable inline in the preview table
- `handleDescriptionChange` migrates both `categoryMappings` and `typeOverrides` keys when description changes

**Sub-components** (`src/components/csv-import/`):
- `TransactionPreviewTable.tsx` - Pagination (50/page), bulk select, duplicate highlighting, inline editable description
- `CategorySelector.tsx` - Searchable dropdown with color previews

### Frontend: Splitwise Modal

**Component**: `src/components/SplitwiseSplitModal.tsx`

- Accordion UI: Groups (above) then Friends
- Favourites stored in localStorage (`sw_fav_friends`, `sw_fav_groups`) — starred items sort to top
- Split types: `equal`, `exact`, `percent`
- For `equal` split: group-level checkbox selection (whole group at once); members shown as info-only
- For `exact`/`percent`: individual member selection
- Group selection is exclusive (selecting one group disables all others and all friends)
- `deriveGroupId()` sends `group_id` to backend when a group is selected
- Props: `onSuccess(message: string)`, `onError(message: string)` — uses toast notifications

### Frontend: Toast Notifications

**Component**: `src/components/Toast.tsx`
- `useToast()` hook returns `{ toasts, showToast, dismissToast }`
- `showToast(message, 'success' | 'error')` — auto-dismisses after 4 seconds
- `<ToastContainer toasts={toasts} onDismiss={dismissToast} />` — fixed position top-right

### Frontend: Inline Category Editing (TransactionsPage)

- Clicking a category badge (or "**+ Add category**" placeholder) opens an inline `<select>` dropdown
- On change: calls `PATCH /transactions/{id}` via `updateTransactionCategory()`, updates local state, shows toast
- No full list reload needed — only the affected transaction row is updated

## Important Patterns & Conventions

### CSV Import Workflow

1. Upload CSV → `/parsers/detect` → auto-detect format
2. `/transactions/import-csv/preview` → parse, suggest categories, detect duplicates/transfers (no DB writes)
3. User reviews in UI (can edit descriptions, assign categories, change types, remove rows)
4. `/transactions/import-csv/confirm` → save with user-reviewed categories

### Adding a New Bank Parser

1. Create `parsers/my_bank_parser.py` — inherit from `CSVParser`, implement all abstract methods
2. Export from `parsers/__init__.py`
3. Import and instantiate in `csv_import/__init__.py` `initialize_parsers()`

### Database Schema Changes

1. Create `migrations/00X_description.sql`
2. Use `IF NOT EXISTS` clauses; INSERT format: `INSERT INTO schema_migrations (version) VALUES ('00X') ON CONFLICT (version) DO NOTHING;`
3. The `schema_migrations` table only has a `version` column — do NOT include `description` in the INSERT

### SQLAlchemy Join Gotcha

`Transaction` has two FK paths to `Account` (`account_id` and `transfer_to_account_id`). Always use explicit onclause:
```python
.join(Account, Transaction.account_id == Account.id)
```

### Backend Model Relationships

- User → Accounts, Transactions, Categories, ImportSessions, CategoryRules (all one-to-many)
- Account → Transactions (one-to-many)
- Category → Transactions (one-to-many)
- ImportSession → Transactions (one-to-many)

### Frontend State Management

- **Authentication**: Context API in `lib/auth.tsx`
- **API Calls**: Axios with interceptors in `services/api.ts`
- **Form State**: React hooks (useState, useEffect)

## Key Files Reference

### Backend Core
- `app/main.py` - FastAPI app, router registration
- `app/core/config.py` - Settings from environment variables
- `app/core/database.py` - SQLAlchemy setup
- `app/core/security.py` - JWT and password hashing

### Backend Services
- `app/services/csv_import/__init__.py` - Parser initialization (`initialize_parsers`)
- `app/services/csv_import/import_service.py` - Main import orchestrator
- `app/services/splitwise_service.py` - Splitwise SDK wrapper
- `app/services/sankey_service.py` - Sankey diagram data generation

### Backend API
- `app/api/transactions.py` - Transaction CRUD + CSV import endpoints
- `app/api/settings.py` - Splitwise endpoints + `SplitwiseExpenseCreate` schema
- `app/api/analytics.py` - Analytics/Sankey endpoint

### Frontend Core
- `src/App.tsx` - Routes and app structure
- `src/lib/auth.tsx` - Auth context provider
- `src/services/api.ts` - API client with all types and functions
- `src/types/csv.ts` - TypeScript types for CSV import

### Frontend Components
- `src/components/CSVUploadModalNew.tsx` - Import wizard
- `src/components/SplitwiseSplitModal.tsx` - Splitwise split UI
- `src/components/Toast.tsx` - Toast notification system
- `src/components/csv-import/TransactionPreviewTable.tsx` - Preview table
- `src/components/csv-import/CategorySelector.tsx` - Category dropdown

### Frontend Pages
- `src/pages/TransactionsPage.tsx` - Transactions list with inline category editing
- `src/pages/AnalyticsPage.tsx` - Sankey diagram (month picker, current month default)

## Environment Setup

**Backend** (`.env`):
```
DATABASE_URL=postgresql://financeuser:financepass@localhost:5432/financedb
SECRET_KEY=<generate with: python3 -c "import secrets; print(secrets.token_urlsafe(32))">
DEBUG=true
```

**Frontend** (`.env`):
```
VITE_API_URL=http://localhost:8000
```

## Current Status

✅ **Phase 1**: Multi-bank CSV import with parsers (Amex, Discover Card, Discover Savings, Chase Bank, Chase PDF, Capital One, Citi, BofA, Wells Fargo)
✅ **Phase 2**: Category intelligence & transfer detection
✅ **Phase 3**: Multi-step frontend import wizard (with inline description editing)
✅ **Phase 4**: Sankey diagram visualization (Analytics page)
✅ **Splitwise Integration**: Split expenses with friends/groups, toast feedback, splitwise_split badge on transactions
✅ **Inline Category Editing**: Click any category cell in Transactions page to update

## Frontend Design System

### Theme: "Obsidian Finance"
A dark, refined aesthetic inspired by premium finance apps (Robinhood-style). The visual language is data-dense but calm — dark surfaces, strong typographic hierarchy, and a single mint-green accent that signals positive value.

**Never deviate from this theme.** Do not introduce light backgrounds, white modals, or color schemes inconsistent with the palette below.

### Color Palette (CSS variables in `src/index.css`)
| Token | Value | Usage |
|---|---|---|
| `background` | `hsl(225 13% 6%)` | Page background |
| `card` | `hsl(228 12% 9%)` | Modal/card surfaces |
| `secondary` | `hsl(225 10% 13%)` | Input backgrounds, table headers, hover |
| `border` | `hsl(225 10% 16%)` | All borders and dividers |
| `foreground` | `hsl(220 20% 92%)` | Primary text |
| `muted-foreground` | `hsl(220 10% 50%)` | Labels, metadata, placeholders |
| `primary` | `hsl(158 100% 42%)` | Mint-green — CTAs, active states |
| `destructive` | `hsl(350 84% 57%)` | Errors, delete, expense amounts |
| `profit` (custom) | `hsl(158 100% 42%)` | Income amounts, positive values |

### Typography
- **Display / Headings**: `font-display` → Syne (geometric, distinctive)
- **Body**: DM Sans (clean, readable)
- **Numbers / Monospace**: `font-mono` → JetBrains Mono (all currency, dates, codes)
- **Section labels**: always `text-xs uppercase tracking-widest text-muted-foreground font-medium`
- **Page titles**: `font-display text-2xl font-bold text-foreground`
- **Card titles**: `font-display text-sm font-semibold text-foreground`

### Component Patterns

**Modals** — always use this structure:
```tsx
<div className="fixed inset-0 z-50 bg-black/70 backdrop-blur-sm flex items-center justify-center p-4" onClick={onClose}>
  <div className="bg-card border border-border rounded-2xl p-6 w-full max-w-md shadow-2xl animate-fade-up" onClick={(e) => e.stopPropagation()}>
    <h3 className="font-display text-lg font-semibold text-foreground mb-5">Title</h3>
    {/* form fields */}
    <div className="flex gap-3 justify-end pt-1">
      <Button variant="outline" size="sm">Cancel</Button>
      <Button size="sm">Save</Button>
    </div>
  </div>
</div>
```

**Form fields** — always label + input stacked:
```tsx
<div className="space-y-1.5">
  <label className="text-xs uppercase tracking-widest text-muted-foreground font-medium">Field Name</label>
  <Input value={...} onChange={...} placeholder="..." />
</div>
```

**Native `<select>`** — use this class string (not the shadcn Select component):
```
className="flex h-9 w-full rounded-lg border border-border bg-secondary px-3 py-1 text-sm text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
```

**Tables** — dark headers, divide-y rows, hover effect:
```tsx
<table className="w-full">
  <thead>
    <tr className="border-b border-border">
      <th className="px-5 py-3 text-left text-xs font-medium uppercase tracking-widest text-muted-foreground">Col</th>
    </tr>
  </thead>
  <tbody className="divide-y divide-border">
    <tr className="txn-row">{/* .txn-row has hover:bg-secondary/30 */}</tr>
  </tbody>
</table>
```

**Currency amounts**:
- Income / positive: `text-profit font-mono font-semibold` with `+` prefix
- Expense / negative: `text-destructive font-mono font-semibold` with `−` prefix (use `−` not `-`)
- Neutral: `text-foreground font-mono font-semibold`

**Error messages**:
```tsx
<div className="p-4 rounded-xl bg-destructive/10 border border-destructive/20 text-sm text-destructive">
  {error}
</div>
```

**Loading spinner**:
```tsx
<div className="flex items-center gap-3 text-muted-foreground">
  <div className="w-5 h-5 border-2 border-border border-t-primary rounded-full animate-spin" />
  <span className="text-sm">Loading…</span>
</div>
```

**Empty states**:
```tsx
<div className="flex flex-col items-center justify-center py-16 gap-3">
  <div className="w-12 h-12 rounded-2xl bg-muted flex items-center justify-center">
    <IconName className="w-6 h-6 text-muted-foreground" />
  </div>
  <p className="text-sm text-muted-foreground">Descriptive empty message</p>
</div>
```

### Animation
- Page sections: `animate-fade-up` on first element, `animate-fade-up-1` through `animate-fade-up-5` for staggered children
- Modals always get `animate-fade-up`
- Defined in `src/index.css` as `@keyframes fade-up`

### Rules
1. **No inline styles** — all styling via Tailwind classes. Inline `style` is only acceptable for dynamic values like `backgroundColor: category.color` (user-defined colors).
2. **No light surfaces** — never use `bg-white`, `bg-gray-*`, or any light background inside the app shell.
3. **Use shadcn components** for interactive elements: `<Button>`, `<Input>`, `<Badge>` from `src/components/ui/`. Use native `<select>` with the styled class above (not the Radix Select component — it's harder to style for dark mode).
4. **Icons from lucide-react only** — consistent weight and style.
5. **Tailwind v3** — project uses Tailwind 3.4.x with a JS config (`tailwind.config.js`). Do NOT upgrade to v4 (it uses a different PostCSS plugin and CSS-based config).

## Common Issues

**"Maximum call stack size exceeded" on Analytics page (Sankey)**:
- Almost always caused by a cycle in the graph data — an income and expense category sharing the same name creates NodeX → Cash Flow → NodeX
- Fix is in `sankey_service.py`: income uncategorized = "Uncategorized Income"; expense nodes that collide get " (Expenses)" suffix
- Frontend also filters out links where source===target or value<=0

**"Column does not exist" database errors**:
- Check migration status: `docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "SELECT * FROM schema_migrations;"`
- Restart backend to re-run migrations: `docker-compose restart backend`

**Splitwise "tuple object has no attribute getid"**:
- SDK `createExpense()` returns `(Expense, Errors)` tuple — use `isinstance(result, tuple)` check

**Splitwise "A person was included on this expense multiple times"**:
- Skip current user when iterating participants from group members

**Splitwise "You cannot add an expense that does not involve yourself"**:
- Current user must be in the participants list with `paid_share = txn_amount`

**AmbiguousForeignKeysError on Transaction joins**:
- Use explicit onclause: `.join(Account, Transaction.account_id == Account.id)`

**"No accounts found" when importing**:
- Create an account first using the "+ Create Account" button on dashboard

**Frontend shows "Network Error"**:
- Verify backend is running: `curl http://localhost:8000/health`
- Check CORS settings in `app/main.py`
- Verify VITE_API_URL in frontend `.env`
