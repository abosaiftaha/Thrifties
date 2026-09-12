# Capability Map: Thrifties Mobile

> Status: **AWAITING APPROVAL** — module boundaries, dependency direction and build
> order must be reviewed before any module spec is written (spec-driven-development Phase 0).

## Modules

| Module id | Responsibility | Depends on | Phase |
|---|---|---|---|
| `app-shell` | RN scaffold, navigation tree, theming, i18n + RTL, env config, error boundary, crash reporting | — | 1 |
| `design-system` | Design tokens from the mockups (palette, type, spacing, radii) + primitives: Button, Input, Chip, Card, Badge, Sheet, EmptyState | `app-shell` | 1 |
| `identity` | Phone OTP sign-in, session lifecycle, secure token storage, guest browsing, profile read | `app-shell` | 1 |
| `catalog` | Home feed, category nav, search, filters, item detail, seller public profile, saved/wishlist | `design-system`, `identity` (optional session) | 1 |
| `cart-checkout` | Cart, shipping address, payment method selection, order placement | `catalog`, `identity` | 1 |
| `orders` | Buyer order history, status timeline, delivery confirmation, post-delivery rating | `cart-checkout` | 1 |
| `seller-console` | Sell-an-item flow, my listings, sold items, earnings | `identity`, `catalog` | 2 |
| `messaging` | Buyer/seller conversations, contact-leakage detection | `identity`, `orders` | 2 |

## Dependency direction

```
app-shell
   ├── design-system ──┐
   ├── identity ───────┤
   │                   ▼
   │                catalog
   │                   │
   │                   ▼
   │              cart-checkout
   │                   │
   │                   ▼
   │                orders
   │                 │   │
   └─────────────────┘   └──► messaging
                    └──► seller-console
```

No cycles. `catalog` never imports from `cart-checkout`; the cart owns its own
line-item shape and maps from a catalog `Item` at the boundary.

## Build order

```
app-shell → design-system → identity → catalog → cart-checkout → orders
                                                      └─────────► seller-console, messaging (Phase 2, parallel)
```

## Boundary notes

- **`catalog` → `cart-checkout`**: catalog exposes a read-only `Item` contract. Cart
  snapshots price, size and title at add-time; it does not hold a live reference,
  because thrift stock is quantity-of-one and items disappear.
- **`identity` is optional for `catalog`**: browsing must work signed-out. Auth is
  demanded at "Add to cart", not at app open. This is a deliberate cold-start decision.
- **`seller-console` and `messaging` are Phase 2** and deliberately excluded from the
  first shippable slice — see SPEC.md § Out of Scope for the reasoning.

## Open decision blocking the module specs

**Supply model: curated brands vs. peer-to-peer.** The pitch deck describes
admin-approved Instagram thrift *brands*; the mockups show an individual seller with a
self-serve listing flow. `seller-console` and `identity` differ materially between the
two (business verification + admin-uploaded listings vs. open self-serve onboarding).
`app-shell` through `orders` are unaffected, so Phase 1 can proceed while this is
decided.
