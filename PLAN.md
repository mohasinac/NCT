# New City Tronics — Platform Plan

**Business**: Local electronics shop · ~10k barcoded parts + 5k+ non-coded parts
**Stack**: Next.js 16 (App Router) · TypeScript · Tailwind CSS 4 · next-intl · Adapter-backed services
**Languages**: English (`en`) + Tamil (`ta`)
**Deployment**: Cloud (Vercel + Firebase) OR self-hosted (Raspberry Pi / VPS / Docker)
**Core principle**: Staff keep the system open all day. Every customer touchpoint flows through it.

---

## Table of Contents

0. [Development Setup](#development-setup)
1. [System Overview](#system-overview)
2. [Design Philosophy](#design-philosophy)
3. [Theming System](#theming-system)
4. [User Roles & Permissions](#user-roles--permissions)
5. [Data Schemas](#data-schemas)
6. [Adapter Architecture](#adapter-architecture)
7. [Search System](#search-system)
8. [Storefront Pages](#storefront-pages)
9. [Operations Panel Pages](#operations-panel-pages)
10. [Source Layout](#source-layout)
11. [API Routes](#api-routes)
12. [Notification System](#notification-system)
13. [Print Outputs](#print-outputs)
14. [Internationalisation](#internationalisation)
15. [Implementation Milestones](#implementation-milestones)
16. [Seeded Data](#seeded-data)
17. [Environment Variables](#environment-variables)
18. [Success Metrics](#success-metrics)

---

## Development Setup

### Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Node.js | ≥ 20 LTS | [nodejs.org](https://nodejs.org) |
| npm | ≥ 10 | bundled with Node.js |
| Firebase CLI | ≥ 13 | `npm i -g firebase-tools` |
| Git | ≥ 2.40 | [git-scm.com](https://git-scm.com) |

**Recommended VS Code extensions**: ESLint · Prettier · Tailwind CSS IntelliSense · Firebase Explorer

---

### Local Development Commands

```bash
npm install              # install dependencies

npm run dev              # dev server (Turbopack) — http://localhost:3000
npm run build            # production build
npm run start            # run production build locally
npm run typecheck        # tsc --noEmit
npm run lint             # next lint
npm test                 # run all unit tests (Vitest)
npm run test:e2e         # Playwright end-to-end tests
npm run emulators        # firebase emulators:start (Auth, Firestore, Storage)
npm run seed             # upsert seed documents (dev only — /dev/seed page)
npm run search:reindex   # full re-index of all products into search engine
```

`package.json` scripts:

```json
{
  "scripts": {
    "dev":            "next dev --turbopack",
    "build":          "next build",
    "start":          "next start",
    "lint":           "next lint",
    "typecheck":      "tsc --noEmit",
    "test":           "vitest run",
    "test:watch":     "vitest",
    "test:e2e":       "playwright test",
    "emulators":      "firebase emulators:start --import=./emulator-data --export-on-exit",
    "search:reindex": "tsx scripts/reindex-search.ts"
  }
}
```

---

### Git Workflow & Branch Strategy

```
main          ← production-ready; protected; CI must pass before merge
  └─ dev      ← integration branch; all phase branches merge here first
       ├─ phase/m0-bootstrap
       ├─ phase/m1-adapters
       ├─ phase/m2-auth
       ├─ phase/m3-theme
       ├─ phase/m4-dashboard-inquiries
       ├─ phase/m5-walkin-print
       ├─ phase/m6-storefront
       ├─ phase/m7-orders-payments
       └─ phase/m8-refunds
```

**Commit convention (Conventional Commits):**

```
feat(search): add instant suggestion dropdown with keyboard nav
fix(cart):    prevent double-submit on slow network
chore(deps):  bump typesense to 2.x
refactor(products): extract PriceTag into standalone component
```

**Per-phase gate** — before merging a phase branch to `dev`:

1. `npm run typecheck` — zero errors
2. `npm run lint` — zero errors
3. `npm run build` — succeeds
4. Manual smoke-test of all new pages / flows

---

### Coding Standards

| Area | Rule |
|------|------|
| Language | TypeScript strict mode. `any` is forbidden — use `unknown` + type guards |
| Imports | Absolute paths via `@/` alias (`@/components`, `@/lib`, `@/types`, etc.) |
| Components | Server Components by default; add `"use client"` only when needed |
| State | Zustand for cart. No prop-drilling beyond 2 levels — use context or Zustand |
| Data fetching | Server Components fetch data via repository functions. Client components use `onSnapshot` only for real-time |
| Styling | Tailwind utility classes only. No inline `style={}` except for dynamic values not expressible in Tailwind. No hardcoded colours — CSS variables only |
| File naming | `PascalCase.tsx` for components, `camelCase.ts` for non-component modules |
| Constants | All magic strings / collection names in `constants/`. Never hardcode Firestore paths in pages |
| Security | Sanitise rich text before rendering. Never trust client-provided user IDs for writes — always read from server session |
| Error handling | `try/catch` with typed errors at API route boundaries only |
| Strings | No hardcoded strings in JSX — use `useTranslations()` / `getTranslations()` from `next-intl` |

---

### Testing Strategy

| Layer | Tool | Scope |
|-------|------|-------|
| Unit | Vitest | `lib/` utilities (formatCurrency, search query builder, discount calc, ref generators) |
| Component | Vitest + React Testing Library | UI primitives (Button, Input, Badge, SearchBox) |
| Integration | Vitest + Firestore emulator | API routes (`/api/checkout`, `/api/inquiries`, `/api/search`) |
| E2E | Playwright | Critical flows: browse → search → add to cart → checkout; staff quote inquiry; walk-in sale |

Test files co-located: `lib/formatCurrency.test.ts`, `components/ui/Button.test.tsx`.

---

## System Overview

Two distinct surfaces. One shared data layer.

```
┌──────────────────────────────────────────────────────────────────┐
│  STOREFRONT  (public)                                            │
│  Landing page · Shop · Gallery · Inquiry · Order tracking        │
│  Looks like a global brand — staff never touch this              │
├──────────────────────────────────────────────────────────────────┤
│  OPERATIONS PANEL  (staff / admin / owner)                       │
│  Inquiries · Orders · Products · Customers · Finance · Reports   │
│  Staff see only what they need. Owner sees everything.           │
└──────────────────────────────────────────────────────────────────┘
```

Staff and the owner manage operations. The storefront takes care of itself — content is driven by the product catalog and CMS blocks the owner can update without code.

---

## Design Philosophy

### "Keep it open" drives every operations panel decision

- Dashboard is genuinely useful at a glance — not a form-filling destination
- Common actions (log a call, walk-in sale, quote an inquiry) are 1–2 clicks from anywhere
- New inquiries and orders surface immediately via real-time badge — no refresh
- The storefront is self-service; staff handle exceptions only

### Catalog grows from demand, not upfront entry

With 15k items and limited staff, a big-bang catalog entry will never happen.

- Phone call → staff logs inquiry → fulfilled → optionally catalog the item
- Walk-in sale → quick-add → "Catalog this?" prompt
- Barcode scan → distributor API auto-fills name, specs, image

After 6 months the catalog reflects what customers actually ask for — prioritised automatically.

---

## Theming System

The shop owner can change the storefront colour theme from Settings without touching code. The operations panel has its own fixed neutral theme (dark/light toggle only).

### Architecture

Themes are stored as a set of CSS custom properties inside `settings/theme` in the database. On every request the layout reads the active theme and injects a `<style>` tag with those variables. No rebuild required.

```
settings/theme {
  active: 'default' | 'ocean' | 'slate' | 'amber' | 'dark' | 'custom'
  custom: ThemeTokens   // only used when active === 'custom'
  colorScheme: 'auto' | 'light' | 'dark'
  showThemeSwitcher: boolean
}
```

### ThemeTokens Schema

```ts
interface ThemeTokens {
  // Brand colours
  colorPrimary: string        // e.g. "#1d4ed8"
  colorPrimaryFg: string      // foreground on primary bg
  colorSecondary: string
  colorSecondaryFg: string
  colorAccent: string
  colorAccentFg: string

  // Surface colours
  colorBg: string             // page background
  colorSurface: string        // card / panel background
  colorBorder: string

  // Text
  colorText: string           // body text
  colorTextMuted: string      // secondary text
  colorTextInverse: string    // text on dark backgrounds

  // Status
  colorSuccess: string
  colorWarning: string
  colorError: string
  colorInfo: string

  // Typography
  fontSans: string            // e.g. "Inter, system-ui, sans-serif"
  fontMono: string
  fontSerif?: string
  fontSizingBase: string      // e.g. "16px"
  borderRadius: string        // e.g. "0.5rem"
}
```

### Preset Themes

| Name | Description | Primary |
|------|-------------|---------|
| `default` | Clean blue — professional electronics | `#1d4ed8` |
| `ocean` | Deep teal — modern tech feel | `#0d9488` |
| `slate` | Near-monochrome — minimal & sharp | `#334155` |
| `amber` | Warm amber — local shop feel | `#b45309` |
| `dark` | Full dark — operator favourite | `#6366f1` |
| `custom` | Owner-defined — full control | any |

Every preset ships with both light and dark variants. The storefront respects `prefers-color-scheme` by default; owner can lock to always-light or always-dark.

### CSS Variable Convention

All Tailwind utility classes in JSX reference CSS variables only. No hardcoded colours anywhere.

```css
/* globals.css — default theme */
:root {
  --color-primary:      #1d4ed8;
  --color-primary-fg:   #ffffff;
  --color-secondary:    #64748b;
  --color-secondary-fg: #ffffff;
  --color-accent:       #f59e0b;
  --color-accent-fg:    #000000;
  --color-bg:           #f8fafc;
  --color-surface:      #ffffff;
  --color-border:       #e2e8f0;
  --color-text:         #0f172a;
  --color-text-muted:   #64748b;
  --color-text-inverse: #ffffff;
  --color-success:      #16a34a;
  --color-warning:      #d97706;
  --color-error:        #dc2626;
  --color-info:         #0284c7;
  --font-sans:          "Inter", system-ui, sans-serif;
  --font-mono:          "JetBrains Mono", monospace;
  --font-sizing-base:   16px;
  --border-radius:      0.5rem;
}

[data-theme="dark"] {
  --color-bg:           #0f172a;
  --color-surface:      #1e293b;
  --color-border:       #334155;
  --color-text:         #f1f5f9;
  --color-text-muted:   #94a3b8;
}
```

The root layout server component reads `settings/theme` and injects a `<style>` tag that overrides these defaults with the active theme's tokens. Zero client JS needed for initial render.

### Tailwind Config

```ts
// tailwind.config.ts
theme: {
  extend: {
    colors: {
      primary:        'var(--color-primary)',
      'primary-fg':   'var(--color-primary-fg)',
      secondary:      'var(--color-secondary)',
      accent:         'var(--color-accent)',
      bg:             'var(--color-bg)',
      surface:        'var(--color-surface)',
      border:         'var(--color-border)',
      text:           'var(--color-text)',
      'text-muted':   'var(--color-text-muted)',
      success:        'var(--color-success)',
      warning:        'var(--color-warning)',
      error:          'var(--color-error)',
      info:           'var(--color-info)',
    },
    fontFamily: {
      sans:  ['var(--font-sans)'],
      mono:  ['var(--font-mono)'],
      serif: ['var(--font-serif, Georgia, serif)'],
    },
    borderRadius: {
      DEFAULT: 'var(--border-radius)',
    },
  }
}
```

### Theme Switcher (Storefront)

Small floating widget bottom-right (owner can disable). Renders swatches for each preset + a dark/light toggle.

### Theme Editor (Settings → Appearance)

```
Settings → Appearance
  ├── Preset themes: [Default] [Ocean] [Slate] [Amber] [Dark]
  ├── Color Scheme:  [Auto] [Light] [Dark]
  ├── Show theme switcher on storefront: [on/off]
  └── Custom Theme (accordion — unlocked when "Custom" chosen)
       ├── Primary colour  (colour picker)
       ├── Accent colour   (colour picker)
       ├── Background + Surface (colour pickers)
       ├── Typography (font family dropdowns — Google Fonts list)
       ├── Border radius (slider: sharp → pill → circle)
       └── [Preview Live]  [Save & Apply]
```

Preview renders a live mini-storefront mockup in the settings page using the chosen values. Saving writes to `settings/theme` and invalidates the layout cache.

---

## User Roles & Permissions

| Role | Who | Access |
|------|-----|--------|
| `customer` | Website visitors | Storefront only. Browse, buy, inquire, track. |
| `staff` | Shop employees | Operations panel: inquiries, orders, walk-in sales, print. |
| `admin` | Store manager | Everything staff + products, customer records, reports. |
| `owner` | Business owner | Everything admin + payment settings, refunds, staff accounts, theme. |

Role hierarchy: `owner > admin > staff > customer`

### Permission Matrix

| Action | staff | admin | owner |
|--------|:-----:|:-----:|:-----:|
| View / log inquiries | ✓ | ✓ | ✓ |
| Quote / source / decline inquiry | ✓ | ✓ | ✓ |
| View / create orders | ✓ | ✓ | ✓ |
| Mark payment received | ✓ | ✓ | ✓ |
| Ship / fulfil order | ✓ | ✓ | ✓ |
| Create / edit products | — | ✓ | ✓ |
| Delete products | — | — | ✓ |
| View reports | — | ✓ | ✓ |
| Approve / deny refunds | — | ✓ | ✓ |
| Payment settings | — | — | ✓ |
| Staff account management | — | — | ✓ |
| Appearance / theme settings | — | — | ✓ |
| Edit policy text | — | ✓ | ✓ |
| Edit CMS blocks | — | ✓ | ✓ |

---

## Data Schemas

### Product

```ts
interface Product {
  id: string
  slug: string
  name: string                          // English
  nameTa?: string                       // Tamil manual override
  description: string                   // rich text, English
  descriptionTa?: string
  type: 'listed' | 'gallery'
  category: string                      // Category.id
  subcategory?: string
  tags: string[]
  images: string[]                      // storage keys / URLs
  barcode?: string
  sku?: string
  price?: number                        // undefined for gallery items
  compareAtPrice?: number               // shows "was ₹X"
  stock?: number
  lowStockThreshold?: number
  requiresConfirmation: boolean         // "Request to Buy" instead of cart
  nonRefundable: boolean
  warrantyPeriodDays?: number
  specs?: Record<string, string>        // { Voltage: "12V", Current: "2A" }
  specsTa?: Record<string, string>      // Tamil spec values
  datasheetUrl?: string
  supplierName?: string
  distributorPartNumber?: string
  createdFrom: 'inquiry' | 'walkin' | 'manual' | 'barcode_scan' | 'import'
  featured: boolean
  publishedAt?: Timestamp
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

### Category

```ts
interface Category {
  id: string
  slug: string
  name: string
  nameTa?: string
  parentId?: string                     // null = top-level
  icon?: string                         // lucide-react icon name
  order: number
  createdAt: Timestamp
}
```

### Inquiry

```ts
type InquiryStatus =
  | 'new'
  | 'viewed'
  | 'quoted'
  | 'accepted'
  | 'declined'
  | 'sourcing'
  | 'fulfilled'
  | 'unavailable'

/*
  State machine:
  new → viewed → quoted → accepted → fulfilled
                        → declined
               → sourcing → quoted → accepted → fulfilled
                                   → unavailable
*/

interface Inquiry {
  id: string
  ref: string                           // INQ-2026-0042
  customerName: string
  phone: string
  email?: string
  description: string
  imageUrls: string[]
  quantity: number
  urgency: 'today' | 'this_week' | 'no_rush'
  source: 'web' | 'phone' | 'walkin' | 'whatsapp'
  status: InquiryStatus
  staffNote?: string                    // internal — never shown to customer
  quotedPrice?: number
  estimatedDate?: Timestamp
  assignedTo?: string                   // AppUser.id
  convertedOrderId?: string
  catalogProductId?: string
  timeline: InquiryEvent[]
  createdAt: Timestamp
  updatedAt: Timestamp
}

interface InquiryEvent {
  at: Timestamp
  by: string                            // user ID
  action: string                        // human-readable e.g. "Quoted ₹450"
  note?: string
}
```

### Order

```ts
type OrderStatus =
  | 'pending_confirmation'
  | 'confirmed'
  | 'processing'
  | 'packed'
  | 'shipped'
  | 'delivered'
  | 'completed'
  | 'cancelled'
  | 'refund_requested'
  | 'refund_approved'
  | 'refund_denied'

/*
  State machine:
  pending_confirmation → confirmed → processing → packed → shipped → delivered → completed
                                                                    → refund_requested
                       → cancelled                                  → refund_approved (full/partial)
                                                                    → refund_denied
*/

type PaymentStatus = 'pending' | 'paid' | 'partially_refunded' | 'fully_refunded'

interface Order {
  id: string
  ref: string                           // ORD-2026-0042
  type: 'online' | 'manual' | 'converted'
  customerId?: string
  customerName: string
  phone: string
  email?: string
  shippingAddress?: Address
  items: OrderItem[]
  subtotal: number
  discountAmount: number
  couponCode?: string
  shippingAmount: number
  taxAmount: number
  total: number
  paymentMethod: 'cash' | 'upi' | 'card' | 'razorpay' | 'cod'
  paymentStatus: PaymentStatus
  paymentRef?: string                   // UPI transaction ID, Razorpay order ID, etc.
  status: OrderStatus
  requiresConfirmation: boolean
  assignedTo?: string
  internalNote?: string
  shippingTrackingNumber?: string
  shippingCourier?: string
  refundHistory: RefundEvent[]
  fromInquiryId?: string
  timeline: OrderEvent[]
  createdAt: Timestamp
  updatedAt: Timestamp
}

interface OrderItem {
  productId?: string                    // undefined for quick-add items
  name: string
  sku?: string
  barcode?: string
  quantity: number
  unitPrice: number
  totalPrice: number
  warrantyPeriodDays?: number
  catalogued: boolean
}

interface OrderEvent {
  at: Timestamp
  by: string
  status: OrderStatus
  note?: string
}

interface Address {
  line1: string
  line2?: string
  city: string
  state: string
  pincode: string
  country: string
}
```

### Refund

```ts
interface RefundEvent {
  id: string
  requestedAt: Timestamp
  requestedBy: 'customer' | 'staff'
  requestedByUserId?: string
  reason: string
  requestedAmount: number
  decision: 'pending' | 'approved_full' | 'approved_partial' | 'denied'
  decidedBy?: string                    // owner / admin user ID
  decidedAt?: Timestamp
  approvedAmount?: number
  decisionNote?: string
  processedAt?: Timestamp
  razorpayRefundId?: string
}
```

### Customer

```ts
interface Customer {
  id: string
  name: string
  phone: string
  email?: string
  authUid?: string                      // Firebase Auth UID if registered
  addresses: Address[]
  defaultAddressIndex: number
  orderCount: number
  inquiryCount: number
  totalSpend: number
  tags: string[]                        // e.g. ['repeat', 'wholesale']
  internalNote?: string
  createdAt: Timestamp
  updatedAt: Timestamp
}
```

### AppUser (Staff / Admin / Owner)

```ts
type UserRole = 'staff' | 'admin' | 'owner'

interface AppUser {
  id: string
  authUid: string
  name: string
  email: string
  phone?: string
  role: UserRole
  active: boolean
  notificationPrefs: NotificationPrefs
  createdAt: Timestamp
  updatedAt: Timestamp
}

interface NotificationPrefs {
  newInquiry: boolean
  newOrder: boolean
  lowStock: boolean
  refundRequest: boolean
  emailEnabled: boolean
}
```

### Settings

```ts
// settings/shop
interface ShopSettings {
  name: string
  tagline?: string
  address: string
  city: string
  state: string
  pincode: string
  phone: string
  whatsapp: string
  email: string
  hours: string                         // "Mon–Sat 9 am – 7 pm"
  gstNumber?: string
  logoUrl?: string
  faviconUrl?: string
  socialLinks: {
    instagram?: string
    facebook?: string
    youtube?: string
  }
}

// settings/payment
interface PaymentSettings {
  cashEnabled: boolean
  upiEnabled: boolean
  upiId?: string
  cardEnabled: boolean
  razorpayEnabled: boolean
  razorpayKeyId?: string
  razorpayKeySecret?: string            // never exposed to client bundle
  codEnabled: boolean
  codMinimumOrder?: number
}

// settings/policies
interface PolicySettings {
  refundWindowDays: number
  refundNote?: string
  shippingPolicy: string                // rich text
  returnsPolicy: string
  privacyPolicy: string
  termsOfService: string
}

// settings/theme
interface ThemeSettings {
  active: 'default' | 'ocean' | 'slate' | 'amber' | 'dark' | 'custom'
  custom?: ThemeTokens
  colorScheme: 'auto' | 'light' | 'dark'
  showThemeSwitcher: boolean
}

// settings/notifications
interface NotificationSettings {
  emailFrom: string
  whatsappEnabled: boolean
  whatsappLinkBase: string              // "https://wa.me/91XXXXXXXXXX?text="
}
```

### CMS Block

```ts
interface CmsBlock {
  id: string
  key: string                           // 'hero_heading', 'announcement_text', …
  type: 'text' | 'richtext' | 'image' | 'product_list' | 'boolean'
  value: unknown
  valueTa?: unknown                     // Tamil variant
  updatedAt: Timestamp
}
```

### Cart Session

```ts
interface CartSession {
  id: string                            // stored in cookie
  items: CartItem[]
  couponCode?: string
  expiresAt: Timestamp                  // TTL: 7 days
  createdAt: Timestamp
  updatedAt: Timestamp
}

interface CartItem {
  productId: string
  name: string
  price: number
  quantity: number
  imageUrl?: string
  requiresConfirmation: boolean
}
```

### Translation Cache

```ts
interface TranslationCache {
  id: string                            // hash(text + targetLang)
  sourceText: string
  targetLang: string
  result: string
  adapter: string                       // 'libre' | 'google'
  createdAt: Timestamp
}
```

---

## Adapter Architecture

App code calls interfaces. Adapters decide where data goes. No Firebase / Prisma imports in business logic.

`ADAPTER_PROFILE=cloud|selfhosted|docker` switches everything at startup.

### Interfaces

```ts
// IDataAdapter — list queries use @mohasinac/sievejs SieveModel
// Each adapter translates SieveModel to its native query mechanism:
//   cloud      → createFirebaseAdapter()  from '@mohasinac/sievejs/adapters/firebase'
//   selfhosted → createPrismaAdapter()    from '@mohasinac/sievejs/adapters/prisma'
//   docker     → createPrismaAdapter()    from '@mohasinac/sievejs/adapters/prisma'
interface IDataAdapter {
  get<T>(collection: string, id: string): Promise<T | null>
  list<T>(collection: string, sieve?: SieveModel, fields?: SieveFields): Promise<AdapterPage<T>>
  create<T>(collection: string, data: Omit<T, 'id'>): Promise<T>
  update<T>(collection: string, id: string, data: Partial<T>): Promise<T>
  delete(collection: string, id: string): Promise<void>
  transaction<T>(fn: (tx: AdapterTransaction) => Promise<T>): Promise<T>
}

// SieveModel (from '@mohasinac/sievejs/models')
// { filters?: string; sorts?: string; page?: number; pageSize?: number }
// DSL: filters="price>=100,price<=500,inStock==true,tags@=arduino", sorts="-createdAt,name"
// OR:  filters="(price|compareAtPrice)>=100",  sorts="name,-updatedAt"
//
// SieveFields — per-collection map of filterable/sortable fields
// Defined as named exports in src/lib/repositories/{collection}.ts
// Example for Products:
//   export const PRODUCT_SIEVE_FIELDS: SieveFields = {
//     name:      { canFilter: true, canSort: true },
//     price:     { canFilter: true, canSort: true },
//     stock:     { canFilter: true, canSort: true },
//     category:  { canFilter: true, canSort: false },
//     featured:  { canFilter: true, canSort: false },
//     createdAt: { path: 'createdAt', canFilter: true, canSort: true },
//   }

interface AdapterPage<T> {
  items: T[]
  total: number
  page: number
  pageSize: number
}
```

```ts
// IAuthAdapter
interface IAuthAdapter {
  verifyToken(token: string): Promise<AuthClaims>
  createUser(data: CreateUserInput): Promise<AppUser>
  setRole(uid: string, role: UserRole): Promise<void>
  sendOtp(phone: string): Promise<void>
  verifyOtp(phone: string, otp: string): Promise<string>  // returns session token
  revokeToken(uid: string): Promise<void>
}
```

```ts
// IStorageAdapter
interface IStorageAdapter {
  upload(path: string, data: Buffer, contentType: string): Promise<string>
  delete(url: string): Promise<void>
  getSignedUrl(url: string, expiresInSeconds: number): Promise<string>
}
```

```ts
// IEmailAdapter
interface IEmailAdapter {
  send(opts: EmailOptions): Promise<void>
}
interface EmailOptions {
  to: string
  subject: string
  html: string
  replyTo?: string
  attachments?: Array<{ filename: string; content: Buffer }>
}
```

```ts
// IRealtimeAdapter
interface IRealtimeAdapter {
  publish(channel: string, event: string, data: unknown): Promise<void>
}
// Client-side subscribe() is in the hook layer, not the adapter
```

```ts
// ITranslationAdapter
interface ITranslationAdapter {
  translate(text: string, targetLang: string): Promise<string>
  translateBatch(texts: string[], targetLang: string): Promise<string[]>
}
```

```ts
// IPaymentAdapter
interface IPaymentAdapter {
  createOrder(opts: PaymentOrderInput): Promise<PaymentOrder>
  verifyWebhook(payload: unknown, signature: string): Promise<PaymentEvent>
  refund(opts: RefundInput): Promise<RefundResult>
}
```

```ts
// ISearchAdapter
interface ISearchAdapter {
  index(document: SearchDocument): Promise<void>
  indexBatch(documents: SearchDocument[]): Promise<void>
  delete(id: string): Promise<void>
  search(query: SearchQuery): Promise<SearchPage>
  suggest(q: string, limit?: number): Promise<SearchSuggestion[]>
}

interface SearchDocument {
  id: string
  slug: string
  name: string
  nameTa?: string
  description: string
  category: string
  subcategory?: string
  tags: string[]
  barcode?: string
  sku?: string
  distributorPartNumber?: string
  specsFlat: string                 // all spec values concatenated for full-text search
  price?: number
  stock?: number
  type: 'listed' | 'gallery'
  imageUrl?: string
  publishedAt?: number              // unix timestamp
}

interface SearchQuery {
  q: string
  filters?: {
    category?: string
    subcategory?: string
    minPrice?: number
    maxPrice?: number
    inStockOnly?: boolean
    type?: 'listed' | 'gallery'
  }
  sort?: 'relevance' | 'price_asc' | 'price_desc' | 'newest' | 'name_az'
  page?: number
  hitsPerPage?: number
}

interface SearchPage {
  hits: SearchHit[]
  total: number
  page: number
  totalPages: number
  processingTimeMs: number
  correctedQuery?: string           // "Did you mean: …?" from Typesense
}

interface SearchHit {
  id: string
  name: string
  slug: string
  imageUrl?: string
  price?: number
  stock?: number
  category: string
  highlights: Record<string, string>  // e.g. { name: "IRF<em>540</em>N" }
}

interface SearchSuggestion {
  id: string
  name: string
  slug: string
  imageUrl?: string
  price?: number
  category: string
}
```

### Profiles

| Profile | Data | Auth | Storage | Email | Realtime | Translation | Payment | Search | Runs on |
|---------|------|------|---------|-------|----------|-------------|---------|--------|---------|
| `cloud` | Firestore | Firebase Auth | Firebase Storage | Resend | Firestore | Google / LibreTranslate | Razorpay | Typesense Cloud | Vercel + Firebase |
| `selfhosted` | SQLite (Prisma) | Local JWT | Local filesystem | Nodemailer | SSE | LibreTranslate | Manual / UPI | Typesense (Docker) | Pi / VPS / PC |
| `docker` | PostgreSQL (Prisma) | Local JWT | S3-compatible | Nodemailer | SSE / WS | LibreTranslate | Manual / Razorpay | Typesense (Docker) | Any Docker host |
| `dev` | In-memory | Local JWT | Local filesystem | Console | In-memory | Console stub | Manual stub | In-memory (JS filter) | localhost |

**SieveJS adapter per profile:** `cloud` → `createFirebaseAdapter()` from `@mohasinac/sievejs/adapters/firebase` · `selfhosted` + `docker` → `createPrismaAdapter()` from `@mohasinac/sievejs/adapters/prisma` · `dev` → in-memory JS filter fallback. Each `IDataAdapter.list()` constructs a `SieveProcessorBase` with the matching adapter and the caller-supplied `SieveFields`, then calls `processor.apply(sieve, baseQuery)`.

### SieveJS Integration — End-to-End Flow

```
Browser URL
  ?filters=category==ics,price>=10&sorts=-createdAt&page=2&pageSize=24
        │
        ▼
  useUrlTable (src/hooks/useUrlTable.ts)
    ├─ reads/writes filter + sort + page state to URL query params
    └─ passes { filters, sorts, page, pageSize } to data fetcher
        │
        ▼
  Next.js API Route  (e.g. GET /api/products)
    └─ createNextRouteHandler(PRODUCT_SIEVE_FIELDS, db, 'products')
         from '@mohasinac/sievejs/integrations/next'
         ├─ parses query params into SieveModel
         ├─ validates fields against PRODUCT_SIEVE_FIELDS (blocks injection)
         └─ calls db.list('products', sieveModel, PRODUCT_SIEVE_FIELDS)
        │
        ▼
  IDataAdapter.list()  (e.g. FirebaseDataAdapter)
    └─ new SieveProcessorBase(createFirebaseAdapter(firestoreRef))
         .apply(sieveModel, baseQuery)
         → returns AdapterPage<T> { items, total, page, pageSize }
```

**`SIEVE_FIELDS` constants** are the only place where filterable/sortable columns are declared. Any field not listed is silently ignored — this prevents arbitrary Firestore index costs and Prisma query injection.

```ts
// src/lib/repositories/products.ts
export const PRODUCT_SIEVE_FIELDS: SieveFields = {
  name:       { canFilter: true,  canSort: true  },
  price:      { canFilter: true,  canSort: true  },
  stock:      { canFilter: true,  canSort: true  },
  category:   { canFilter: true,  canSort: false },
  subcategory:{ canFilter: true,  canSort: false },
  featured:   { canFilter: true,  canSort: false },
  type:       { canFilter: true,  canSort: false },
  createdAt:  { canFilter: true,  canSort: true  },
}
```

The admin DataTable, shop `/shop` filter sidebar, and all `GET` list API routes all consume the same DSL — no duplicated filter logic anywhere in the codebase.

### Factory

```ts
// src/lib/adapters.ts
const profile = process.env.ADAPTER_PROFILE ?? 'cloud'

export const db         = createDataAdapter(profile)
export const auth       = createAuthAdapter(profile)
export const storage    = createStorageAdapter(profile)
export const email      = createEmailAdapter(profile)
export const realtime   = createRealtimeAdapter(profile)
export const translate  = createTranslationAdapter(profile)
export const payment    = createPaymentAdapter(profile)
export const search     = createSearchAdapter(profile)
```

### Self-hosted Minimum Spec

- Raspberry Pi 4 (2 GB+), any Windows / Linux / macOS
- Node.js 20+ · ~200 MB RAM idle · ~500 MB disk
- No internet required after install — email optional, WhatsApp is a link
- SQLite = one file; backup = `cp /data/db.sqlite /data/backups/`
- Images served from `/data/uploads/` as Next.js static assets
- Typesense selfhosted: ~30 MB RAM, single Docker container, runs on the same Pi

---

## Search System

With 15k+ products Firestore's native query has hard limits:

- No substring matching — `name >= 'ir'` only does prefix on one field
- No multi-field search — can't search `name` AND `sku` AND `barcode` together
- No fuzzy matching — typo `irfh540` won't find `IRF540N`
- No relevance ranking — results sorted by field value, not match quality

A dedicated search engine (`ISearchAdapter`) is mandatory for the catalog.

### Recommended Engine: Typesense

| Concern | Answer |
|---------|--------|
| Cloud profile | Typesense Cloud — free tier: 10k docs, 100k searches/month |
| Selfhosted profile | Typesense in Docker — free, open-source, same API client |
| RAM on Raspberry Pi 4 | ~30 MB idle for Typesense |
| TypeScript SDK | `typesense` (official) |
| Cloud/selfhosted parity | Same SDK; only the `host` env var changes |
| Alternative | Algolia (free 10k ops/month) if Typesense is not preferred |

### Search Index Schema

The `products` Typesense collection has the following schema:

```
Field             Type          Facet    Search
─────────────     ──────────    ─────    ──────
id                string        —        —
slug              string        —        —
name              string        —        ✓ (weight 5)
nameTa            string?       —        ✓ (weight 3)
description       string        —        ✓ (weight 1)
category          string        ✓        ✓
subcategory       string?       ✓        ✓
tags              string[]      ✓        ✓
barcode           string?       —        ✓ (weight 4)
sku               string?       —        ✓ (weight 4)
distributorPartNumber  string?  —        ✓ (weight 4)
specsFlat         string        —        ✓ (weight 2)  // all spec values joined
price             float         —        —  (used for sorting and range filter)
stock             int32         —        —
type              string        ✓        —
publishedAt       int64         —        —
```

### Indexing Triggers

Products are indexed synchronously after database writes in the repository layer:

```ts
// src/lib/repositories/products.ts
async function createProduct(data) {
  const product = await db.create('products', data)
  await search.index(toSearchDocument(product))   // ← after DB write
  return product
}

async function updateProduct(id, data) {
  const product = await db.update('products', id, data)
  await search.index(toSearchDocument(product))   // ← update index
  return product
}

async function deleteProduct(id) {
  await db.delete('products', id)
  await search.delete(id)                         // ← remove from index
}
```

Full re-index: `POST /api/admin/search/reindex` (owner-only) — reads all products from DB, calls `search.indexBatch()`. Run once on initial setup, and after any index schema changes.

### Search Page (`/[locale]/search`)

- Route: `app/[locale]/search/page.tsx` — **SSR, no cache** (`export const dynamic = 'force-dynamic'`)
- `?q=irf540&category=semiconductors&sort=price_asc&page=2`
- Left sidebar: category facet, subcategory facet, price range slider, in-stock toggle
- Sort selector: Relevance / Price ↑ / Price ↓ / Newest / A–Z
- Product cards with highlighted match terms (`<em>IRF540</em>N`)
- Empty state with `correctedQuery` from Typesense: "Did you mean: **IRF540N**?"
- Zero results + zero suggestions: "Try a broader search or [Submit an Inquiry]"
- URL-driven state — all filters and sort are query params, browser back works

### Instant Suggestions (Header SearchBox)

```
GET /api/search/suggest?q=irf&limit=6
```

- Debounced at **250 ms** client-side in `useSearch` hook
- Returns max 6 `SearchSuggestion` items (name, slug, image, price, category)
- Dropdown shows: product image thumbnail, name, category tag, price
- Fully keyboard navigable: `↑` `↓` Enter, Escape closes
- Enter or clicking "Search" → navigates to `/[locale]/search?q=irf`
- Clicking a suggestion → navigates directly to `/[locale]/shop/[slug]`
- No results in suggestions → shows "Press Enter to search all results"

### Admin + Walk-in Sale Search

The walk-in sale modal's catalog search and the admin products list both use `search.search()` — not Firestore queries. Staff typing a barcode, part number, or name in the modal get instant results across all 15k products.

### `InMemorySearchAdapter` (dev profile)

Filters the in-memory product array using JS `includes()` on name + sku + barcode + tags. No external process needed during local development. Results have no highlights and no ranking, but the interface contract is identical so switching to Typesense is a config change, not a code change.

---

## Storefront Pages

```
/[locale]/                              Landing page
/[locale]/shop                          Product listing (search, filter, sort)
/[locale]/shop/[slug]                   Product detail
/[locale]/gallery                       Parts gallery (inquiry-only items)
/[locale]/gallery/[slug]                Gallery item detail
/[locale]/inquiry                       Submit inquiry (no login required)
/[locale]/inquiry/[ref]                 Inquiry status tracker
/[locale]/cart                          Cart
/[locale]/checkout                      Checkout
/[locale]/checkout/success              Order confirmation
/[locale]/orders/[ref]                  Order tracker (phone + ref, no login)
/[locale]/account/                      Customer account (optional login)
/[locale]/account/orders                Order history
/[locale]/account/inquiries             Inquiry history
/[locale]/account/refunds               Refund requests
/[locale]/policies/shipping             Shipping policy
/[locale]/policies/returns              Returns & refunds policy
/[locale]/policies/privacy              Privacy policy
/[locale]/policies/terms                Terms of service
```

### Landing Page Layout

```
┌──────────────────────────────────────────────────┐
│  HEADER                                          │
│  Logo · Nav · Search · Cart · EN|தமிழ் · Account │
├──────────────────────────────────────────────────┤
│  ANNOUNCEMENT BAR  (CMS, optional)               │
│  "Free shipping above ₹499"                      │
├──────────────────────────────────────────────────┤
│  HERO  (CMS — image + heading + CTA)             │
│  "Shop Now" · "Find a Part" · "Inquire"          │
├──────────────────────────────────────────────────┤
│  CATEGORY GRID  (driven by categories table)     │
├──────────────────────────────────────────────────┤
│  FEATURED PRODUCTS  (admin-curated)              │
├──────────────────────────────────────────────────┤
│  WHY SHOP HERE  (CMS block)                      │
│  Fast sourcing · 15k+ parts · Warranty · Local   │
├──────────────────────────────────────────────────┤
│  INQUIRY CTA BANNER  (CMS)                       │
│  "Can't find it? We'll source it."               │
├──────────────────────────────────────────────────┤
│  RECENT ARRIVALS  (auto: latest listed products) │
├──────────────────────────────────────────────────┤
│  FOOTER                                          │
│  Address · Hours · Phone · WhatsApp · Socials    │
│  Policy links · EN|தமிழ்                         │
└──────────────────────────────────────────────────┘
```

### Shop (`/[locale]/shop`)

- Instant search with suggestions (debounced 300 ms)
- Filters: category, subcategory, price range, in-stock only, brand  
  URL-driven SieveJS DSL: `?filters=category==ics,price>=10,price<=500,inStock==true&sorts=price&page=2&pageSize=24`
- Sort: price ↑↓, newest, popularity
- Product card: image, name, price, stock badge, "Add to Cart" / "Request to Buy"
- Infinite scroll or pagination (configurable in settings)

### Product Detail (`/[locale]/shop/[slug]`)

- Image gallery (zoom, swipe on mobile)
- Name, price, stock status, SKU, barcode
- Description (rich text)
- Specs table (key / value)
- Warranty period badge
- Qty selector + "Add to Cart" (listed) or "Request to Buy" (requiresConfirmation)
- `[Translate to Tamil]` toggle — swaps content values in place, no navigation
- Related products
- Datasheet download button

---

## Operations Panel Pages

```
/[locale]/admin                               Dashboard
/[locale]/admin/inquiries                     Inquiry list
/[locale]/admin/inquiries/new                 Log phone / WhatsApp inquiry
/[locale]/admin/inquiries/[id]                Inquiry detail + actions
/[locale]/admin/orders                        Order list
/[locale]/admin/orders/new                    Walk-in / phone sale
/[locale]/admin/orders/[id]                   Order detail + fulfilment
/[locale]/admin/orders/[id]/refund            Refund review (owner/admin)
/[locale]/admin/products                      Catalog list
/[locale]/admin/products/new                  Create product
/[locale]/admin/products/[id]                 Edit product
/[locale]/admin/customers                     Customer list
/[locale]/admin/customers/[id]                Customer detail + full history
/[locale]/admin/reports                       Reports overview
/[locale]/admin/reports/demand                Most-requested uncatalogued items
/[locale]/admin/reports/refunds               Refund audit log
/[locale]/admin/settings                      Settings index (owner only)
/[locale]/admin/settings/shop                 Shop info, hours, contact
/[locale]/admin/settings/payment              Payment methods + Razorpay config
/[locale]/admin/settings/policies             Refund window + policy text
/[locale]/admin/settings/notifications        Per-staff alert preferences
/[locale]/admin/settings/staff                Staff account management
/[locale]/admin/settings/appearance           Theme editor (owner only)
/[locale]/admin/cms                           CMS blocks editor
```

### Dashboard Layout

```
┌──────────────────────────────────────────────────────────────┐
│  New City Tronics      [+ Log Inquiry]  [+ Walk-in Sale]     │
├───────────────┬──────────────────────────────────────────────┤
│               │  ACTION QUEUE (today)                        │
│  Inquiries(3) │  ● NEW  Ramesh · IRF540N · urgent            │
│  Orders   (5) │  ● NEW  Priya · 5× USB-C cable              │
│  Sourcing (2) │  ● QUOTED  Kumar · Arduino Uno · awaiting    │
│  Refunds  (1) │  ● SHIP  Order #1042 · packed               │
│               │  ● REFUND  Order #1038 · pending review      │
│  Products     │                                              │
│  Customers    │  TODAY'S STATS                               │
│  Finance      │  Sales ₹4,200 · Inquiries 7 · Fulfilled 3   │
│  Reports      │  Avg response time: 14 min                   │
│  Settings     │                                              │
└───────────────┴──────────────────────────────────────────────┘
```

Real-time via Firestore listeners (`cloud`) or SSE `/api/events` (`selfhosted`). No polling.

---

## Source Layout

```
src/
  adapters/
    interfaces/
      IDataAdapter.ts
      IAuthAdapter.ts
      IStorageAdapter.ts
      IEmailAdapter.ts
      IRealtimeAdapter.ts
      ITranslationAdapter.ts
      IPaymentAdapter.ts
      ISearchAdapter.ts
      index.ts
    firebase/
      FirebaseDataAdapter.ts
      FirebaseAuthAdapter.ts
      FirebaseStorageAdapter.ts
      FirestoreRealtimeAdapter.ts
    sqlite/
      SqliteDataAdapter.ts              // Prisma + SQLite
    postgres/
      PostgresDataAdapter.ts            // Prisma + PostgreSQL
    local/
      LocalAuthAdapter.ts               // JWT (jose)
      LocalStorageAdapter.ts            // /data/uploads
    email/
      ResendEmailAdapter.ts
      NodemailerEmailAdapter.ts
      ConsoleEmailAdapter.ts            // dev — logs to terminal
    realtime/
      FirestoreRealtimeAdapter.ts
      SseRealtimeAdapter.ts
    translation/
      LibreTranslateAdapter.ts
      GoogleTranslateAdapter.ts
      CachedTranslationAdapter.ts
    payment/
      ManualPaymentAdapter.ts
      RazorpayPaymentAdapter.ts
    search/
      TypesenseSearchAdapter.ts         // cloud + selfhosted + docker
      InMemorySearchAdapter.ts          // dev profile — JS array filter, no external process
  lib/
    adapters.ts                         // profile factory
    theme.ts                            // loadThemeTokens() → CSS string
    constants.ts
    repositories/
      products.ts                       // exports PRODUCT_SIEVE_FIELDS; create/update/delete also call search.index / search.delete
      orders.ts                         // exports ORDER_SIEVE_FIELDS
      inquiries.ts                      // exports INQUIRY_SIEVE_FIELDS
      customers.ts                      // exports CUSTOMER_SIEVE_FIELDS
      users.ts
      settings.ts
      cms.ts
      cart.ts
      translations.ts
    email-templates/
      inquiry-confirmation.tsx
      inquiry-quoted.tsx
      order-confirmation.tsx
      order-shipped.tsx
      refund-decision.tsx
      new-staff-account.tsx
  stores/
    useCartStore.ts                     // Zustand: cart items, add/remove/clear/count
    useAuthStore.ts                     // Zustand: current storefront customer session
  app/
    [locale]/
      layout.tsx                        // injects theme <style> tag + Zustand providers
      page.tsx                          // landing
      shop/
      gallery/
      search/
        page.tsx                        // SSR, force-dynamic — full-text search results
        loading.tsx
      inquiry/
      cart/
      checkout/
      orders/
      account/
      policies/
    [locale]/(admin)/
      layout.tsx                        // auth guard + sidebar
      page.tsx                          // dashboard
      inquiries/
      orders/
      products/
      customers/
      reports/
      settings/
        appearance/                     // theme editor
      cms/
    api/
      auth/ · inquiries/ · orders/ · products/
      customers/ · cart/ · checkout/
      payments/razorpay/webhook/
      refunds/ · translate/ · upload/
      events/                           // SSE (selfhosted)
      settings/theme/ · settings/shop/ · settings/payment/
      search/
        route.ts                        // GET /api/search
        suggest/route.ts                // GET /api/search/suggest
      admin/search/reindex/route.ts     // POST — owner only, full re-index
  components/
    ui/
      Button.tsx · Input.tsx · Select.tsx · Textarea.tsx
      Modal.tsx · Toast.tsx · Badge.tsx · Spinner.tsx
      Tabs.tsx · Accordion.tsx · DataTable.tsx
      ImageUploader.tsx · RichTextEditor.tsx
      ColorPicker.tsx                   // for theme editor
    layout/
      Header.tsx · Footer.tsx · AdminSidebar.tsx
      ThemeSwitcher.tsx                 // floating storefront widget
      SearchBox.tsx                     // header instant-search input + suggestion dropdown
    storefront/
      ProductCard.tsx · ProductGallery.tsx
      CategoryGrid.tsx · InquiryForm.tsx · CartDrawer.tsx
      SearchResults.tsx                 // search page result grid with hit highlights
      SearchFilters.tsx                 // sidebar: category facets, price range, in-stock toggle
    admin/
      DashboardQueue.tsx · InquiryDetail.tsx · OrderDetail.tsx
      RefundModal.tsx · WalkInSaleModal.tsx · BarcodeScanner.tsx
      ThemeEditor.tsx · ThemePreview.tsx
  hooks/
    useCart.ts
    useRealtime.ts
    useProductTranslation.ts
    useTheme.ts                         // dark/light client toggle
    useUrlTable.ts                      // filter/sort/page as SieveJS DSL in URL (?filters=&sorts=&page=&pageSize=)
    useSearch.ts                        // debounced instant suggestions (250 ms), navigate on Enter
  constants/
    routes.ts
    collections.ts
    themes.ts                           // preset ThemeTokens objects (exported as named typed consts)
    categories.ts                       // seeded category tree
  types/
    index.ts                            // all shared interfaces
  middleware.ts                         // next-intl + auth protection
```

---

## API Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/products` | — | List with search / filter / sort / cursor |
| `GET` | `/api/products/[id]` | — | Single product |
| `POST` | `/api/products` | admin+ | Create |
| `PATCH` | `/api/products/[id]` | admin+ | Update |
| `DELETE` | `/api/products/[id]` | owner | Delete |
| `GET` | `/api/inquiries` | staff+ | List |
| `POST` | `/api/inquiries` | — | Submit (public) |
| `GET` | `/api/inquiries/[id]` | staff+ | Get |
| `PATCH` | `/api/inquiries/[id]` | staff+ | Update status / quote / assign |
| `GET` | `/api/orders` | staff+ | List |
| `POST` | `/api/orders` | — | Place order (public checkout) |
| `GET` | `/api/orders/[id]` | staff+ / owner | Get |
| `PATCH` | `/api/orders/[id]` | staff+ | Update status |
| `GET` | `/api/customers` | admin+ | List |
| `GET` | `/api/customers/[id]` | admin+ | Detail |
| `POST` | `/api/cart` | — | Create / update session |
| `GET` | `/api/cart/[id]` | — | Get cart |
| `POST` | `/api/checkout` | — | Process checkout |
| `POST` | `/api/payments/razorpay/webhook` | — (HMAC) | Razorpay event |
| `POST` | `/api/refunds` | — / staff | Request refund |
| `PATCH` | `/api/refunds/[id]` | admin+ | Approve / deny |
| `POST` | `/api/translate` | — | Translate product content |
| `POST` | `/api/upload` | staff+ | Upload image |
| `GET` | `/api/events` | staff+ | SSE stream (selfhosted profile) |
| `GET` | `/api/settings/theme` | — | Active theme tokens (public, cached) |
| `PATCH` | `/api/settings/theme` | owner | Update theme |
| `GET` | `/api/settings/shop` | — | Public shop info |
| `PATCH` | `/api/settings/shop` | owner | Update |
| `PATCH` | `/api/settings/payment` | owner | Update |
| `PATCH` | `/api/settings/policies` | admin+ | Update |
| `GET` | `/api/auth/otp/send` | — | Send phone OTP |
| `POST` | `/api/auth/otp/verify` | — | Verify OTP → token |
| `GET` | `/api/search` | — | Full-text search (`?q=&category=&sort=&page=&inStock=`) |
| `GET` | `/api/search/suggest` | — | Instant suggestions (`?q=&limit=6`) — debounced 250 ms |
| `POST` | `/api/admin/search/reindex` | owner | Full re-index all products (after schema change / initial setup) |

All non-public routes: bearer token in `Authorization` header, verified via `auth.verifyToken()`.  
All request bodies: Zod-validated at the route boundary before any processing.  
All `GET` list routes (`/api/products`, `/api/inquiries`, `/api/orders`, `/api/customers`) accept SieveJS DSL params: `?filters=&sorts=&page=&pageSize=` — handled via `createNextRouteHandler` from `@mohasinac/sievejs/integrations`.

### Razorpay Webhook Security

```ts
const expectedSig = crypto
  .createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET!)
  .update(rawBody)
  .digest('hex')
if (sig !== expectedSig) return Response.json({ error: 'Unauthorized' }, { status: 401 })
```

---

## Notification System

| Event | Staff / Owner | Customer |
|---|---|---|
| New web inquiry | Real-time badge + email | Confirmation email + WhatsApp |
| Inquiry quoted | — | Email + WhatsApp |
| Inquiry declined | — | Email |
| New online order | Real-time badge + email | Confirmation email |
| Order confirmed | — | Email + WhatsApp |
| Order shipped | — | Email + WhatsApp + tracking link |
| Order delivered | — | Email + review nudge |
| Refund requested | Owner + admin badge + email | Confirmation |
| Refund decision | — | Email + WhatsApp |
| Low stock alert | Admin + owner email | — |
| New staff account | New staff email | — |

**Real-time**: Firestore listeners (`cloud`) or SSE `/api/events` (`selfhosted`). No polling.  
**WhatsApp**: `wa.me/` deep link — zero cost, works in every deployment profile.  
**Email**: Resend (`cloud`) or Nodemailer SMTP (`selfhosted`; Gmail app password works).

---

## Print Outputs

All server-side rendered with `@media print` CSS. No PDF library needed. Works fully offline.

| Output | Who prints | Contents |
|--------|-----------|----------|
| Invoice | Staff / auto on checkout | Items, qty, price, tax, total, payment method, ref, shop address |
| Warranty card | Staff | Product, purchase date, expiry date, shop contact, terms |
| Shelf label | Staff | Product name, barcode / QR code, SKU, price, category |
| Refund receipt | Auto on refund approval | Original order ref, refund amount, date, reason |
| Delivery challan | Staff | Order items list for driver / courier |

---

## Internationalisation

### Static UI — fully translated

- All labels, buttons, status text, errors, notifications: `messages/en.json` + `messages/ta.json`
- Locale switcher: `EN | தமிழ்` in header — storefront and operations panel
- Default: `en`. Routes: `/en/...` and `/ta/...`
- Tamil font: **Noto Sans Tamil** via `next/font` — works offline, no CDN dependency

### Namespace Scaffold

`messages/en.json` is organised by namespace. Add Tamil equivalents to `messages/ta.json`.

```json
{
  "nav":      { "shop": "Shop", "inquiry": "Inquiry", "cart": "Cart", "account": "Account" },
  "home":     { "hero_cta": "Shop Now", "inquiry_cta": "Can't find it? Ask us" },
  "shop":     { "sort_relevance": "Relevance", "sort_price_asc": "Price ↑", "filter_in_stock": "In stock only", "no_results": "No results for \"{{q}}\"" },
  "search":   { "placeholder": "Search 15,000+ parts…", "suggestions_heading": "Suggestions", "view_all": "View all results for \"{{q}}\"" },
  "product":  { "add_to_cart": "Add to Cart", "translate": "Translate to Tamil", "show_original": "Show Original", "inquiry": "Request this part" },
  "cart":     { "empty": "Your cart is empty", "checkout": "Checkout", "remove": "Remove" },
  "checkout": { "place_order": "Place Order", "payment_cod": "Cash on Delivery", "payment_upi": "UPI" },
  "inquiry":  { "submit": "Submit Inquiry", "ref_label": "Your reference", "track": "Track Inquiry" },
  "admin":    { "dashboard": "Dashboard", "inquiries": "Inquiries", "orders": "Orders", "products": "Products", "customers": "Customers", "settings": "Settings" },
  "errors":   { "not_found": "Not found", "unauthorized": "Unauthorized", "server": "Something went wrong" },
  "policies": { "returns": "Returns Policy", "privacy": "Privacy Policy", "warranty": "Warranty Policy" }
}
```

### Product Content — dynamic, on demand

Server always returns English. `useProductTranslation(product)` hook returns same-shape object with translated values when Tamil is active. UI code never changes — it reads `product.name` regardless of locale.

```
English: { name: "Water Pump Module",        specs: { Voltage: "12V"  } }
Tamil:   { name: "வாட்டர் பம்ப் மாட்யூல்",  specs: { Voltage: "12வி" } }
```

Fields translated: `name`, `description`, `tags`, `specs` values  
Fields passed through unchanged: `price`, `stock`, `barcode`, `sku`, `images`, spec keys

Toggle: `[Translate to Tamil]` / `[Show Original]` — preference persisted in `localStorage`  
Cache: `translations` collection — `hash(text+lang) → result` — populated on first use

**Translation adapters**: `LibreTranslateAdapter` (selfhosted, free, fully offline) · `GoogleTranslateAdapter` (cloud) · `CachedTranslationAdapter` (wraps either)

---

## Implementation Milestones

### M0 — Project Bootstrap (2 days)

- `npx create-next-app@latest` — App Router, TypeScript, Tailwind CSS 4
- Install: `next-intl`, `zod`, `react-hook-form`, `@hookform/resolvers`, `lucide-react`, `@mohasinac/sievejs`, `zustand`
- `messages/en.json` + `messages/ta.json` scaffolds with namespace structure
- `middleware.ts` — next-intl locale routing (`/en`, `/ta`)
- Tailwind config with all CSS variable tokens (per Theming section)
- `globals.css` — default theme CSS custom properties
- `src/types/index.ts` — all interfaces from this plan
- `src/constants/collections.ts`, `routes.ts`, `themes.ts`, `categories.ts`
- `src/lib/adapters.ts` factory stub resolved to `InMemoryDataAdapter` + `ConsoleEmailAdapter` + `InMemorySearchAdapter`
- `src/stores/useCartStore.ts` and `useAuthStore.ts` (Zustand, empty initial state)

**Deliverable**: `npm run dev` opens a blank themed page at `/en/`.

---

### M1 — Adapter Scaffold (3 days)

- All 8 interfaces fully defined in `src/adapters/interfaces/` (including `ISearchAdapter`)
- `ConsoleEmailAdapter` — `console.log` stub, no SMTP
- `InMemoryDataAdapter` — array-backed dev adapter
- `InMemorySearchAdapter` — JS `Array.filter` over in-memory product list; no Typesense process needed during dev
- `TypesenseSearchAdapter` stub — wires to Typesense when `ADAPTER_PROFILE` is `cloud`, `selfhosted`, or `docker`
- `LocalAuthAdapter` — JWT with `jose`, `httpOnly` cookie session
- `FirebaseDataAdapter` — Firestore Admin SDK
- `FirebaseAuthAdapter` — Firebase Auth Admin
- `FirebaseStorageAdapter` — Firebase Storage Admin
- `FirestoreRealtimeAdapter` — publishes to Firestore collection watched by client
- `PRODUCT_SIEVE_FIELDS`, `ORDER_SIEVE_FIELDS`, `INQUIRY_SIEVE_FIELDS`, `CUSTOMER_SIEVE_FIELDS` defined in their respective repository files
- `GET /api/products`, `/api/orders`, `/api/inquiries`, `/api/customers` each wired with `createNextRouteHandler` from `@mohasinac/sievejs/integrations/next`
- `src/hooks/useUrlTable.ts` scaffolded — reads/writes filter + sort + page state to URL query params
- `ADAPTER_PROFILE=cloud` → Firebase + Typesense Cloud; `ADAPTER_PROFILE=dev` → in-memory + InMemorySearch

**Deliverable**: `GET /api/products?filters=category==ics,price>=10&sorts=-createdAt&page=1&pageSize=24` returns a filtered, sorted `AdapterPage<Product>`. `GET /api/search?q=pump` returns results.

---

### M2 — Auth + Role Guard (2 days)

- Sign-in page at `/[locale]/admin/login` (email + password, no external OAuth)
- `LocalAuthAdapter` issues JWT in `httpOnly` cookie
- `middleware.ts` — protect `/[locale]/admin/*`, redirect unauthenticated to login
- `requireRole(minRole, request)` server helper — returns 403 if role too low
- Seed one `owner` user in startup script

**Deliverable**: `/en/admin` unauthenticated → redirected to login. Login works. Role guard returns 403.

---

### M3 — Theme System (2 days)

- `src/constants/themes.ts` — all 5 preset `ThemeTokens` objects
- `src/lib/theme.ts` — `loadThemeTokens()` reads `settings/theme` → returns CSS `<style>` string
- Root `app/[locale]/layout.tsx` — calls `loadThemeTokens()`, injects `<style>` server-side
- `src/hooks/useTheme.ts` — client toggle writes `data-theme="dark"` on `<html>`, respects `prefers-color-scheme`
- `ThemeSwitcher.tsx` — floating widget (swatches + dark/light toggle)
- `ThemeEditor.tsx` + `ThemePreview.tsx` — `settings/appearance` page
- `PATCH /api/settings/theme` — owner-only, Zod-validated, saves to `settings/theme`

**Deliverable**: Owner changes preset in Settings → storefront switches colours in next request without rebuild.

---

### M4 — Dashboard + Inquiries (1 week)

Zero products needed. Staff workflow goes live end-to-end.

- Admin shell: sidebar nav, top bar, real-time badge counter
- Dashboard: action queue (real-time listener), today's stats cards
- `[+ Log Inquiry]` quick form (modal, 4 fields: name, phone, description, urgency)
- Inquiry list: sortable table with status / source / urgency / date / assigned-to filter
- Inquiry detail: full state machine UI
  - Quote (price + ETA → customer notified)
  - Mark Sourcing (supplier note)
  - Decline (reason → customer notified)
  - Add internal note
  - Assign to staff member
  - Convert to Order (pre-fills order form)
  - Catalog this item (pre-fills product form)
- Public inquiry form `/[locale]/inquiry` + status tracker `/[locale]/inquiry/[ref]`
- Email + WhatsApp notifications on submission and status changes

**Deliverable**: Staff can receive, view, quote, and close a customer inquiry end-to-end.

---

### M5 — Walk-in Sales + Print (1 week)

- `[+ Walk-in Sale]` modal with two item entry modes:
  - **Catalog search** — debounced live search, select → qty
  - **Quick-add** — name, price, optional barcode, optional image
- Customer details (name + phone — optional for cash sales)
- Payment method selector (cash / UPI / card)
- "Catalog this item?" prompt for quick-add items
- Link to open inquiry → auto-closes matched inquiry on sale
- Invoice, warranty card, delivery challan — `@media print`
- "Share via WhatsApp" → `wa.me/` link with invoice URL
- Shelf label + barcode/QR generation (`bwip-js`, server-side rendered)

**Deliverable**: Walk-in sale created, printed, and closed in under 60 seconds.

---

### M6 — Storefront (3 weeks)

- Landing page: all CMS sections wired and editable
- `/shop` — search, URL-driven filter + sort, pagination
- `/search` — full-text search page (SSR, `force-dynamic`):
  - `?q=` drives `search.search()` — returns `SearchPage` with `hits`, `total`, `totalPages`
  - Left sidebar: `SearchFilters` (category facets, price range slider, in-stock toggle)
  - Sort selector: Relevance / Price ↑ / Price ↓ / Newest / A–Z
  - Hit cards display highlighted matches (`hits[].highlights`)
  - "No results" → spelling suggestion; none found → [Submit an Inquiry] CTA
  - All filter + sort state in URL query string (sharable, back-button safe)
- `SearchBox` in header: instant suggestions via `useSearch` (250 ms debounce), keyboard-navigable dropdown, Enter → `/search?q=`
- Product detail — gallery, specs, translate toggle, related products
- Parts gallery + inquiry form pre-fill
- Cart (cookie-backed `CartSession` → `useCartStore` Zustand)
- Guest checkout → `POST /api/checkout` → confirmation page
- Order tracking `/orders/[ref]` — phone + ref, no login
- Category tree seeded from `src/constants/categories.ts`

**Deliverable**: Customer types a part name in the header → instant suggestions appear → pressing Enter shows full search page with facets, highlights, and pagination. Guest checkout works end-to-end.

---

### M7 — Orders + Payments (2 weeks)

- Full order state machine in operations panel
- Manual UPI / cash confirmation (staff marks "Payment received")
- Payment reference number recorded for audit trail
- Razorpay integration (optional, owner-activated):
  - Checkout → `POST /api/checkout` → Razorpay order → client Razorpay JS → webhook confirms
  - HMAC webhook verification (see API Routes)
- COD flow — status advances to `confirmed` on delivery
- Stock decrement on `confirmed` transition (transactional)
- `requiresConfirmation` → "Request to Buy" → staff confirms availability → payment

**Deliverable**: Online UPI and COD orders work end-to-end. Razorpay is optional.

---

### M8 — Refund System (1 week)

- Refund request: from customer (order tracker) + from staff (order detail)
- `POST /api/refunds` → `RefundEvent { decision: 'pending' }` on order
- Owner + admin get real-time badge + email notification
- Refund review page `/admin/orders/[id]/refund`:
  - Approve full → Razorpay full refund API (if applicable) → `refund_approved`
  - Approve partial → staff enters amount + reason → partial Razorpay refund
  - Deny → staff enters reason → `refund_denied`
- Customer notified of decision by email + WhatsApp
- Refund receipt auto-generated on approval
- Audit log `/admin/reports/refunds` — permanent, no delete
- Configurable refund window (days after delivery) in Settings → Policies

**Deliverable**: Owner approves/partially-approves/denies refunds; customer sees decision on tracker.

---

### M9 — Product Catalog + Barcode (2 weeks)

- Product CRUD: all fields including Tamil overrides
- Barcode scan → Digi-Key / Mouser / LCSC API → auto-fills name, specs, image, datasheet
- `bwip-js` barcode / QR generation (server-side, no client bundle cost)
- Shelf label print
- "Catalog from inquiry" — pre-fills product form from inquiry description + images
- Low stock email alerts when `stock <= lowStockThreshold`
- Bulk CSV import for existing inventory

**Deliverable**: Admin scans barcode → verifies auto-fill → sets price + stock → published in one flow.

---

### M10 — Tamil Translation (1 week)

- `LibreTranslateAdapter` + `GoogleTranslateAdapter` + `CachedTranslationAdapter`
- `useProductTranslation(product)` hook — returns translated product shape
- `[Translate to Tamil]` toggle on all product + gallery pages
- `localStorage` preference persisted across sessions
- Manual Tamil override fields on product edit form (take precedence over auto-translation)
- `POST /api/translate` caches result in `translations` collection

**Deliverable**: Tamil translation works on product pages, results cached, manual overrides respected.

---

### M11 — Customer Accounts (1 week)

- Phone OTP or email + password sign-in
- Account pages: order history, inquiry history, refund requests
- Saved addresses with default selection
- Repeat-customer detection — tag + badge in admin customer list

**Deliverable**: Customer signs in, views history, requests refund from account.

---

### M12 — Self-hosted Hardening (1 week)

- `selfhosted` profile: SQLite + local JWT + local file storage + SSE
- Prisma schema covering all collections
- `next.config.ts` `output: 'standalone'` — runs as `node server.js`
- `Dockerfile` + `docker-compose.yml`
- Raspberry Pi 4 setup guide (Node install, systemd service, `.env` config)
- Daily SQLite backup cron (`/data/backups/db-YYYY-MM-DD.sqlite`)
- `.env.example` — every variable documented with description

**Deliverable**: Full stack runs on Raspberry Pi 4 with `node server.js`, no cloud services required.

---

### M13 — Reports + Intelligence (ongoing)

- Sales summary: revenue by category, time period, staff member
- Demand report: most-requested uncatalogued items (fuzzy-grouped by description)
- Refund audit: total refunded amount, denial rate, top reasons
- Top repeat customers
- Sourcing batching: group open `sourcing` inquiries by supplier for bulk ordering
- OCR on inquiry images: extract printed part numbers to pre-search catalog
- CSV export for all reports

---

## Seeded Data

### Category Tree

```
Semiconductors       → ICs / Transistors / MOSFETs / Diodes / Voltage Regulators
Passives             → Resistors / Capacitors / Inductors / Crystals / Fuses
Modules & Dev Boards → Arduino / Raspberry Pi / ESP32 / Sensors / Displays
Connectors & Cables  → USB / HDMI / Power / Jumpers / Headers / Audio
Power                → Adapters / Batteries / Chargers / UPS / Solar
Networking           → Routers / Switches / Access Points / SFP / Cables
Peripherals          → Keyboards / Mice / Monitors / Webcams / Headsets
Tools                → Soldering / Multimeters / Oscilloscopes / Hand tools / ESD
```

### Default CMS Blocks

| Key | Type | Default value |
|-----|------|---------------|
| `hero_heading` | text | "Your trusted source for electronics parts" |
| `hero_subheading` | text | "15,000+ parts in stock. Same-day sourcing." |
| `hero_cta_label` | text | "Shop Now" |
| `hero_image` | image | *(blank — owner uploads)* |
| `announcement_text` | text | *(blank)* |
| `announcement_enabled` | boolean | `false` |
| `why_shop_items` | richtext | 4 default bullet points |
| `inquiry_cta_heading` | text | "Can't find what you need?" |
| `inquiry_cta_body` | text | "We'll source it — usually same day." |
| `featured_section_enabled` | boolean | `true` |
| `featured_section_title` | text | "Featured Products" |

---

## Environment Variables

```env
# Adapter profile
ADAPTER_PROFILE=cloud             # cloud | selfhosted | docker

# Firebase (cloud profile)
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=

# Resend (cloud email)
RESEND_API_KEY=
RESEND_FROM_ADDRESS=noreply@newcitytronics.com

# Razorpay (optional)
RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
RAZORPAY_WEBHOOK_SECRET=

# Self-hosted / Docker
DATABASE_URL=file:/data/db.sqlite  # or postgresql://...
JWT_SECRET=                         # 32+ random chars
UPLOAD_DIR=/data/uploads

# Nodemailer (selfhosted email)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=

# LibreTranslate (selfhosted translation — optional)
LIBRE_TRANSLATE_URL=http://localhost:5000

# Typesense (search)
TYPESENSE_HOST=localhost              # cloud: xxx.typesense.net
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http              # cloud: https
TYPESENSE_API_KEY=                   # admin key — server only, never sent to browser
NEXT_PUBLIC_TYPESENSE_SEARCH_ONLY_KEY=  # search-only key — safe for browser instant-suggest

# App
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## Success Metrics (6 months post-launch)

| Metric | Goal |
|--------|------|
| Dashboard open during shop hours | Consistent daily use — primary goal |
| Phone inquiries logged | > 90 % captured |
| Catalog growth | Week-over-week increase driven by real transactions |
| Refund processing time | Under 24 hours from request to decision |
| Time to quote an inquiry | Under 2 minutes |
| Customer return rate | Rising repeat orders / inquiries from same customer |
| Theme customisation | Owner sets brand colours within first week of launch |

