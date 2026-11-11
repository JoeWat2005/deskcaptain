> [!WARNING]
> **Deprecated: no longer maintained.** Last updated in April 2026; kept for reference.

# DeskCaptain

AI receptionist software for small businesses. It captures and qualifies leads from web chat and phone calls, takes bookings, and gives each organisation a dashboard for leads, conversations, bookings and billing.

## Features

- **AI web chat.** Each organisation's public landing page (`/<slug>/landing`) includes a chat widget. An OpenAI model collects the visitor's name, phone number, intent and preferred callback window.
- **Phone calls.** A webhook at `/api/reception/voice/webhook` receives call events from Vapi. It records the lead, transcript and recording details, and chooses between transferring the call and arranging a callback based on business hours and the configured transfer number.
- **Lead qualification and alerts.** A lead counts as qualified once it has a name, phone number and intent. Qualified leads trigger an email alert through Resend.
- **Bookings.** Staff profiles synced from organisation members, staff availability and booking settings, slot search, a public booking form, and automatic assignment to the first free slot.
- **Team dashboard.** At `/<slug>/dashboard`, with overview, analytics, bookings, chats, leads, enquiries, links page, landing page styling, organisation members, notifications, audit log, settings, billing and invoices.
- **Public pages.** Each organisation gets a customisable landing page and a links page (`/<slug>/links`).
- **Accounts and billing through Clerk.** Clerk handles sign-in, organisations, invitations and subscriptions. Clerk webhooks keep users, organisations, memberships, subscriptions and payment attempts in sync with the database. The phone receptionist and the receptionist settings and leads APIs require a paid plan.
- **Accessibility.** High-contrast, reduced-motion and large-text preferences, applied before the page renders.
- **Analytics.** Optional Google Analytics.

## Tech stack

- Next.js 16.2 (App Router), React 19.2 and TypeScript 5
- Prisma 7 with the PostgreSQL driver adapter (`@prisma/adapter-pg`)
- PostgreSQL
- Clerk (`@clerk/nextjs` 7) for authentication, organisations and billing, with Svix for webhook verification
- OpenAI chat completions API, called over HTTP
- Tailwind CSS 4, daisyUI 5 and shadcn/ui components built on Base UI
- ApexCharts, GSAP, TanStack Table, Sonner and lucide-react
- Resend for email (through its HTTP API) and Vapi for voice (through a webhook)
- pytest and Playwright for integration tests

## Getting started

### Prerequisites

- Node.js and npm. No engine version is pinned in `package.json`.
- A PostgreSQL database
- A Clerk application with organisations enabled
- An OpenAI API key for the web chat
- Optional: a Resend API key for lead alerts, a Vapi account for phone calls and a Google Analytics measurement ID
- Python 3 and pip, to run the integration tests

### Installation

```bash
npm install
```

### Configuration

Create a `.env` file in the project root. Files matching `.env*` are ignored by git. Next.js reads it for the app, and `prisma.config.ts` and `prisma/seed.ts` load it through dotenv.

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | PostgreSQL connection string used by Prisma and the app |
| `CLERK_WEBHOOK_SIGNING_SECRET` | Signing secret for the Clerk webhook endpoint, `/api/webhooks/clerk` |
| `OPENAI_API_KEY` | OpenAI API key for the web chat receptionist |
| `OPENAI_MODEL` | Model used for web chat (default `gpt-4.1-mini`) |
| `RECEPTION_LLM_PROVIDER` | Chat provider. Only `openai`, the default, is supported. |
| `RECEPTION_DEFAULT_TIMEZONE` | Fallback time zone (default `Europe/London`) |
| `EMAIL_PROVIDER_API_KEY` | Resend API key for qualified-lead emails. If it is unset, emails are skipped with a warning. |
| `NOTIFICATION_FROM_EMAIL` | Sender address for qualified-lead emails |
| `VAPI_WEBHOOK_SECRET` | Shared secret expected on the voice webhook, sent as `x-vapi-secret`, `x-webhook-secret` or a bearer token |
| `INTERNAL_BACKFILL_TOKEN` | Token for `POST /api/internal/backfill-memberships`, sent as `x-internal-backfill-token` or a bearer token |
| `NEXT_PUBLIC_GA_MEASUREMENT_ID` | Optional Google Analytics measurement ID |
| `SEED_ORG_ID` | Organisation to seed when `npm run seed` gets no `--orgId` argument |

Clerk's publishable and secret keys are also required. `@clerk/nextjs` reads them from the environment, but this repository does not reference them by name, so use the variable names from Clerk's Next.js setup instructions.

### Database

```bash
npm run prisma:migrate    # prisma migrate dev: apply the migrations in prisma/migrations
npm run prisma:generate   # generate the Prisma client into generated/prisma
```

`npm run dev` and `npm run build` also run `prisma generate` first, through the `predev` and `prebuild` scripts. `npm run prisma:studio` opens Prisma Studio.

### Clerk webhooks and demo data

