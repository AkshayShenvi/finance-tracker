# Finance Tracker Features

This document tracks the planned and implemented features of the Finance Tracker application.

## Current Status: Phases 1-5 Complete ✅

**Last Updated:** February 18, 2026

---

## Completed Features

### Phase 1: Multi-Bank CSV Import - Backend ✅ (COMPLETED)

#### **Backend: Strategy Pattern for CSV Parsers** ✅
- [x] Base parser architecture with abstract class (`CSVParser`)
- [x] Parser registry for auto-detection (`ParserRegistry`)
- [x] Utility functions for common parsing operations
- [x] Individual parser implementations (2 total):
  - [x] American Express Credit Card (`AmexParser`)
  - [x] Discover Bank/Card (`DiscoverBankParser`)
- [x] Import service orchestrator (`CSVImportService`)
- [x] Legacy compatibility layer for existing AMEX code

#### **Database Schema Additions** ✅
- [x] `import_sessions` table - Track CSV import history
- [x] `category_rules` table - Auto-categorization rules
- [x] `bank_parser_templates` table - Custom parser configurations
- [x] `transactions.import_session_id` field - Link to import session
- [x] `transactions.raw_data` field (JSONB) - Store original CSV data
- [x] `accounts.bank_name` field - Bank identifier
- [x] `accounts.account_number_last4` field - For transfer matching
- [x] Migration script: `migrations/001_add_import_tracking.sql`

#### **Extended API Endpoints** ✅
- [x] `GET /api/v1/parsers` - List available parsers
- [x] `POST /api/v1/parsers/detect` - Auto-detect parser from CSV
- [x] `POST /api/v1/transactions/import-csv/preview` - Preview without saving
- [x] `POST /api/v1/transactions/import-csv/confirm` - Execute import
- [x] `POST /api/v1/transactions/import-csv` - Legacy AMEX import (maintained)

### Phase 2: Category Intelligence & Transfer Detection ✅ (COMPLETED)

#### **Category Intelligence** ✅
- [x] Auto-suggestion service (`CategorySuggester`)
  - [x] User-defined rules with pattern types (exact, keyword, regex)
  - [x] Priority-based rule ordering
  - [x] Built-in patterns for 9 common categories:
    - Grocery (Walmart, Target, Safeway, etc.)
    - Dining (Restaurants, DoorDash, Uber Eats, etc.)
    - Gas (Shell, Chevron, Exxon, etc.)
    - Transport (Uber, Lyft, Parking, etc.)
    - Utilities (Electric, Internet, Phone, etc.)
    - Entertainment (Netflix, Spotify, Movies, etc.)
    - Shopping (Amazon, eBay, Best Buy, etc.)
    - Health (Pharmacy, Doctor, Gym, etc.)
    - Travel (Airlines, Hotels, Car Rental, etc.)
- [x] Category rules API endpoints:
  - [x] `GET /api/v1/categories` - List user's categories
  - [x] `GET /api/v1/categories/rules` - List rules
  - [x] `POST /api/v1/categories/rules` - Create rule
  - [x] `PATCH /api/v1/categories/rules/{id}` - Update rule
  - [x] `DELETE /api/v1/categories/rules/{id}` - Delete rule

#### **Transfer Detection** ✅
- [x] Transfer detector service (`TransferDetector`)
- [x] Multi-strategy detection:
  - [x] Keyword-based (transfer, payment, ACH, Zelle, etc.)
  - [x] Account name matching
  - [x] Account last 4 digit matching
  - [x] Matching opposite transactions (same date, amount, opposite type)
- [x] Confidence scoring for transfer candidates
- [x] Integration with import preview

### Phase 3: Frontend - Multi-Step CSV Import ✅ (COMPLETED)

#### **Multi-Step Import Wizard** ✅
- [x] 4-step wizard interface with visual progress indicator
- [x] **Step 1: Upload & Detection**
  - [x] Account selection dropdown
  - [x] File upload with validation
  - [x] Skip duplicates checkbox
- [x] **Step 2: Preview**
  - [x] Parser detection results with confidence score
  - [x] Transaction preview table
  - [x] Statistics summary (total, duplicates, transfers)
