# Spec: Thrifties Mobile — `app-shell`

> Module id: `app-shell` (see `CAPABILITY-MAP.md`)
> Status: **AWAITING APPROVAL** — no dependencies beyond the bare scaffold are installed
> until this spec is approved.

---

## Assumptions

These were not stated in the request. Correct any that are wrong before Phase 2.

1. **React Native bare workflow, not Expo.** Explicitly requested. Expo *modules* are
   still installed (via `install-expo-modules`) so we can use `expo-image`; the app is
   not an Expo-managed project and keeps its own `ios/` and `android/` directories.
2. **TypeScript, strict mode.** The scaffold is TS by default; we turn on `strict`.
3. **Both platforms, iOS first.** Mockups are iOS. Android must build from day one but
   visual polish is iOS-led. *(Note: Android has the larger install base in Jordan — if
   Android is actually the primary target, say so now, it changes the QA order.)*
4. **Arabic + English with full RTL.** Jordan market. This is not a Phase 2 nicety —
   retrofitting RTL after the layouts are written is expensive, so it lands in `app-shell`.
5. **Prices in JOD**, 2-decimal minor units stored as integers (fils), formatted at
   the edge. Never float arithmetic on money.
6. **A REST backend exists or will exist**, owned separately. The app talks to it over
   HTTPS with a bearer token. No backend code lives in this repo.
7. **Phone-number + OTP authentication**, not email/password. Standard for the region.
8. **Minimum OS:** the RN 0.87 defaults (iOS 16.0, Android API 24) unless we raise them.

## Pinned versions (as scaffolded)

| | Version |
|---|---|
| React Native | 0.87.1 |
| React | 19.2.3 |
| TypeScript | ^6.0.3 |
| Community CLI | 20.2.0 |
| Node (required) | **>= 22.11.0** — see below |

> ⚠️ **Node version.** RN 0.87 declares `engines.node >= 22.11.0`. The default `node` on
> this machine is v20.20.2, which will produce engine warnings and can break Metro.
> `v23.8.0` is installed under nvm and a `.nvmrc` pinning it is committed. Run
> `nvm use` in the project directory before any yarn/metro command.

---

## Objective

A React Native marketplace app where buyers discover pre-loved fashion from
admin-approved sellers, add items to a cart, and check out on-platform.

`app-shell` specifically delivers the foundation every other module builds on:
a navigable, themed, localised, instrumented app skeleton with no business logic.

**Users:** buyers aged 19–27 in Amman (primary), approved sellers (Phase 2).

**Success for this module:** a developer can clone the repo, run one install command,
launch on both platforms, navigate the full tab structure with placeholder screens,
switch language and see the layout mirror correctly, and have a crash reported to Sentry.

---

## Tech Stack

| Concern | Choice | Why this over the alternative |
|---|---|---|
| Framework | React Native (Community CLI, New Architecture on) | Requested. New Arch is the default from 0.76 on. |
| Language | TypeScript, `strict: true` | Non-negotiable for a team codebase. |
| Navigation | `@react-navigation/native` v7 + `native-stack` + `bottom-tabs` | Matches the 5-tab bar with centre FAB in the mockups. Same library already used in `alinmapay-mobile`. |
| Server state | TanStack Query v5 (`@tanstack/react-query`) | Caching, pagination, retry and stale-while-revalidate for the catalog feed. **Note:** `alinmapay-mobile` uses the legacy `react-query` package — use the TanStack-scoped one here. |
| Client state | Zustand | Only cart, session and filter draft are truly client state. Redux Toolkit is the house alternative — pick RTK if team familiarity outweighs the boilerplate. |
| Styling | `react-native-unistyles` v3 | Already in the house stack; compile-time themes, no re-render on theme switch, first-class RTL. Alternative: NativeWind. The mockups are bespoke, so a component kit (gluestack) would be fought rather than used. |
| Lists | `@shopify/flash-list` | The catalog is an infinite 2-column image grid; `FlatList` drops frames at that cell count. |
| Images | `expo-image` (via `install-expo-modules`) | Disk caching, blurhash placeholders, correct `contentFit`. `react-native-fast-image` is effectively unmaintained. |
| Forms | `react-hook-form` + `zod` | Uncontrolled inputs = fewer re-renders. Zod infers TS types from the schema; Yup (house default) does not, as well. |
| HTTP | `axios` + interceptors | Token refresh and error normalisation in one place. House convention. |
| Secure storage | `react-native-keychain` | Refresh tokens only. |
| Fast storage | `react-native-mmkv` | Synchronous — cart and filters survive cold start without an async gate on first render. |
| i18n | `i18next` + `react-i18next` + `I18nManager` | Arabic/English with RTL mirroring. |
| Sheets | `@gorhom/bottom-sheet` | Filters and size pickers. |
| Gestures/animation | `react-native-gesture-handler`, `react-native-reanimated` | Peer requirement of flash-list and bottom-sheet anyway. |
| Push | `@react-native-firebase/messaging` + `notifee` | Order status *and* the size/brand drop alerts — the single highest-value retention feature. |
| Monitoring | `@sentry/react-native` | Crashes + performance. Non-optional for a payments flow. |
| Env config | `react-native-config` | `.env` per scheme; no secrets committed. |
| Unit/component tests | Jest + `@testing-library/react-native` | Scaffold ships Jest. |
| E2E | Maestro | YAML flows, no native build harness. Detox is more powerful and far more maintenance. |
| Lint/format | ESLint (`@react-native` config) + Prettier + `typescript-eslint` | Scaffold default, extended. |
| Hooks | `husky` + `lint-staged` | Blocks unformatted/untyped commits. |
| CI | GitHub Actions | Typecheck + lint + unit tests on PR. |

