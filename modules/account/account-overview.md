# Account Module Overview

**Module ID**: `account`  
**Scope**: Core stories for viewing and updating cardholder accounts  
**Location**: `/app/pages/AccountViewPage.tsx`, `/app/pages/AccountUpdatePage.tsx`

## Purpose

Provide accurate, compliant visibility of a customer’s financial profile and a guarded channel to adjust account/customer data. The experiences align with the CardDemo UX system (SystemHeader, responsive Grid, chips) and share common hooks (`useAccountView`, `useAccountUpdate`) that encapsulate API, validation and spinner logic.

## Key Capabilities

1. Account lookup by 11-digit ID with inline validation, helper text and `Search` button (uses MUI `TextField`, `InputAdornment`, `Button`).
2. Data masking toggle for SSN/card numbers plus `Show Test Accounts` panel available in `import.meta.env.DEV`.
3. Financial + customer + contact dashboards composed of MUI `Card`, `Stack`, `Grid`, `Chip` and icons from `@mui/icons-material`.
4. Update workflow with `Edit Mode` toggle, `Switch`, dropdowns (`MenuItem`), local validation (ZIP regex, numeric checks), unsaved-change chips and confirmation dialog (`Dialog`).
5. Keyboard shortcuts: `F3/Escape` to exit (calls `handleExit`), `F5` to trigger `handleUpdate` when dirty, `F12` to reset the form/state.
6. API hooks that transparently handle both MSW `ApiResponse` wrappers and real backend responses (`success`/`errorMessage`).

## API Contracts

- `GET /account-view?accountId={accountId}` → Returns `AccountViewResponse` with metadata, cards, balances, SSN and card number.
- `GET /account-view/initialize` → Bootstraps the viewer with dates and `transactionId`.
- `GET /accounts/{accountId}` → Retrieves editable `AccountUpdateData` (account + customer + contact fields).
- `PUT /accounts/{accountId}` → Persists `AccountUpdateSubmission`; responds with `AccountUpdateResponse` (success flag + optional errors).

## Data Models

- `AccountViewResponse` (currentDate/time, transactionId, accountStatus, balance/limit, customer info, cardNumber, messages).
- `AccountUpdateData` (activeStatus, balance numbers, open/expiration/reissue dates, customer names, SSN, FICO, contact details, zip/state/country codes).
- `AccountUpdateResponse` (success, data, errors) for optimistic UI updates.

## Dependencies & Patterns

- Shared hooks: `useMutation` (handles retries, aborts, MSW vs backend responses) and `apiClient`.
- UI building blocks: Material-UI components, `SystemHeader`, `LoadingSpinner`, `Alert`.
- Routing/auth: Pages read `localStorage.userRole`, push to `/login`/`/menu/*`, so coverage is tied to `react-router-dom`.
- Test fixtures: `app/mocks/accountHandlers.ts` contains handlers for every API plus DEV-only scenarios (`test-accounts`, `test-error`).
- Validation helpers: `formatCurrency`, `formatDate`, `formatSSN`, `formatCardNumber` (local functions in screens).

## Business Rules

1. Account ID must be 11 digits and non-zero before any API call.
2. Only `activeStatus` = `Y` accounts are editable; statuses display `Chip` color-coded.
3. Financial limits/values respect positive checks (credit limit, cash credit), and `availableCredit` is implied as `creditLimit - currentBalance`.
4. SSN/card numbers default to masked format; toggling reveals full values for authorized viewers.
5. ZIP codes follow `/^\d{5}(-\d{4})?$/`; FICO score is displayed with tiered colors (>=750 green, >=650 amber, else red).

## User Story Guidance

### Sample Template
As [back-office agent], I want to search by account ID and surface the latest financial snapshot so I can respond to customer inquiries.

### Complexity Ranges
- Simple (1-2 pts): Add a read-only field into `AccountViewScreen` or adjust helper copy.
- Medium (3-5 pts): Introduce new validation (e.g., SSN formatting) or add a derived card/payer chip.
- Complex (5-8 pts): Integrate external scoring service, add audit trail, or extend update workflow to multi-step reviews.

## Performance / Readiness

- View requests aim for <500ms (P95) because dashboards drive call center efficiency.
- Updates should complete in <1s (P95) and integrate `LoadingSpinner` + `Dialog`.
- Risk: no optimistic locking (`@Version` on entities) and no i18n – align with future tech debt backlog.

## References

- `app/components/account/AccountViewScreen.tsx`  
- `app/components/account/AccountUpdateScreen.tsx`  
- `app/hooks/useAccountView.ts`  
- `app/hooks/useAccountUpdate.ts`  
- `app/hooks/useApi.ts`  
- `app/mocks/accountHandlers.ts`