- [x] **Step 3: Categorize**
  - [x] Individual category assignment per transaction
  - [x] Bulk selection and actions
  - [x] Suggested categories highlighted
- [x] **Step 4: Confirm**
  - [x] Import summary with statistics
  - [x] New categories created list
  - [x] Error reporting

#### **React Components** ✅
- [x] `CSVUploadModalNew.tsx` - Main wizard orchestrator
- [x] `TransactionPreviewTable.tsx` - Transaction table with features:
  - [x] Pagination (50 rows per page)
  - [x] Duplicate highlighting (yellow background)
  - [x] Transfer candidate indicators (blue badge)
  - [x] Bulk selection with checkboxes
  - [x] Summary statistics (total, duplicates, transfers)
  - [x] Type badges (income/expense with colors)
  - [x] Amount formatting with +/- indicators
- [x] `CategorySelector.tsx` - Dropdown with features:
  - [x] Search/filter functionality
  - [x] Color preview circles
  - [x] Suggested category badge
  - [x] Keyboard navigation
  - [x] Click-outside to close

#### **TypeScript & API Integration** ✅
- [x] Type definitions in `src/types/csv.ts`
- [x] Updated `src/services/api.ts` with new endpoints
- [x] Type-safe API calls for all new endpoints
- [x] Dashboard integration with new modal

---

## Core Features (Previously Completed) ✅

### User Authentication ✅
- [x] User registration with email/password
- [x] JWT-based authentication
- [x] Login/logout functionality
- [x] Protected routes
- [x] Password hashing with bcrypt

### Account Management ✅
- [x] Create accounts (checking, savings, credit card, investment, cash, other)
- [x] View account list
- [x] Track account balances (initial and current)
- [x] Multi-currency support (USD default)

### Transaction Management ✅
- [x] Transaction types (income, expense, transfer)
- [x] Category assignment
- [x] Basic duplicate detection
- [x] Transfer tracking with `transfer_to_account_id`
- [x] Transaction notes field

### Categories ✅
- [x] Category model with hierarchical support (parent_id)
- [x] Auto-create categories from CSV import
- [x] Category colors (hex codes)
- [x] Category icons

### Basic UI ✅
- [x] Dashboard page with statistics cards
- [x] Login/Register pages
- [x] CSV import button on dashboard
- [x] Protected routes with redirect

### Phase 4: Sankey Diagram Visualization ✅ (COMPLETED)

#### **Backend: Sankey Service** ✅
- [x] `SankeyService` class for data aggregation
  - [x] Income → Accounts flow calculation
  - [x] Accounts → Expense Categories flow calculation
  - [x] Account → Account transfers (optional)
- [x] Analytics API endpoint
  - [x] `GET /api/v1/analytics/sankey`
  - [x] Query parameters: start_date, end_date, include_transfers
  - [x] Response with nodes and links structure
  - [x] Summary statistics (total income, expenses, net savings)

#### **Frontend: Analytics Page** ✅
- [x] Sankey chart component using Recharts
  - [x] Nodes with category colors
  - [x] Links with proportional thickness
  - [x] Hover tooltips with amounts
  - [x] Legend showing node types
- [x] Analytics page layout (`/analytics`)
  - [x] Date range picker with presets (Last 7/30/90/365 Days)
  - [x] Include transfers toggle
  - [x] Summary cards (Total Income, Total Expenses, Net Savings)
- [x] Navigation integration
  - [x] Add Analytics button to Dashboard
  - [x] Add route in App.tsx

### Phase 5: Polish & Advanced Features ✅ (COMPLETED)

#### **Import History** ✅
- [x] Import history page (`/import-history`)
  - [x] List import sessions with statistics
  - [x] View transactions from specific import
  - [x] Session details with status badges
  - [x] Transaction breakdown by import
- [x] API endpoints
  - [x] `GET /api/v1/transactions/import-sessions`
  - [x] `GET /api/v1/transactions/import-sessions/{id}/transactions`

#### **Category Rules Management** ✅
- [x] Category rules UI page (`/category-rules`)
  - [x] List all rules with edit/delete actions
  - [x] Create new rule modal
  - [x] Pattern type selector (keyword/exact/regex)
  - [x] Priority input
  - [x] Active/inactive toggle
  - [x] Help section explaining rule types