**Deliberately excluded from Phase 1:** in-app chat, promoted listings, analytics
dashboards, recommendation engine, courier API integration, any component library.

---

## Commands

```
Install:        yarn install && npx pod-install ios
Start Metro:    yarn start
Run iOS:        yarn ios
Run Android:    yarn android
Typecheck:      yarn tsc --noEmit
Lint:           yarn lint
Lint (fix):     yarn lint --fix
Format:         yarn prettier --write "src/**/*.{ts,tsx}"
Unit tests:     yarn test
Unit + coverage: yarn test --coverage
E2E:            maestro test e2e/
Clean iOS:      cd ios && xcodebuild clean && rm -rf ~/Library/Developer/Xcode/DerivedData
Clean Android:  cd android && ./gradlew clean
Reset Metro:    yarn start --reset-cache
```

---

## Project Structure

Feature-first, one directory per module id in the capability map. A feature owns its
screens, components, hooks, API calls and types; anything two features need moves up
to `src/shared`.

```
Thrifties/
├── android/                    → native Android project
├── ios/                        → native iOS project
├── e2e/                        → Maestro flows (.yaml)
├── src/
│   ├── app/                    → composition root
│   │   ├── App.tsx             → providers, only providers
│   │   ├── navigation/         → RootNavigator, TabNavigator, linking, param lists
│   │   └── providers/          → QueryProvider, ThemeProvider, I18nProvider
│   ├── features/
│   │   ├── identity/
│   │   ├── catalog/
│   │   ├── cart-checkout/
│   │   ├── orders/
│   │   ├── seller-console/     → Phase 2
│   │   └── messaging/          → Phase 2
│   │       ├── screens/        → one file per screen
│   │       ├── components/     → feature-local only
│   │       ├── hooks/          → useCart, useItemQuery, …
│   │       ├── api/            → endpoint fns + query keys
│   │       ├── store/          → zustand slice, if any
│   │       └── types.ts
│   ├── shared/
│   │   ├── ui/                 → design-system primitives (Button, Chip, Card…)
│   │   ├── hooks/
│   │   ├── utils/              → money.ts, date.ts, …
│   │   └── types/
│   ├── theme/                  → tokens.ts, unistyles.ts, light/dark themes
│   ├── api/                    → axios client, interceptors, error mapping
│   ├── i18n/                   → index.ts, locales/en.json, locales/ar.json
│   └── config/                 → env accessors, constants
├── __tests__/                  → cross-cutting tests; unit tests co-locate as *.test.ts
├── SPEC.md
├── CAPABILITY-MAP.md
└── tasks/                      → plan.md + todo.md (Phase 2/3 output)
```

**Path aliases** (`babel-plugin-module-resolver` + `tsconfig` paths):
`@app/*`, `@features/*`, `@shared/*`, `@theme/*`, `@api/*`, `@i18n/*`, `@config/*`.
No `../../../` imports.

---

## Code Style

```tsx
// src/features/catalog/components/ItemCard.tsx
import { Pressable, View } from 'react-native';
import { Image } from 'expo-image';
import { StyleSheet } from 'react-native-unistyles';

import { Text } from '@shared/ui/Text';
import { formatJod } from '@shared/utils/money';
import type { CatalogItem } from '../types';

type ItemCardProps = {
  item: CatalogItem;
  onPress: (id: string) => void;
};

export function ItemCard({ item, onPress }: ItemCardProps) {
  return (
    <Pressable style={styles.card} onPress={() => onPress(item.id)}>
      <Image
        source={item.imageUrl}
        placeholder={{ blurhash: item.blurhash }}
        contentFit="cover"
        style={styles.image}
      />
      <View style={styles.meta}>
        <Text variant="body" numberOfLines={1}>
          {item.title}
        </Text>
        <Text variant="price">{formatJod(item.priceFils)}</Text>
      </View>
    </Pressable>
  );
}

const styles = StyleSheet.create((theme) => ({
  card: {
    borderRadius: theme.radii.lg,
    backgroundColor: theme.colors.surface,
    overflow: 'hidden',
  },
  image: { aspectRatio: 1, width: '100%' },
  meta: { gap: theme.spacing.xs, padding: theme.spacing.sm },
}));
```

**Conventions**

