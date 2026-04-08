# Prompt Strategy Document: OTT App Watchlist Feature

**Feature:** Option A — add/remove titles, full Watchlist screen, on-card saved indicator.

---

## 1. Feature breakdown (build order)

1. **Types + API** — `src/types/watchlist.ts` + `src/lib/api/watchlist.ts` (`getWatchlist`, `addToWatchlist`, `removeFromWatchlist`) via `fetchWithAuth` in `src/lib/api/client.ts`; 401/403 like profile/library.
2. **React Query** — `src/hooks/useWatchlist.ts`: query key `['watchlist']` (or user-scoped if library does); add/remove mutations; `useIsTitleWatchlisted(id)` from cached list or id `Set` (no per-card fetches).
3. **Watchlist page** — `src/app/(app)/watchlist/page.tsx`: auth like profile/library; loading/empty/error; grid or list; remove action; no raw `fetch`.
4. **Card toggle** — `src/components/watchlist/WatchlistToggleButton.tsx` on `ContentCard` / `TitleCard`; `titleId`; `stopPropagation`, `type="button"`.
5. **Nav** — Link `/watchlist` from `Header.tsx` / menu / tabs; logged-out behavior matches existing auth.
6. **Polish** — Skeletons like browse; optional pull-to-refresh if present; cache stays consistent after navigation; double-tap/offline like rest of app.

---

## 2. Two full prompts (CDIR)

### Task 2 — React Query watchlist hooks

**Context:** After task 1, `src/lib/api/watchlist.ts` exposes the three functions with `fetchWithAuth`; types in `src/types/watchlist.ts`. Bookmark UI needs “is `titleId` saved?” from one source: TanStack Query cache for `['watchlist']` (or same user-scoped pattern as library). Add/remove must update Watchlist page and cards without reload—invalidate or optimistic cache writes, one strategy for the feature.

**Decomposition:** (1) `useWatchlistQuery` wrapping `useQuery({ queryKey: ['watchlist'], queryFn: getWatchlist })`. (2) Derive id set from data; `useIsTitleWatchlisted` updates when query/mutations change. (3) `useMutation` for add/remove; optimistic updates only if shape matches API; else invalidate. Rollback on error like existing cart/checkout hooks.

**Instructions:** File `src/hooks/useWatchlist.ts` (match sibling hook layout). Exports: `useWatchlistQuery`, `useAddToWatchlist`, `useRemoveFromWatchlist`, `useIsTitleWatchlisted`. Use `useQueryClient` for `setQueryData` / `invalidateQueries`; no parallel `useState` list. Same query key everywhere. Optional pure updaters in `src/lib/watchlist/cacheUpdaters.ts` for tests.

**Risks:** Wrong keys → stale saved state on cards. Half-built optimistic items break the grid. Note idempotency / disable button if double-submit is possible.

### Task 3 — Watchlist screen

**Context:** App Router under `src/app/(app)/`; auth via `src/lib/auth/session.tsx` (or equivalent). Data only through `useWatchlistQuery` and `useRemoveFromWatchlist` from `src/hooks/useWatchlist.ts`. UI from `src/components/ui/`; layout/spacing like `browse/page.tsx` or library. Title `Link` hrefs must match catalog cards and `middleware.ts` (no guessed paths).

**Decomposition:** (1) Auth gate identical to `profile/page.tsx` or `library/page.tsx`. (2) States: spinner, error + refetch, empty, list. (3) Row tap → title detail; remove is separate control calling `useRemoveFromWatchlist`. (4) Remove buttons: `aria-label` includes title name.

**Instructions:** Create `src/app/(app)/watchlist/page.tsx` as `'use client'` if library/browse do user data that way; else match their RSC pattern. No `fetch` in page. Reuse `ContentCard` or add `WatchlistRow`. No new deps. Tests only if project already RTL-tests pages.

**Risks:** Don’t fetch watchlist in RSC without same session story as library. Out of scope: `WatchlistToggleButton` on browse (task 4).

---

## 3. Plan Mode outline (Task 2)

**Expected before coding:** Chosen query key(s); invalidate vs optimistic; cache shape after mutations; how `useIsTitleWatchlisted` stays in sync.

**Ask:** Full `getWatchlist` payload or ids only? Idempotent APIs? Invalidate only `['watchlist']` or catalog too? Pending UI needed?

**Files:** `src/lib/api/watchlist.ts`, `src/types/watchlist.ts`, `src/app/providers.tsx`, a reference `useMutation` with rollback (e.g. cart).

**Steps:** Sample JSON → list UI surfaces → pick cache strategy → list edge cases (empty, duplicate, 409) → manual test card ↔ Watchlist page → implement `useWatchlist.ts` and note exports for tasks 3–4.

---

## 4. `.cursor/rules` additions

| Rule | Why |
|------|-----|
| Membership from TanStack Query `['watchlist']` only — no second `useState` source of truth. | Avoids page vs grid drift. |
| Watchlist HTTP only in `src/lib/api/watchlist.ts`. | One auth/error path. |
| `WatchlistToggleButton` uses `useWatchlist` mutations, not inline `fetch`. | Single fix path for add/remove. |

---

## 5. AI failure anticipation

1. **Per-card queries / fake `getWatchlistItem`.** *Catch:* grep `useQuery` in card components. *Follow-up:* “Drop per-title fetches. `useIsTitleWatchlisted` reads `['watchlist']` cache only; diff limited to hooks + one card file.”
2. **Toggle click navigates card.** *Catch:* manual click test. *Follow-up:* “`type='button'`, `stopPropagation` on the toggle; `Link` must not wrap the toggle.”

---

## 6. One thing I learned

I learned treating the model output like a junior PR meant my checklist (grep for per-card useQuery, click-test the card toggle, logged-out nav) caught issues the prompt didn’t prevent—prompting reduced mistakes; it didn’t replace review.

---