- [x] Full CRUD operations
  - [x] Create, read, update, delete rules
  - [x] Toggle rule activation status

---

## In Progress

_No features currently in progress._

---

## Planned Features

### Phase 6: Testing & Additional Features 📋 (FUTURE)

#### **Custom Parser Templates** 📋
- [ ] Parser template creation page
  - [ ] Upload sample CSV
  - [ ] Column mapping interface (drag & drop)
  - [ ] Date format selector
  - [ ] Amount convention selector
  - [ ] Test parser with sample file
- [ ] Template management
  - [ ] Save templates
  - [ ] Edit existing templates
  - [ ] Use custom templates in import flow

#### **Custom Parser Templates** 📋
- [ ] Parser template creation page
  - [ ] Upload sample CSV
  - [ ] Column mapping interface (drag & drop)
  - [ ] Date format selector
  - [ ] Amount convention selector
  - [ ] Test parser with sample file
- [ ] Template management
  - [ ] Save templates
  - [ ] Edit existing templates
  - [ ] Use custom templates in import flow

#### **Testing & Documentation** 📋
- [ ] Backend tests
  - [ ] Unit tests for all parsers
  - [ ] Integration tests for import flow
  - [ ] Category suggester tests
  - [ ] Transfer detector tests
- [ ] Frontend tests
  - [ ] Component tests (Vitest)
  - [ ] E2E tests (Playwright)
- [ ] Documentation
  - [ ] API documentation (OpenAPI/Swagger)
  - [ ] User guide for CSV import
  - [ ] Parser development guide
  - [ ] Deployment guide

---

## Future Enhancements (Out of Scope)

### Intelligence & Automation 🔮
- AI-powered categorization using machine learning
- Spending anomaly detection
- Bill payment reminders
- Subscription detection and tracking
- Predictive spending forecasts

### Integrations 🔮
- Bank API integration (Plaid/Yodlee)
- Receipt OCR and matching
- Email parsing for receipts/bills
- Calendar integration for bill reminders
- Webhook notifications

### Advanced Features 🔮
- Budget tracking and enforcement
- Financial goal tracking
- Debt payoff planning
- Investment portfolio tracking
- Multi-currency with real-time forex
- Recurring transaction detection
- Split transactions
- Tags and advanced filtering

### Collaboration 🔮
- Shared accounts (household)
- Multi-user access levels
- Activity audit trail
- Comments on transactions
- Shared category rules marketplace

### Mobile 🔮
- React Native mobile app
- Biometric authentication
- Receipt camera capture
- Push notifications
- Offline mode
- Home screen widgets

---

## Technical Debt & Improvements

### Backend ⚠️
- [ ] Add database indexes for performance
  - [ ] `transactions.transaction_date`
  - [ ] `transactions.transaction_type`
  - [ ] `transactions.category_id`
- [ ] Implement caching layer (Redis)
- [ ] Add background job processing (Celery)
- [ ] API rate limiting
- [ ] OpenAPI/Swagger documentation
- [ ] Error tracking (Sentry)

### Frontend ⚠️
- [ ] Implement proper error boundaries
- [ ] Add loading states for all async operations
- [ ] Code splitting and lazy loading
- [ ] Service worker for offline support
- [ ] Performance monitoring
- [ ] Enable TypeScript strict mode
- [ ] Add toast notifications library
- [ ] Implement proper form validation (Zod)

### DevOps ⚠️
- [ ] Docker production-ready setup
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Automated database migrations in CI
- [ ] Environment management
- [ ] Monitoring and logging (Prometheus/Grafana)
- [ ] Load testing
- [ ] Security scanning

---

## Statistics

- **Total Features Planned:** 200+
- **Completed:** ~75 (Phases 1-5)
- **In Progress:** 0
- **Next Up:** Custom Parser Templates & Testing
- **Future:** 125+

---

## Legend
- ✅ Completed
- 🚧 In Progress
- 📋 Planned (Next Sprint)
- ⚠️ Technical Debt
- 🔮 Future Enhancement (Backlog)
- ❌ Cancelled

---

**Project Status:** On Track
**Current Phase:** Phases 1-5 Complete
**Last Major Update:** Completed Phase 4 (Sankey Visualization) & Phase 5 (Import History + Category Rules)