- Named exports only. No `export default` except screens registered by the navigator.
- Components: `PascalCase.tsx`. Hooks: `useThing.ts`. Utils: `camelCase.ts`.
- `function` declarations for components, arrow functions for callbacks.
- `type` over `interface` unless declaration merging is genuinely needed.
- Props type declared immediately above the component, named `<Component>Props`.
- No inline styles, no magic colours, no magic spacing — everything from `theme`.
- No literal user-facing strings in components; `t('catalog.emptyState')`.
- Money is `priceFils: number` (integer) end to end; formatted only for display.
- Use logical layout props (`marginStart`/`marginEnd`, `start`/`end`) so RTL mirrors
  automatically. `marginLeft`/`marginRight` are a lint error.

---

## Testing Strategy

| Level | Tool | Location | Covers |
|---|---|---|---|
| Unit | Jest | `src/**/*.test.ts` co-located | Money formatting, cart maths, filter/query-key builders, validation schemas |
| Component | Jest + `@testing-library/react-native` | `src/**/*.test.tsx` co-located | Renders from props, fires callbacks, empty/loading/error states |
| Integration | Jest + MSW | `__tests__/integration/` | A screen against mocked HTTP — pagination, retry, error mapping |
| E2E | Maestro | `e2e/*.yaml` | The four critical flows below |

**Critical E2E flows** (these must pass before any release build):
1. Browse → filter by size → open item → add to cart → checkout → order confirmed
2. Sign in with phone OTP
3. Cart survives app cold start
4. Language switch to Arabic mirrors the layout and keeps navigation working

**Coverage:** 80% lines on `src/shared/utils` and every `store/` slice (pure logic, cheap
to test, expensive to get wrong). No global coverage gate — it just produces tests
written to satisfy a number. Every bug fix ships with a regression test (see
`test-driven-development`, Prove-It pattern).

---

## Boundaries

**Always**
- Run `yarn tsc --noEmit && yarn lint && yarn test` before committing.
- Add both `en.json` and `ar.json` keys in the same commit as the UI that uses them.
- Use theme tokens and logical layout props.
- Write money as integer fils.
- Verify a new UI on both platforms **and** in RTL before calling it done.
- Keep `SPEC.md` and `CAPABILITY-MAP.md` current when a decision changes.

**Ask first**
- Adding any dependency (especially one with native code — it costs build time forever).
- Changing navigation structure or route param types.
- Changing the API contract or error shape.
- Bumping the React Native version, or touching `Podfile` / `build.gradle`.
- Adding a third state manager, or moving state between Zustand and TanStack Query.
- Enabling Hermes/JSC, New Architecture, or minSdk/deployment-target changes.

**Never**
- Commit `.env`, keystores, `.p12`/provisioning profiles, or API keys.
- Hardcode colours, spacing, or user-facing strings.
- Use `any` (use `unknown` and narrow) or `@ts-ignore` without an adjacent comment
  explaining why, and a linked issue.
- Edit files under `node_modules/`, `ios/Pods/`, or generated dirs — use `patch-package`.
- Skip or delete a failing test to get green.
- Store tokens in AsyncStorage or MMKV — Keychain only.
- Log PII or full API responses in production builds.

---

## Success Criteria

`app-shell` is done when **all** of these are true:

1. `yarn ios` and `yarn android` both launch on a clean clone after `yarn install && npx pod-install ios`.
2. `yarn tsc --noEmit`, `yarn lint`, and `yarn test` all exit 0.
3. The 5-tab bar from the mockups renders with placeholder screens, and every tab is reachable.
4. A stack push and pop work with typed route params (no `any` in the param list).
5. Toggling language to Arabic mirrors layout direction and the tab bar without a crash, and persists across a cold start.
6. Theme tokens are read from `src/theme/tokens.ts`; the five palette colours from the mockups are the only colours defined.
7. A deliberately thrown error appears in Sentry with a readable source-mapped stack.
8. CI runs typecheck + lint + unit tests on every PR and blocks merge on failure.
9. Path aliases resolve in both Metro and `tsc` (no relative imports crossing feature boundaries).

---

## Open Questions

1. **Supply model — blocking for `seller-console` and `identity`.** Curated
   admin-approved Instagram brands (per the pitch deck) or open peer-to-peer
   self-serve (per the mockups)? Phase 1 modules can proceed either way.
2. **Cash on Delivery.** The checkout mockup pre-selects COD. If Thrifties never holds
   the money, the commission model is unenforceable. Is COD in or out for v1?
3. **Multi-seller cart.** The cart mockup shows three items from three sellers. That
   commits us to split shipping, split payouts and partial refunds. Single-seller
   orders in v1 would remove a large amount of work — confirm which.
4. **Android or iOS first** for QA and store submission?
5. **Backend:** does one exist, and is there an API contract/OpenAPI spec to generate
   types from? If not, `catalog` needs a mock server (MSW) in Phase 1.
6. **App name / bundle id:** scaffolded as `Thrifties` / `com.thrifties.app`. Confirm
   before the Apple and Google listings are created — renaming later is painful.
