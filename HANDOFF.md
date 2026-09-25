# S'ovo — Agent Handoff Notes

Read this file first in any new session before touching code. It replaces re-explaining
the project. Update it (don't just append) whenever a listed item's status changes.

## Environment
- Repo: `Thoriii-Gloriii/Chatting-Sovo` (React + Vite + Capacitor Android, Tailwind, Supabase backend)
- Supabase project ref: `wltqcibehtucglflorga`
- Known blocker: the connected GitHub App integration has **read-only** access (confirmed via
  failed `push_files` and `create_pull_request` calls, both `403 Resource not accessible by
  integration`). To let an agent push/PR directly: GitHub → Settings → Applications →
  Installed GitHub Apps → find the Claude/Anthropic app → Configure → ensure this repo is in
  its repository access list and **Contents** + **Pull requests** are set to Read and write.
  Until then, an agent should prepare changes and either ask the user to merge a PR, or give
  the user a compare-branch link to merge manually in the GitHub UI.

## Completed
- Merged `feature/reconciled-updates` → `main` (direct-chat header now resolves live
  per-viewer instead of a baked-in name; StatusReelsView is genuinely full-screen with a
  close button).
- Revoked/flagged an exposed GitHub PAT that had been left in plaintext in an earlier
  session transcript — confirm with the user this was actually revoked if picking this back up.
- **Fixed: messages silently failing to send.** Root cause found and fixed directly in
  Supabase (not app code): `public.conversations` had RLS enabled with SELECT and UPDATE
  policies but **no INSERT policy**, so every new-conversation `upsert`/`insert` from the
  client (starting a direct chat, replying to a status, creating a group) was silently
  rejected by RLS, which meant the subsequent message INSERT also failed its RLS check
  (`EXISTS (... conversations WHERE id = conversationId AND uid IN members)` — the
  conversation row never existed). Confirmed via `postgres_logs` showing repeated
  `new row violates row-level security policy for table "conversations"` immediately
  followed by the same for `"messages"`. Fix applied as migration
  `add_conversations_insert_policy`:
  ```sql
  create policy "members can create their conversations"
  on public.conversations for insert to authenticated
  with check (auth.uid()::text = any(members));
  ```
  This is live in the database already — no app rebuild needed. Should be verified by
  sending a message in a brand-new chat.

## Pending — ranked punch list (user's stated priority order)
User is going through these **one at a time**; don't jump ahead without them saying so.

1. **(tie) Notch / safe-area bleed-through** — Root cause identified, not yet fixed:
   `index.html` has `viewport-fit=cover` but no CSS anywhere uses
   `env(safe-area-inset-*)`. Screenshot the user shared shows real device status-bar/notch
   content bleeding over a full-bleed background (most likely the StatusReelsView story
   viewer — its top overlay row sits at a fixed `top-2`, not safe-area-aware; the close (X)
   button already correctly uses `max(0.5rem, env(safe-area-inset-top))` as a model to copy).
   Fix plan: add safe-area padding to `src/index.css`/`src/styles/tokens.css` globally, and
   pad the StatusReelsView top overlay + bottom reply bar, and ChatRoom/App headers, to
   respect `env(safe-area-inset-top/bottom)`.
2. **(tie) UI not flush to all 4 edges / scroll reveals a black void** — Root cause
   identified, not yet fixed: `App.tsx`'s root div has a leftover inline
   `style={{ zoom: '0.95' }}` (search for `id="sovo-app-root"`) — almost certainly why
   content doesn't reach the true screen edges. Also no `overscroll-behavior: none` or fixed
   `html, body` height anywhere, which is what allows the rubber-band/void on overscroll.
   Fix plan: remove the zoom hack, set `html, body { height: 100%; margin: 0;
   overscroll-behavior: none; }`, use `100dvh` instead of `min-h-screen` where relevant.
3. **(tie) Consolidate scattered E2EE/security badges into a Settings "About" section** —
   Full inventory already taken via GitHub code search:
   - `src/App.tsx` — "Knox E2EE" pill in the shared top header (visible on every tab)
   - `src/components/StatusReelsView.tsx` — small "E2EE" tag next to each story's expiration badge
   - `src/components/ChatRoom.tsx` — persistent encryption banner row below the chat header
     (shows fingerprint), plus a header lock-icon button opening `E2EEVerificationModal`
   - `src/components/CallsView.tsx` — "E2EE Audio & Video" pill
   - `src/components/SettingsView.tsx` — "4096-bit Zero-Knowledge" header pill, and the
     "Export & View 4096-bit Cryptographic Identity" button
   - NOT flagged for removal (contextual, one-time, not persistent chrome): the E2EE note in
     `GroupCreateModal.tsx`, and the E2EE badges on `AuthLanding.tsx`/`SplashScreen.tsx`
     (pre-login marketing, not an in-app window)
   Fix plan: strip the persistent ones listed above, fold their info into one new "About &
   Security" section in Settings.
4. **(tie) Reorganize Settings into categories** — `SettingsView.tsx` (21KB) read in full;
   currently one long always-expanded page. Plan: keep profile card + edit up front and Sign
   Out visible; convert the rest (Privacy & Security toggles, Linked Devices, Storage,
   new About section) into a category list with internal `activeSection` state that expands
   into a detail panel — no App.tsx routing changes needed, self-contained in the component.
5. **Dark/light mode toggle** — Not started. `UserSettings.darkMode` field already exists in
   the type/default settings but nothing reads it. Real blocker: colors are hardcoded hex
   Tailwind arbitrary values (`bg-[#07070b]` etc.) across ~14 components instead of the CSS
   variables already defined in `src/styles/tokens.css` (`--color-bg`, `--color-surface`,
   etc.). Proper fix is a real refactor: add a light-theme override block keyed on
   `[data-theme="light"]`, then migrate components to consume the variables, then wire a
   toggle that sets `data-theme` on `<html>` and persists via `settings.darkMode`.
6. **Forced update / auto-download-and-install mechanism** — Not started, no code written.
   Plan sketched: new Supabase table (e.g. `app_config`: latest_version, apk_url,
   force_update, changelog), version read via `@capacitor/app`'s `App.getInfo()` (not yet a
   dependency — check `package.json`), compare on launch, modal with a download button.
   Honest caveat already given to the user: Android will not allow a sideloaded app to
   silently self-install without at least one OS "install unknown app" confirmation tap —
   true silent one-tap auto-update only happens via Play Store distribution. Needs native
   Android manifest changes (`REQUEST_INSTALL_PACKAGES`, a `FileProvider`), not pure JS.

## How to resume in a new session
Tell the new agent: *"Continue the S'ovo project — read HANDOFF.md at the root of
github.com/Thoriii-Gloriii/Chatting-Sovo, then pick up wherever the ranked list says we
left off."* If Claude's persistent memory is active on the account, it also already holds a
short pointer to this same file under `/areas/sovo-chat-app.md`.
