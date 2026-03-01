# BlackRoad OS

> **The operating layer for modern commerce — payments, identity, and infrastructure in one cohesive platform.**

[![npm version](https://img.shields.io/npm/v/@blackroad/core.svg?style=flat-square)](https://www.npmjs.com/org/blackroad)
[![License: Proprietary](https://img.shields.io/badge/license-Proprietary-red.svg?style=flat-square)](./LICENSE)
[![Stripe Verified Partner](https://img.shields.io/badge/Stripe-Verified%20Partner-635BFF?style=flat-square&logo=stripe&logoColor=white)](https://stripe.com)
[![Status: Production](https://img.shields.io/badge/status-production-brightgreen?style=flat-square)](https://github.com/BlackRoad-OS)

---

## Table of Contents

1. [Overview](#overview)
2. [Key Features](#key-features)
3. [Architecture](#architecture)
4. [Getting Started](#getting-started)
   - [Prerequisites](#prerequisites)
   - [Installation (npm)](#installation-npm)
   - [Environment Variables](#environment-variables)
5. [Stripe Integration](#stripe-integration)
   - [Configuration](#configuration)
   - [Accepting Payments](#accepting-payments)
   - [Webhooks](#webhooks)
   - [Testing Payments](#testing-payments)
6. [API Reference](#api-reference)
7. [End-to-End Testing (E2E)](#end-to-end-testing-e2e)
8. [Deployment](#deployment)
9. [Repository Index](#repository-index)
10. [Security & Compliance](#security--compliance)
11. [Support](#support)
12. [License](#license)

---

## Overview

**BlackRoad OS** is a production-grade commerce operating system built for teams that demand reliability, security, and speed. It provides a unified SDK and API layer that connects your application to Stripe, identity providers, cloud infrastructure, and beyond — eliminating the integration overhead that slows engineering teams down.

> **Note:** Active development has moved to the [BlackRoad-OS](https://github.com/BlackRoad-OS) GitHub organization. This repository is part of the legacy `blackboxprogramming` archive. See [Repository Index](#repository-index) for the canonical source of truth.

---

## Key Features

| Feature | Description |
|---|---|
| 💳 **Stripe-Native Payments** | One-line checkout, subscriptions, invoicing, and payout orchestration |
| 🔐 **Identity & Access** | SSO, RBAC, and audit logging out of the box |
| ⚡ **Edge-Ready API** | Sub-50 ms response times with global edge deployment |
| 📦 **npm Packages** | Modular, tree-shakeable packages published to `@blackroad` scope |
| 🔄 **Webhooks** | Reliable event delivery with automatic retries and signature verification |
| 🧪 **E2E Test Suite** | Playwright-based end-to-end tests covering all critical payment paths |
| 📊 **Observability** | Built-in structured logging, metrics, and distributed tracing |

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      Client Apps                         │
│         (Web · Mobile · CLI · Third-party Integrations)  │
└───────────────────────┬──────────────────────────────────┘
                        │  HTTPS / WebSocket
┌───────────────────────▼──────────────────────────────────┐
│                  BlackRoad OS API                         │
│              (blackroad-os-api · REST + GraphQL)          │
└──────┬────────────────┬────────────────┬─────────────────┘
       │                │                │
┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│  Payments   │  │  Identity   │  │ Prism Console│
│  (Stripe)   │  │  (Auth/SSO) │  │  (Dashboard) │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## Getting Started

### Prerequisites

- **Node.js** `>= 18.x`
- **npm** `>= 9.x` or **pnpm** `>= 8.x`
- A **Stripe** account — [sign up free](https://dashboard.stripe.com/register)
- BlackRoad OS API key (contact [support@blackroad.io](mailto:support@blackroad.io))

### Installation (npm)

Install the core SDK:

```bash
npm install @blackroad/core
```

Install optional modules as needed:

```bash
# Stripe payments module
npm install @blackroad/payments

# Identity & access control
npm install @blackroad/identity

# CLI toolchain
npm install --global @blackroad/cli
```

Initialize your project:

```bash
npx @blackroad/cli init
```

### Environment Variables

Create a `.env` file at the root of your project:

```dotenv
# BlackRoad OS
BLACKROAD_API_KEY=bro_live_...
BLACKROAD_ENV=production          # production | staging | development

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Optional — Stripe Connect (marketplace / platform)
STRIPE_ACCOUNT_ID=acct_...
```

> ⚠️ **Never commit secret keys.** Add `.env` to your `.gitignore` and use a secrets manager (AWS Secrets Manager, HashiCorp Vault, etc.) in production.

---

## Stripe Integration

BlackRoad OS ships with a first-class Stripe integration. All Stripe interactions are handled through the `@blackroad/payments` module, which wraps the official Stripe SDK with opinionated defaults, retry logic, and webhook verification built-in.

### Configuration

```typescript
import { BlackRoad } from '@blackroad/core';
import { PaymentsModule } from '@blackroad/payments';

const app = new BlackRoad({
  apiKey: process.env.BLACKROAD_API_KEY,
  modules: [
    new PaymentsModule({
      stripe: {
        secretKey: process.env.STRIPE_SECRET_KEY,
        webhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
      },
    }),
  ],
});
```

### Accepting Payments

**One-time payment (Checkout Session):**

```typescript
import { payments } from '@blackroad/payments';

const session = await payments.checkout.create({
  lineItems: [
    {
      price: 'price_1ABC...', // Stripe Price ID
      quantity: 1,
    },
  ],
  mode: 'payment',
  successUrl: 'https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}',
  cancelUrl: 'https://yourapp.com/cancel',
});

// Redirect the user
redirect(session.url);
```

**Subscriptions:**

```typescript
const session = await payments.checkout.create({
  lineItems: [{ price: 'price_monthly_...', quantity: 1 }],
  mode: 'subscription',
  successUrl: 'https://yourapp.com/dashboard',
  cancelUrl: 'https://yourapp.com/pricing',
});
```

### Webhooks

BlackRoad OS automatically verifies Stripe webhook signatures. Register your webhook handler:

```typescript
import { webhooks } from '@blackroad/payments';

// Express / Next.js API route
export async function POST(req: Request) {
  await webhooks.handle(req, {
    'checkout.session.completed': async (event) => {
      const session = event.data.object;
      await fulfillOrder(session);
    },
    'invoice.payment_failed': async (event) => {
      const invoice = event.data.object;
      // invoice.customer is a Stripe customer ID; expand or fetch the customer
      // to retrieve their email address.
      const customer = await stripe.customers.retrieve(invoice.customer as string);
      await notifyCustomer((customer as Stripe.Customer).email);
    },
  });
}
```

Register your endpoint URL in the [Stripe Dashboard](https://dashboard.stripe.com/webhooks):

```
https://yourapp.com/api/webhooks/stripe
```

### Testing Payments

Use Stripe's test mode keys (`sk_test_...`, `pk_test_...`) and test card numbers:

| Card Number | Scenario |
|---|---|
| `4242 4242 4242 4242` | Successful payment |
| `4000 0000 0000 0002` | Card declined |
| `4000 0025 0000 3155` | 3D Secure authentication required |
| `4000 0000 0000 9995` | Insufficient funds |

> Full list of test cards: [Stripe Testing Docs](https://stripe.com/docs/testing#cards)

---

## API Reference

Full API documentation is available at:

**[https://docs.blackroad.io/api](https://docs.blackroad.io/api)**

| Module | npm Package | Docs |
|---|---|---|
| Core | `@blackroad/core` | [docs/core](https://docs.blackroad.io/core) |
| Payments | `@blackroad/payments` | [docs/payments](https://docs.blackroad.io/payments) |
| Identity | `@blackroad/identity` | [docs/identity](https://docs.blackroad.io/identity) |
| CLI | `@blackroad/cli` | [docs/cli](https://docs.blackroad.io/cli) |

---

## End-to-End Testing (E2E)

BlackRoad OS ships with a Playwright-based E2E test suite covering all critical user journeys, including payment flows.

### Running E2E Tests

```bash
# Install Playwright browsers (first time only)
npx playwright install

# Run the full E2E suite
npm run test:e2e

# Run payment-specific tests only
npm run test:e2e -- --grep "payments"

# Run in headed mode (visual)
npm run test:e2e -- --headed
```

### E2E Configuration

Tests are configured in `playwright.config.ts`. The suite covers:

- ✅ Checkout flow (one-time payment)
- ✅ Subscription creation and cancellation
- ✅ Webhook delivery and processing
- ✅ Invoice generation and download
- ✅ Stripe Connect onboarding (platform flows)
- ✅ Failed payment recovery paths

> **CI/CD:** E2E tests run automatically on every pull request via GitHub Actions.

---

## Deployment

### Docker

```bash
docker pull ghcr.io/blackroad-os/blackroad-os:latest
docker run -p 3000:3000 --env-file .env ghcr.io/blackroad-os/blackroad-os:latest
```

### Kubernetes (via Helm)

```bash
helm repo add blackroad https://charts.blackroad.io
helm repo update
helm install blackroad-os blackroad/blackroad-os \
  --set stripe.secretKey="$STRIPE_SECRET_KEY" \
  --set stripe.webhookSecret="$STRIPE_WEBHOOK_SECRET"
```

### Vercel / Next.js

```bash
npx @blackroad/cli deploy --platform vercel
```

---

## Repository Index

All active development lives in the **BlackRoad-OS** GitHub organization. Use this index to find the right repository.

### Core Platform

| Repository | Description | Status |
|---|---|---|
| [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) | Core platform monorepo | ✅ Active |
| [BlackRoad-OS/blackroad-os-api](https://github.com/BlackRoad-OS/blackroad-os-api) | REST & GraphQL API server | ✅ Active |
| [BlackRoad-OS/blackroad-os-operator](https://github.com/BlackRoad-OS/blackroad-os-operator) | Kubernetes operator | ✅ Active |
| [BlackRoad-OS/blackroad-os-prism-console](https://github.com/BlackRoad-OS/blackroad-os-prism-console) | Admin dashboard (Prism) | ✅ Active |

### SDKs & Packages

| Package | npm | Repository |
|---|---|---|
| `@blackroad/core` | [npmjs.com](https://www.npmjs.com/package/@blackroad/core) | [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) |
| `@blackroad/payments` | [npmjs.com](https://www.npmjs.com/package/@blackroad/payments) | [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) |
| `@blackroad/identity` | [npmjs.com](https://www.npmjs.com/package/@blackroad/identity) | [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) |
| `@blackroad/cli` | [npmjs.com](https://www.npmjs.com/package/@blackroad/cli) | [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) |

### Websites & Marketing

| Repository | URL | Description |
|---|---|---|
| [BlackRoad-AI/BlackRoad.io](https://github.com/BlackRoad-AI/BlackRoad.io) | [blackroad.io](https://blackroad.io) | Marketing site (hosted under the BlackRoad-AI org) |

### Legacy (Archived)

| Legacy Repository | Replaced By |
|---|---|
| [blackboxprogramming/blackroad](https://github.com/blackboxprogramming/blackroad) | [BlackRoad-OS/blackroad-os](https://github.com/BlackRoad-OS/blackroad-os) |
| [blackboxprogramming/blackroad-api](https://github.com/blackboxprogramming/blackroad-api) | [BlackRoad-OS/blackroad-os-api](https://github.com/BlackRoad-OS/blackroad-os-api) |
| [blackboxprogramming/blackroad-operator](https://github.com/blackboxprogramming/blackroad-operator) | [BlackRoad-OS/blackroad-os-operator](https://github.com/BlackRoad-OS/blackroad-os-operator) |
| [blackboxprogramming/blackroad-prism-console](https://github.com/blackboxprogramming/blackroad-prism-console) | [BlackRoad-OS/blackroad-os-prism-console](https://github.com/BlackRoad-OS/blackroad-os-prism-console) |

---

## Security & Compliance

- All API endpoints are protected with TLS 1.3.
- Stripe webhook signatures are verified on every inbound event using `stripe.webhooks.constructEvent`.
- Secret keys are never logged or stored in plaintext.
- Dependency audits are run automatically via `npm audit` and Dependabot.
- To report a security vulnerability, email **security@blackroad.io** (do not open a public issue).

---

## Support

| Channel | Link |
|---|---|
| 📖 Documentation | [docs.blackroad.io](https://docs.blackroad.io) |
| 💬 Community Discord | [discord.gg/blackroad](https://discord.gg/blackroad) |
| 🐛 Bug Reports | [GitHub Issues](https://github.com/BlackRoad-OS/blackroad-os/issues) |
| 📧 Enterprise Support | [support@blackroad.io](mailto:support@blackroad.io) |
| 🌐 Website | [blackroad.io](https://blackroad.io) |

---

## License

Copyright © 2024–2026 **BlackRoad OS, Inc.** All Rights Reserved.

This software is proprietary and confidential. Unauthorized copying, distribution, or use is strictly prohibited. See [LICENSE](./LICENSE) for full terms.

---

<p align="center">
  Built with ❤️ by <a href="https://blackroad.io">BlackRoad OS, Inc.</a>
</p>
