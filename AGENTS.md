# AGENTS.md — ufabc-next-extension

## 1. Project Overview

Chrome/Firefox browser extension built with WXT + Vue 3 + TypeScript that injects UI into UFABC's academic portals (enrollment system and SIGAA) and syncs student data to `ufabc-next-backend`. No tests. Uses Biome for linting. Extension permissions: `storage`, `cookies`.

---

## 2. Stack and Commands

| Tool | Version |
|---|---|
| Node.js | >22 |
| pnpm | ^9.15.4 |
| WXT | ^0.19.19 |
| Vue | ^3.5 |
| TypeScript | ^5.5 |

```bash
# Install
pnpm install

# Dev (Chrome, hot reload)
pnpm dev

# Dev (Firefox)
pnpm dev:firefox

# Build
pnpm build             # Chrome MV3
pnpm build:firefox     # Firefox

# Package for submission
pnpm zip
pnpm zip:firefox

# Type check
pnpm compile           # vue-tsc --noEmit

# Lint
pnpm lint              # biome lint ./src

# Format
pnpm fmt               # biome format ./src --write
```

**Loading in browser during dev:**
- Chrome: `chrome://extensions` → Developer mode → Load unpacked → `.output/chrome-mv3`
- Firefox: `about:debugging` → Load Temporary Add-on → `.output/firefox-mv2/manifest.json`

---

## 3. Folder Structure

```
src/
├── entrypoints/        # ONE FILE PER EXTENSION ENTRY POINT — auto-registered by WXT
│   ├── background.ts   # Service worker — cookie extraction only
│   ├── matricula.content/  # Content script for UFABC enrollment portal
│   ├── sig.content/        # Content script for SIGAA
│   ├── moodle.content/     # Content script for Moodle
│   └── popup/              # Toolbar popup
├── components/         # Shared Vue components used by content scripts
├── composables/        # Vue composables (useComponentsBuilder, useFilters, useModals, useStorage)
├── services/
│   ├── next.ts         # All calls to https://api.v2.ufabcnext.com — ADD NEW API CALLS HERE
│   └── ufabc-parser.ts # All calls to https://ufabc-parser.com
├── scripts/sig/        # DOM scraping logic for SIGAA pages
├── utils/              # Pure utility functions
├── assets/             # CSS assets
├── public/             # Extension icons (do not rename — used in manifest)
└── messaging.ts        # Extension messaging protocol — ADD NEW MESSAGES HERE
```

---

## 4. Code Patterns

### Adding a new content script

```typescript
// src/entrypoints/example.content/index.ts
export default defineContentScript({
  matches: ['https://example.ufabc.edu.br/*'],
  runAt: 'document_end',
  async main() {
    // your logic
  },
});
```

Then add `host_permissions: ['https://example.ufabc.edu.br/*']` in `wxt.config.ts`.

### Injecting a Vue UI (shadow DOM)

```typescript
// preferred pattern — isolates CSS from host page
const ui = await createShadowRootUi(ctx, {
  name: 'my-widget',
  position: 'inline',
  anchor: '#target-element',
  append: 'first',
  async onMount(container) {
    const wrapper = document.createElement('div');
    container.append(wrapper);
    const app = createApp(MyComponent);
    app.mount(wrapper);
    return { app, wrapper };
  },
  async onRemove(mounted) {
    const m = await mounted;
    m?.app.unmount();
    m?.wrapper.remove();
  },
});
ui.mount();
```

### Background message handler — correct

```typescript
// src/entrypoints/background.ts
onMessage('myCookieFetch', async ({ data }) => {
  const cookie = await browser.cookies.get({ url: data.pageURL, name: 'CookieName' });
  if (!cookie) throw new Error('Cookie not found');
  return cookie;
});
```

```typescript
// src/messaging.ts — add to ProtocolMap
interface ProtocolMap {
  myCookieFetch({ pageURL }: { pageURL: string }): Cookies.Cookie | null;
}
```

### API call — correct

```typescript
// src/services/next.ts
export async function getWidgets(seasonId: string) {
  return nextService<Widget[]>(`/entities/widgets?season=${seasonId}`);
}
```

