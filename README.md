# ZESTORA

ZESTORA is a responsive e-commerce storefront with a server-backed administrator catalog. The public site and admin tools use the same PostgreSQL product, category, and homepage-content records.

## Requirements

- Node.js 20.19+, 22.12+, or 24+
- PostgreSQL 14+ with permission to create tables and indexes
- npm

## Local setup

1. Install the locked dependencies:

   ```powershell
   npm install
   ```

2. Create a PostgreSQL database and login. For example, in `psql`:

   ```sql
   CREATE ROLE zestora_app WITH LOGIN PASSWORD 'use-a-unique-local-password';
   CREATE DATABASE zestora OWNER zestora_app;
   ```

3. Create `.env` by copying `.env.example`, then set `DATABASE_URL` to the connection string for your PostgreSQL database. Generate a private session secret:

   ```powershell
   Copy-Item .env.example .env
   node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"
   ```

   Put the generated value in `SESSION_SECRET`. Do not commit `.env`, publish the database password, or reuse a production secret for local development. For production behind HTTPS, set `NODE_ENV=production` and configure `TRUST_PROXY=true` only when requests pass through a trusted reverse proxy; secure session cookies are enabled automatically in production.

4. Generate the Prisma client, apply migrations, and load the starter catalog:

   ```powershell
   npm run db:generate
   npm run db:migrate
   npm run db:seed
   ```

   Seeding inserts the demonstration products and categories only when they are absent. Re-running it does not overwrite catalog edits.

5. Create the first administrator interactively. The password is entered without echo and must be 12–128 characters:

   ```powershell
   npm run admin:create
   ```

   Admin creation is command-line only; there is no public admin signup endpoint.

6. Start the application:

   ```powershell
   npm start
   ```

   Open <http://localhost:3000/> for the storefront or <http://localhost:3000/admin/login> for administrator sign-in.

## Admin capabilities

- Server-side admin authentication, bcrypt password hashing, PostgreSQL-backed sessions, CSRF tokens, same-origin checks, login rate limiting, and protected dashboard/API routes
- Product creation, editing, search, category filtering, activation, stock, featured/trending/deal flags, and deletion
- JPEG, PNG, and WebP image uploads with a 5 MB limit, content-type signature verification, previews, and persistent files under `uploads/`
- Category creation, editing, image upload, ordering, activation, and deletion safeguards
- Homepage headline, description, and image editing
- Store summary counts and stock/deal views

Uploaded images are stored on the application server's local disk. For a multi-server or hosted production deployment, configure persistent shared storage and a backup policy for `uploads/`.

## API and security notes

- Public product and category data is served by `GET /api/catalog`.
- All write operations are protected under `/api/admin` and require an authenticated admin session plus the session CSRF token.
- The administrator dashboard and its JavaScript are protected by the server; client-side state is not used as proof of admin access.
- Customer email/password registration and login use bcrypt password hashes, PostgreSQL-backed sessions, CSRF and same-origin checks, rate limits, and server-side authorization. Customer sessions expire after eight hours of inactivity.
- Customer profile editing, saved addresses, account password changes, and a server-synced wishlist are available from `/customer/dashboard`.
- Customer orders are not available because checkout and order processing are not implemented. Customer login is email-only; mobile numbers are collected and uniquely stored but are not verified.
- Google/Facebook sign-in and SMS OTP are not enabled. To add social sign-in, configure provider-issued OAuth client IDs and secrets, register exact callback URLs with Google/Facebook, and implement backend state/nonce and token validation before enabling the buttons. SMS OTP additionally requires an account with a real SMS provider; store its API credentials server-side, configure sender/callback settings, and never return OTP values from the API. No provider credentials or fake OTP mode are included.
- Configure TLS at the production reverse proxy. Never expose PostgreSQL to the public internet.

### Customer account setup and integration status

After configuring PostgreSQL, apply the customer account migration with `npm run db:migrate` and start the application with `npm start`. Customer accounts store a bcrypt hash, normalized unique email and mobile values, optional date of birth, and the timestamp at which terms were accepted. Passwords must be 12–72 UTF-8 bytes because bcrypt only processes the first 72 bytes.

Customer pages and APIs:

- `/login` and `/signup` show the existing customer auth design in the requested mode.
- `/customer/dashboard` requires a valid customer session.
- `POST /api/customer/register` and `POST /api/customer/login` create accounts and sign in.
- `GET /api/customer/session` reports the authenticated customer profile and CSRF token.
- `POST /api/customer/logout` invalidates the session.
- Profile, saved address, wishlist, and password endpoints require that customer session and CSRF protection.

Customer sessions use the PostgreSQL-backed session store, an HTTP-only same-site cookie, and an eight-hour idle expiration. The registration endpoint requires explicit terms acceptance. Mobile number is not verified without a configured SMS provider. No order model or checkout exists yet, so the account page explains that order history is unavailable rather than fabricating orders.

#### Social sign-in and SMS OTP (not active)

There are no OAuth callback endpoints or SMS verification endpoints configured in this project yet. Do not add provider secrets to frontend files or enable the social buttons as if they authenticate. To prepare Google sign-in, create a Google OAuth web client, configure its allowed origin and redirect URI to `https://YOUR_DOMAIN/auth/google/callback`, then implement the server callback and verify the provider ID token, state, issuer, audience, and nonce. Facebook requires a Facebook Login app with its app ID/secret and the corresponding `https://YOUR_DOMAIN/auth/facebook/callback` redirect URI and server-side token validation. Those callback URLs are setup targets only; they are not currently implemented routes.

Phone OTP needs a real SMS/verification provider account (typically paid), sender/service configuration, and server-side credentials—for example a provider account SID, auth token, and verification-service ID when using Twilio Verify. Implement expiring, single-use codes, per-phone/IP send limits, attempt limits, and server-only provider calls before treating mobile numbers as verified. No provider or OTP configuration is currently active.

## Tests

Run the focused server-side validation tests:

```powershell
npm test
```

The application needs a configured, reachable PostgreSQL database to start, migrate, seed, create an administrator, and run end-to-end admin/product tests.