Users, organisations and memberships reach the database through the Clerk webhook. Clerk must be able to reach `POST /api/webhooks/clerk`, and the webhook needs to send the events listed under [Clerk sync](#clerk-sync).

To add demo data to an organisation that already exists in the database, pass its database ID. This is `Organization.id`, not the Clerk organisation ID. The seed adds contacts, leads, conversations, bookings, links, notifications and audit logs.

```bash
SEED_ORG_ID=<organisation id> npm run seed
# or
npx prisma db seed -- --orgId=<organisation id>
```

`npm run seed:demo` uses a fixed organisation ID from the original development database.

### Running locally

```bash
npm run dev
```

The app runs at `http://localhost:3000`, the address the integration tests expect.

### Tests

The integration tests are written in Python with pytest and Playwright, and drive the app in headless Chromium.

```bash
pip install -r requirements-test.txt
python -m playwright install chromium
pytest
```

`tests/conftest.py` starts `npm run dev` if nothing is listening on `http://localhost:3000`. Tests that need an organisation read the first slug from the `/stats` page, and are skipped when the database has none.

### Linting and production build

```bash
npm run lint
npm run build
npm run start
```

## How it works

### Routing and access

- `proxy.ts` runs Clerk's middleware. It sends signed-out visitors from `/<slug>/dashboard`, `/dashboard` and `/settings` back to `/`.
- After sign-in, `/post-auth` sends users either to `/onboarding`, which uses Clerk's `CreateOrganization`, or to `/<orgSlug>/dashboard`.
- Dashboard pages call `requireDashboardPageOrg()` (`lib/dashboard/page-access.ts`), and dashboard APIs call `requireDashboardApiOrg()` (`lib/dashboard/access.ts`). Both map the active Clerk organisation to its database record, and the API guard can also require a paid plan.

| Public route | Purpose |
| --- | --- |
| `/` | Marketing page with features and pricing |
| `/pricing` | Redirects to `/#pricing` |
| `/sign-in`, `/sign-up` | Clerk sign-in and sign-up |
| `/onboarding` | Create an organisation |
| `/<slug>/landing` | Organisation landing page with web chat and booking form |
| `/<slug>/links` | Organisation links page |
| `/stats` | Platform summary: user, organisation and membership counts and a table of organisations. It is not behind authentication. |

### Receptionist

- **Web chat.** `components/reception/WebChatWidget.tsx` posts each message to `/api/reception/chat`. `lib/reception/chat.ts` checks business hours and asks the provider in `lib/reception/llm/` for a reply and any lead details. It then saves the conversation and lead through `conversation.ts` and `lead.ts`, and qualified leads are passed to `notify.ts`.
- **Phone.** `lib/reception/voice.ts` checks the Vapi webhook secret and finds the organisation by slug metadata or phone extension. It then records the lead, conversation, transcript and call recording.
- **Configuration.** Each organisation has a `ReceptionistConfig` holding its phone extension, notification email, business hours, FAQ script, transfer number, channel switches and time zone. The dashboard settings page edits all of these except business hours, and only for organisations on a paid plan.
- **Bookings.** `lib/bookings/service.ts` creates bookable staff profiles from organisation members, lists free slots and assigns new bookings to the first available slot within 14 days.

### Clerk sync

`app/api/webhooks/clerk/route.ts` verifies the Svix signature (`verify.ts`) and dispatches events to `app/api/webhooks/handlers/`:

- `user.created`, `user.updated`, `user.deleted`
- `organization.created`, `organization.updated`, `organization.deleted`
- `organizationMembership.created`, `organizationMembership.updated`, `organizationMembership.deleted`
- `subscription.created`, `subscription.updated`, `subscription.active`, `subscription.pastDue`
- `paymentAttempt.created`, `paymentAttempt.updated`

`lib/billing/effective-plan.ts` works out the current plan from the synced subscription items, and any plan without "free" in its name counts as paid. `POST /api/internal/backfill-memberships` re-syncs memberships from Clerk.

### Data model

`prisma/schema.prisma` targets PostgreSQL. It covers:

- users, organisations, contacts and memberships
- subscriptions and subscription items
- receptionist config, leads, conversations, messages and call recordings
- booking settings, staff profiles, staff availability and bookings
- link profiles and items, and page customisation
- notification events and audit logs

Migrations are in `prisma/migrations/`.

## Project structure

```
app/
  [slug]/          Organisation landing, links and dashboard pages
  api/             Route handlers (audit, bookings, chats, customisation, links,
                   notifications, reception, webhooks, internal)
  onboarding/, post-auth/, sign-in/, sign-up/, pricing/, stats/
                   Onboarding, auth, pricing and stats routes
components/        Dashboard, landing, reception, settings and chart components,
                   plus shadcn/ui primitives in components/ui
hooks/             React hooks
lib/               Server logic: Prisma client, reception, bookings, billing,
                   Clerk sync, dashboard access, analytics and accessibility
prisma/            Schema, migrations and seed script
public/            Static assets
tests/             Python Playwright integration tests
proxy.ts           Clerk middleware and route protection
AGENTS.md          Notes for AI coding assistants
```

## Related repositories

- [ai-receptionist](https://github.com/JoeWat2005/ai-receptionist): the early Streamlit prototype of DeskCaptain
- [flotrafic-react-frontend](https://github.com/JoeWat2005/flotrafic-react-frontend) and [flotrafic-fastapi-backend](https://github.com/JoeWat2005/flotrafic-fastapi-backend): Flotrafic, an earlier small-business product
