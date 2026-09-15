# eventkart backend

A REST API for the event coordinator marketplace: customers post requirements, vendors send quotes, customers book and pay into escrow, vendors deliver and get paid out, and admins approve vendors and handle disputes.

Built with **Node.js + Express + Prisma**, using **SQLite** so it runs locally with no external database setup. Swap the `DATABASE_URL` in `.env` for a Postgres connection string when you're ready for production — Prisma handles the rest.

## Setup

```bash
cd backend
npm install
cp .env.example .env        # then edit JWT_SECRET to something random
npx prisma migrate dev --name init
npm run seed                # creates an admin login: admin@eventkart.com / admin123
npm run dev                 # starts the API on http://localhost:4000
```

Run `npx prisma studio` any time to browse/edit the database in a GUI.

## How the pieces map to the frontend

| Frontend screen | Endpoints it needs |
|---|---|
| Homepage / search results | `GET /api/vendors` |
| Vendor profile page | `GET /api/vendors/:id`, `GET /api/reviews/vendor/:vendorId` |
| Post requirement form | `POST /api/requirements` |
| Compare quotes page | `GET /api/requirements/:id` (includes quotes), `POST /api/quotes/:id/accept` |
| Checkout / booking page | `POST /api/payments/:bookingId/pay` |
| Customer bookings dashboard | `GET /api/bookings/mine`, `POST /api/reviews` |
| Vendor dashboard | `GET /api/requirements/open`, `POST /api/quotes`, `GET /api/bookings/mine`, `POST /api/bookings/:id/complete` |
| Admin panel | `GET /api/admin/stats`, `GET /api/admin/vendors/pending`, `POST /api/admin/vendors/:id/approve`, `GET /api/admin/bookings/disputed` |

## Full booking lifecycle (how the pieces connect)

1. **Customer** registers (`POST /api/auth/register`, role `CUSTOMER`) and posts a requirement (`POST /api/requirements`).
2. **Vendor** registers (role `VENDOR`), fills in their profile (`PUT /api/vendors/me`), and waits for admin approval.
3. **Admin** approves the vendor (`POST /api/admin/vendors/:id/approve`).
4. **Vendor** sees the open requirement (`GET /api/requirements/open`) and sends a quote (`POST /api/quotes`).
5. **Customer** views quotes on their requirement (`GET /api/requirements/:id`) and accepts one (`POST /api/quotes/:id/accept`) — this creates a `Booking` in `PENDING_PAYMENT`.
6. **Customer** pays (`POST /api/payments/:bookingId/pay`) — funds are recorded as `HELD` (escrow) and the booking becomes `CONFIRMED`.
7. **Vendor** delivers the event and marks it done (`POST /api/bookings/:id/complete`) — booking becomes `COMPLETED`.
8. **Admin** releases the escrowed funds to the vendor (`POST /api/payments/:bookingId/release`).
9. **Customer** leaves a review (`POST /api/reviews`) — this automatically recalculates the vendor's average rating.

If something goes wrong at any point after payment, either side can raise a dispute (`POST /api/bookings/:id/dispute`), and the admin resolves it by releasing or refunding the payment.

## Authentication

Every protected route expects:
```
Authorization: Bearer <token>
```
The token comes back from `POST /api/auth/register` or `POST /api/auth/login`, and encodes the user's `id` and `role` (`CUSTOMER`, `VENDOR`, or `ADMIN`). There's no public admin signup — create admin accounts via the seed script or directly in the database.

## Commission

The commission rate is set in `.env` as `COMMISSION_RATE` (default `0.12`, i.e. 12%). It's applied when a quote is accepted: `totalAmount = quote price + commission`. Adjust this per-vendor or per-category later by moving the rate onto the `VendorProfile` model if you need more granular pricing.

## Important before going live

- **Payments are simulated.** `POST /api/payments/:bookingId/pay` immediately marks funds as held — it does not call a real payment gateway. Before launch, integrate a provider that supports split/held payments (Razorpay Route, Stripe Connect) and only mark a payment `HELD` after the gateway's webhook confirms the charge.
- **Switch to Postgres** for production — SQLite is great for local development but not for concurrent production traffic.
- **Add rate limiting and input sanitization** on public endpoints before exposing this to the internet.
- **Rotate `JWT_SECRET`** and never commit `.env` to version control.