### Vue composable — correct

```typescript
// src/composables/useExample.ts
export function useExample() {
  const value = ref<string | null>(null);
  function update(v: string) { value.value = v; }
  return { value, update };
}
```

---

## 5. Database Access Rules

No database access in extension. All data:
- **Remote**: fetched via `services/next.ts` (backend) or `services/ufabc-parser.ts`
- **Local**: stored in WXT storage (`import { storage } from 'wxt/storage'`)

WXT storage keys:
- `local:fullStudent` — `MatriculaStudent` object from backend
- `local:student` — scraped SIGAA student data

---

## 6. Test Requirements

No tests currently exist. If adding tests:
- WXT supports Vitest — add `vitest` to devDependencies
- Unit test composables and utility functions
- Integration testing extension behavior requires browser automation tools (Playwright + WebdriverIO)

---

## 7. Build and Pre-Commit Checklist

```bash
pnpm compile    # vue-tsc must pass with 0 errors
pnpm lint       # biome lint must pass
pnpm build      # must produce .output/chrome-mv3/ without errors
```

CI: `.github/workflows/release.yml` handles automated release/zip creation.

---

## 8. Environment Variables

None. Extension uses hardcoded URLs:
- Backend: `https://api.v2.ufabcnext.com` — defined in `src/services/next.ts`
- Parser: `https://ufabc-parser.com` — defined in `src/services/ufabc-parser.ts`

To switch environments (e.g. for local dev), temporarily edit these constants. Do not commit local override URLs.

---

## 9. Absolute Rules

**NEVER:**
- Access cookies directly from content scripts — always send a message to `background.ts` and use `browser.cookies.get()`
- Inject styles directly into host page DOM — always use shadow DOM via `createShadowRootUi()`
- Add host permissions for domains not in `wxt.config.ts` manifest
- Call `browser.cookies.get()` from content scripts — it does not work in MV3 content scripts

**ALWAYS:**
- Define new background message types in `src/messaging.ts` `ProtocolMap` before implementing
- Use `nextService` (from `services/next.ts`) for backend calls — not raw `fetch` or new `ofetch` instances
- Use `ufParserService` (from `services/ufabc-parser.ts`) for parser calls
- Export default `defineContentScript({...})` from all content script `index.ts` files
- Use WXT's `storage` API, not raw `chrome.storage` or `localStorage`
- Run `pnpm compile` before submitting — catches Vue template type errors

---

## 10. Critical Flows — Do Not Break

| Flow | Location | Risk |
|---|---|---|
| Cookie extraction | `background.ts` + `messaging.ts` | Extension can't authenticate with UFABC systems |
| Student sync on matricula page | `matricula.content/index.ts` → `syncMatriculaStudent` + `updateStudent` | Student data won't update; filters won't reflect reality |
| History sync on SIGAA page | `sig.content/index.ts` → `syncHistoryV2` | Grade history stops syncing to backend |
| Shadow DOM mounting | `matricula.content/index.ts:mountUFABCMatriculaFilters` | Filter UI won't appear on enrollment portal |
| getUFEnrolled transform | `matricula.content/index.ts` (called before mount) | `selected` filter shows wrong components |

---

## 11. Ecosystem Context

| Repo | Relationship |
|---|---|
| `ufabc-next-backend` | Backend API — all data reads/writes go here via `https://api.v2.ufabcnext.com` |
| `ufabc-parser.com` | External scraper — provides current enrollment data (`/enrolled`, `/components`) |
| `ufabc-next-web` | Web platform — complementary UI; shares same backend API |
| `ufabc-next-server` | Legacy backend — no longer used by this extension |

---

## 12. Contributing Workflow

1. Branch from `main`
2. Make changes
3. Run `pnpm compile && pnpm lint && pnpm build`
4. Load extension manually in Chrome to test
5. Open PR against `main`
6. CI builds and zips the extension automatically on merge

**PR requirements:**
- No `@ts-ignore` without explanation comment
- New API calls must be in `services/next.ts` or `services/ufabc-parser.ts`
- New content scripts must have `host_permissions` added in `wxt.config.ts`
- Test manually on target UFABC portal pages before marking ready
