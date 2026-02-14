# Claude Quest - App Status Audit

**Date**: 2026-02-14
**Auditor**: Claude (automated)
**Scope**: Full codebase audit — build, lint, dependencies, security, code quality

---

## Summary

| Area | Status | Details |
|------|--------|---------|
| **Build** | WARN | Fails due to Google Fonts network fetch; not a code defect |
| **Lint** | FAIL | 3 errors, 2 warnings |
| **npm audit** | FAIL | 3 vulnerabilities (1 moderate, 2 high) |
| **Security** | CRITICAL | Paywall trivially bypassable via localStorage or admin email |
| **Type Safety** | WARN | 5 instances of `any` types |
| **Tests** | MISSING | No test framework or test files |
| **Unused Deps** | WARN | 3 installed packages (Stripe, Supabase) not imported anywhere |

---

## 1. Build

**Command**: `npm run build` (Next.js 15.5.12, Turbopack)
**Result**: FAIL — Google Fonts fetch failure

```
next/font: error: Failed to fetch `Geist` from Google Fonts.
next/font: error: Failed to fetch `Geist Mono` from Google Fonts.
```

This is an **environment issue**, not a code bug. The build requires network access to Google Fonts at build time. In CI/CD or offline environments this will break. Consider self-hosting fonts or adding a fallback font stack.

---

## 2. Lint (ESLint)

**Command**: `npm run lint`
**Result**: 3 errors, 2 warnings

| File | Line | Severity | Rule | Message |
|------|------|----------|------|---------|
| `context/ProgressContext.tsx` | 3 | warn | `no-unused-vars` | `useState` imported but never used |
| `context/ProgressContext.tsx` | 186 | error | `prefer-const` | `xp` is never reassigned, use `const` |
| `context/ProgressContext.tsx` | 259 | warn | `no-unused-vars` | `source` assigned but never used |
| `types/index.ts` | 122 | error | `no-explicit-any` | `schema: any` — untyped OpenAPI schema |
| `types/index.ts` | 172 | error | `no-explicit-any` | `payload?: any` — untyped event payload |

---

## 3. Dependency Vulnerabilities

**Command**: `npm audit`
**Result**: 3 vulnerabilities

| Package | Severity | Issue |
|---------|----------|-------|
| `js-yaml` 4.0.0–4.1.0 | Moderate | Prototype pollution in merge (`<<`) |
| `qs` <=6.14.1 | High | arrayLimit bypass enables DoS via memory exhaustion |
| `tar` <=7.5.6 | High | Race condition, arbitrary file overwrite, symlink poisoning |

All fixable via `npm audit fix`.

---

## 4. Security Findings

### CRITICAL: Paywall bypass via localStorage

**File**: `context/ProgressContext.tsx:192-193`

```typescript
const isAdminUser = s.userEmail === 'gabe@onewave-ai.com' || s.userEmail === 'gked21@gmail.com';
const isPaidGate = next >= 1 && s.plan === 'free' && !isAdminUser;
```

The `plan` field (`'free' | 'pro'`) is stored in and read from `localStorage`. A user can open browser DevTools and run:

```js
// Bypass 1: Set plan to pro
let data = JSON.parse(localStorage.getItem('onewave:claude-quest:progress'));
data.plan = 'pro';
localStorage.setItem('onewave:claude-quest:progress', JSON.stringify(data));

// Bypass 2: Set admin email
data.userEmail = 'gabe@onewave-ai.com';
localStorage.setItem('onewave:claude-quest:progress', JSON.stringify(data));
```

This grants full access to all paid levels (1-9) without payment.

**Recommendation**: Move subscription/plan verification to a server-side check (Supabase + Stripe). Never trust client-side plan state for access control.

### MEDIUM: PII in localStorage

User email and license token are stored in plaintext in `localStorage`. This data is accessible to any JavaScript running on the page and persists across sessions.

### CLEAN: No XSS or injection vectors

- No `dangerouslySetInnerHTML` usage
- No `eval()`, `Function()`, or `innerHTML` patterns
- All user input rendered safely through React JSX
- Validation uses function callbacks, not string evaluation

---

## 5. Unused Dependencies

These packages are installed but never imported in any source file:

| Package | Version | Size Impact |
|---------|---------|-------------|
| `@supabase/supabase-js` | ^2.58.0 | Adds auth/DB SDK |
| `@stripe/stripe-js` | ^8.0.0 | Adds payment SDK |
| `stripe` | ^19.1.0 | Adds server-side Stripe |

**Recommendation**: Remove until actually integrated: `npm uninstall @stripe/stripe-js stripe @supabase/supabase-js`

---

## 6. Type Safety

5 instances of `any` types across the codebase:

| File | Line | Context |
|------|------|---------|
| `app/page.tsx` | 42 | `char: any` in CharacterCard props |
| `app/journey/page.tsx` | 122 | `level: any` in LevelCard props |
| `components/AchievementNotifications.tsx` | 23 | Event handler cast `as any` |
| `types/index.ts` | 122 | `schema: any` (OpenAPI schema) |
| `types/index.ts` | 172 | `payload?: any` (event payload) |

---

## 7. Testing

**Status**: No tests exist.

- No test runner installed (no Jest, Vitest, Playwright, etc.)
- No `*.test.*` or `*.spec.*` files
- No test script in `package.json`
- No CI/CD configuration

**Recommendation**: Add at minimum:
- Unit tests for validation logic in `lib/levels.ts`
- Unit tests for achievement unlock conditions in `lib/achievements.ts`
- Component tests for `ProgressContext` state transitions

---

## 8. Code Quality Notes

### Error handling
- localStorage operations have proper try-catch with fallback (good)
- Event dispatch in `ProgressContext` lacks try-catch around handler execution — one failing handler could crash others
- Router navigation (`router.push`) not error-handled

### Console statements
- 4x `console.warn()` in `hooks/useLocalStorage.ts` — acceptable for error paths
- No debug `console.log` statements found in production code

### Architecture
- Clean separation: components, context, hooks, lib, types
- All state centralized in `ProgressContext` with localStorage persistence
- Good use of React patterns (custom hooks, context, callbacks)
- `next.config.ts` is empty — using all defaults

---

## Priority Fix List

| Priority | Issue | Effort |
|----------|-------|--------|
| P0 | Paywall bypass (client-side plan check) | Backend required |
| P0 | Hardcoded admin email bypass | Remove or move server-side |
| P1 | `npm audit fix` for 3 vulnerabilities | One command |
| P1 | Fix 3 ESLint errors | Minutes |
| P2 | Remove unused Stripe/Supabase deps | One command |
| P2 | Replace `any` types with proper interfaces | Hours |
| P2 | Add test framework and initial tests | Days |
| P3 | Self-host fonts for offline builds | Hours |
