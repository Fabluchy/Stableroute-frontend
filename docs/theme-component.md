# Theme Component

The `ThemeToggle` component (`src/components/ThemeToggle.tsx`) and the
theme module (`src/lib/theme.ts`) together provide the app's light/dark/system
appearance switching. This document defines the public API so consumers can
integrate theme selection consistently.

## ThemeToggle component

`ThemeToggle` is a client-side, zero-prop component that renders a segmented
button group for picking the active theme.

### Props

`ThemeToggle` accepts **no props**. It is entirely self-contained:

- Reads the active theme from `localStorage` via `readTheme()`.
- Writes selections back to `localStorage` via `writeTheme()`.
- Syncs the `dark` CSS class on `<html>` so Tailwind's `dark:` utilities work
  without a page reload.

### States

| State         | Behaviour                                                                                             |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| **Idle**      | Resolved to `"light"` on first render (SSR-safe default). Syncs from `localStorage` in an effect.     |
| **Selected**  | The active button receives `aria-pressed="true"` and a filled background (`bg-black text-white` in light mode; inverted in dark). |
| **Corrupted** | If `localStorage` contains an unknown value, `readTheme` falls back to `"system"` and the component behaves normally. |
| **Disabled storage** | If `localStorage` is unavailable (private browsing, quota exceeded), the write is silently skipped and the in-memory state still updates so the UI keeps working. |

### Rendering

Renders three `<button>` elements — `light`, `dark`, `system` — inside a
`<div role="group" aria-label="Theme">`. Each button:

- Has `aria-pressed` reflecting the active selection.
- Carries the shared focus ring convention
  (see [STYLEGUIDE.md](./STYLEGUIDE.md#focus-rings)).
- Uses `rounded-full px-3 py-1 text-xs` for consistent sizing.

### Side effects

On mount and whenever the selected theme changes:

1. Calls `effectiveTheme(theme)` to resolve the concrete `"light"` or `"dark"`
   value.
2. Toggles `document.documentElement.classList.toggle('dark', …)` so Tailwind's
   `dark:` variant activates immediately.

### Minimal usage

```tsx
import { ThemeToggle } from '@/components/ThemeToggle';

function SettingsPage() {
  return (
    <section>
      <h2>Appearance</h2>
      <ThemeToggle />
    </section>
  );
}
```

`ThemeToggle` is already used in:

- `src/components/Header.tsx` — renders in the global header on every page.
- `src/app/settings/Client.tsx` — renders in the Appearance section.

---

## Theme module (`src/lib/theme.ts`)

The module exports a small set of pure helpers for reading, writing, and
resolving the theme preference. Use these when you need direct access to the
stored value (e.g., for an `AppearancePreview`) rather than the rendered
toggle.

### `Theme` type

```typescript
export type Theme = 'light' | 'dark' | 'system';
```

The only three valid values. Any other string stored in `localStorage` is
treated as invalid and causes a fallback to `"system"`.

### `THEME_KEY`

```typescript
export const THEME_KEY = 'stableroute.theme';
```

The `localStorage` key. Must stay in sync with `public/theme-init.js`, which
runs before React hydrates and cannot import this module.

### `isTheme(value: unknown): value is Theme`

Type guard. Returns `true` only for `'light'`, `'dark'`, or `'system'`.

```typescript
isTheme('light'); // true
isTheme('midnight'); // false
isTheme(null); // false
```

### `readTheme(): Theme`

Reads the stored preference from `localStorage`. Returns `"system"` when:

- The key is absent.
- The stored value fails the `isTheme` check.
- `localStorage` is unavailable (SSR, privacy mode).
- Deserialization throws.

### `writeTheme(theme: Theme): void`

Writes the preference to `localStorage`. Best-effort: silently skips the write
when `localStorage` rejects access so persistence cannot crash rendering.

### `effectiveTheme(theme: Theme): 'light' | 'dark'`

Resolves a `Theme` value to the concrete `'light'` or `'dark'` that should be
applied:

| Input      | Result                                               |
| ---------- | ---------------------------------------------------- |
| `"light"`  | `"light"`                                            |
| `"dark"`   | `"dark"`                                             |
| `"system"` | `"light"` or `"dark"` depending on `window.matchMedia('(prefers-color-scheme: dark)')` |

On the server (`typeof window === 'undefined'`), `"system"` resolves to
`"light"`.

### Usage example (without ThemeToggle)

```tsx
import { readTheme, effectiveTheme, type Theme } from '@/lib/theme';
import { useEffect, useState } from 'react';

function AppearancePreview() {
  const [theme, setTheme] = useState<Theme>('system');

  useEffect(() => {
    setTheme(readTheme());
    const handler = () => setTheme(readTheme());
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  }, []);

  const resolved = effectiveTheme(theme);

  return (
    <div data-resolved-theme={resolved}>
      Current appearance: {resolved}
    </div>
  );
}
```

---

## Related

- [`docs/theme-storage.md`](./theme-storage.md) — the `localStorage` contract
  and resolution rules.
- [`docs/STYLEGUIDE.md`](./STYLEGUIDE.md) — focus ring conventions used by
  `ThemeToggle`.
- [`src/lib/theme.ts`](../src/lib/theme.ts) — source for the theme module.
- [`src/components/ThemeToggle.tsx`](../src/components/ThemeToggle.tsx) — source
  for the toggle component.
- [`src/lib/useLocalStorage.ts`](../src/lib/useLocalStorage.ts) — the
  `useLocalStorage` hook and serializers that back the theme storage.
