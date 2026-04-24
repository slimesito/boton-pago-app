# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Structure

This is a monorepo with two independent sub-projects, each with their own `CLAUDE.md`:

- `backend/` — Laravel 13 (PHP 8.3) REST API. See `backend/CLAUDE.md` for detailed Laravel guidelines.
- `frontend/` — React 19 + TypeScript + Vite SPA. See `frontend/CLAUDE.md` for frontend guidelines.

## Commands

### Backend (`cd backend`)
```bash
composer run dev        # Concurrent: Laravel server + queue + logs + Vite
composer setup          # One-time: install deps, generate key, migrate, build frontend
composer test           # Clear config cache, then run PHPUnit
php artisan test --compact                                  # Run all tests
php artisan test --compact tests/Feature/ExampleTest.php   # Run a specific file
php artisan test --compact --filter=testName               # Run a specific test
vendor/bin/pint --dirty --format agent                     # Format changed PHP files
```

### Frontend (`cd frontend`)
```bash
npm run dev       # Start Vite dev server (port 5173)
npm run build     # Type-check then build for production
npm run lint      # Run ESLint
```

## Architecture

**Boton de Pago** is a payment gateway integration that bridges a customer-facing checkout UI with the Biopago payment processor.

### Payment flow
1. Frontend collects payer info → `POST /api/payments/init` stores a record and returns a Biopago payment ID and URL.
2. `POST /api/payments/send-token` triggers an OTP delivery to the payer via Biopago.
3. Payer enters OTP → `POST /api/payments/process` calls Biopago and evaluates three approval criteria (status, result, auth code).
4. Biopago redirects to `GET /api/payments/return` (auth bypassed); backend calls `verifyPayment()` and updates the record status.
5. Frontend polls `GET /api/payments/{reference}/status` and renders `SuccessPage` or `FailurePage`.

### Backend layers
| Layer | Location | Responsibility |
|---|---|---|
| Controller | `app/Http/Controllers/PaymentController.php` | HTTP input/output, delegates to service |
| Service | `app/Services/PaymentService.php` | Business logic, approval evaluation, status mapping |
| API client | `app/Services/BiopagoApiService.php` | HTTP calls to Biopago (OAuth, retries, timeouts) |
| Auth | `app/Services/BiopagoAuthService.php` | OAuth token management for Biopago |
| Model | `app/Models/Payment.php` | Eloquent record with `approved()` / `pending()` scopes |

### Frontend pages
- `CheckoutPage` — multi-step form (payer ID → payment groups → OTP → process)
- `SuccessPage` — receipt display with transaction details
- `FailurePage` — error code + retry action

### Database
- **Production**: Oracle DB (yajra/laravel-oci8 driver)
- **Tests**: SQLite in-memory (configured in `phpunit.xml`)
- Key table: `payments` — holds `internal_reference` (unique), `biopago_payment_id`, `status`, `biopago_response` (JSON), and payer identity fields.

### Environment
Backend requires `.env` with Oracle DB credentials and Biopago API credentials (`BIOPAGO_ENV`, `BIOPAGO_BASE_URL`, `BIOPAGO_TOKEN_URL`, `BIOPAGO_CLIENT_ID`, `BIOPAGO_CLIENT_SECRET`, `FRONTEND_URL`).
