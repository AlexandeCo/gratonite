# Phase 8: Web App Polish — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Fix all broken/unstyled Phase 7A components and add search, threads, settings, and composer integration — making the web client feature-complete.

**Architecture:** All frontend work (React + Zustand + TanStack Query). Backend APIs for search, threads, reactions already exist. New CSS goes into the single `styles.css` file. New stores follow Zustand pattern. API client methods added to `lib/api.ts`.

**Tech Stack:** React 18, Zustand, TanStack Query, React Router v6, pure CSS custom properties

---

## Context

Phase 7A created ~12 components (MessageActionBar, ReactionBar, EmojiPicker, PinnedMessagesPanel, AttachmentDisplay, FileUploadButton, ReplyPreview, AttachmentPreview, DeleteMessageModal, EditProfileModal, EditServerProfileModal) but shipped them **without any CSS rules** in `styles.css`. These components render invisible/broken HTML. Additionally, `MessageList` doesn't pass required `onReply`/`onOpenEmojiPicker` props to `MessageItem`, and the `EmojiPicker` search is broken (always returns all emojis). The backend has full search and thread APIs that have zero frontend UI.

---

## Task 1: CSS Foundation — Style All Unstyled Phase 7A Components

**Files:**
- Modify: `apps/web/src/styles.css` (append ~300 lines before the responsive section at ~line 2248)

**What to build:**
Add CSS rules for every unstyled Phase 7A component. All class names already exist in the JSX — this task ONLY adds CSS, no component changes.

CSS blocks needed (use existing design tokens: `--bg-float`, `--bg-soft`, `--bg-elevated`, `--stroke`, `--accent`, `--text-muted`, `--text-faint`, `--danger`, `--radius-sm/md/lg/pill`):

1. **MessageActionBar** (`.message-action-bar`, `.message-action-btn`, `.message-action-btn-danger`): Floating toolbar on message hover. `position: absolute; top: -16px; right: 16px; z-index: 10`. Background `var(--bg-float)`, border, shadow, horizontal flex, gap 2px. Buttons 28px square, icon-only, rounded. Danger hover → red.
   - Also add `position: relative` to `.message-item` so absolute positioning works.

2. **ReactionBar** (`.reaction-bar`, `.reaction-pill`, `.reaction-pill-active`, `.reaction-emoji`, `.reaction-count`, `.reaction-pill-add`): Horizontal flex-wrap below message content. Pills: inline-flex, gap 4px, padding 2px 8px, border, rounded-pill, 13px font. Active variant: accent-tinted border/background. Add button: same pill shape with + icon.

3. **EmojiPicker** (`.emoji-picker`, `.emoji-picker-header`, `.emoji-picker-search`, `.emoji-picker-grid`, `.emoji-picker-category`, `.emoji-picker-items`, `.emoji-picker-item`): Fixed popup, 320px wide, max-height 380px. Background `var(--bg-float)`, z-index 220. Search input in header. Grid scrolls. Items: 8-column grid, 36px cells, 22px font. Hover highlight.

4. **PinnedMessagesPanel** (`.pinned-panel`, `.pinned-panel-header`, `.pinned-panel-title`, `.pinned-panel-close`, `.pinned-panel-list`, `.pinned-panel-item`, `.pinned-panel-item-header`, `.pinned-panel-item-author`, `.pinned-panel-item-time`, `.pinned-panel-item-content`, `.pinned-panel-item-unpin`, `.pinned-panel-empty`, `.pinned-panel-loading`): Right-side panel, 340px wide. Background `var(--bg-elevated)`, border-left. Header flex-between. Items with bottom border, padding. Unpin button ghost-style.

5. **AttachmentDisplay** (`.attachment-display`, `.attachment-image-link`, `.attachment-image`, `.attachment-file`, `.attachment-file-info`, `.attachment-file-name`, `.attachment-file-size`): Below content. Images: max 400x300px, border-radius, object-fit. Files: horizontal flex, background soft, border, padding, icon+text layout.

6. **FileUploadButton** (`.file-upload-btn`): Icon button in/beside composer. Background none, color muted, padding 8px. Hover → text color.

7. **ReplyPreview** (`.reply-preview`, `.reply-preview-content`, `.reply-preview-label`, `.reply-preview-author`, `.reply-preview-snippet`, `.reply-preview-close`): Bar above composer. Background `var(--bg-soft)`, left border 3px accent, padding 8px 12px, flex row. Close button right-aligned.

8. **Message Reply Header** (`.message-reply-header`, `.message-reply-author`, `.message-reply-snippet`): Inline above message body. Small text (12px), muted color, flex row with gap, arrow icon. Author bold, snippet truncated ellipsis.

9. **Message Edit** (`.message-edit-wrapper`, `.message-edit-input`, `.message-edit-actions`, `.message-edit-hint`): Inline editing area. Input styled like composer but inline. Hint: 11px, faint color. Also `.message-item-editing` background highlight.

10. **Delete Modal Preview** (`.message-preview-box`, `.message-preview-author`, `.message-preview-content`): Inside delete confirmation modal. Background soft, border, rounded, padding 12px.

**Commit:** `feat: add CSS for all Phase 7A unstyled components`

---

## Task 2: Fix MessageList Prop Wiring + Composer Reply/Upload Integration

**Files:**
- Modify: `apps/web/src/components/messages/MessageList.tsx` — accept + pass `onReply`, `onOpenEmojiPicker` props
- Modify: `apps/web/src/pages/ChannelPage.tsx` — wire reply state, emoji picker state, render EmojiPicker
- Modify: `apps/web/src/components/messages/MessageComposer.tsx` — integrate ReplyPreview, FileUploadButton, reply-on-send

**What to build:**

1. **MessageList.tsx**: Add `onReply` and `onOpenEmojiPicker` to `MessageListProps`. Pass them through to every `<MessageItem>`.

2. **ChannelPage.tsx**:
   - Add `useState` for `emojiTarget: { messageId: string; x: number; y: number } | null`
   - Create `handleReply(msg)` → calls `useMessagesStore.getState().setReplyingTo(channelId, msg)` (store action already exists)
   - Create `handleOpenEmojiPicker(messageId)` → sets emoji target state
   - Pass both to `<MessageList>`
   - Render `<EmojiPicker>` when `emojiTarget` is set, positioned at target coords. On select: call `api.messages.addReaction(channelId, emojiTarget.messageId, emoji)`, close picker.

3. **MessageComposer.tsx**:
   - Read `replyingTo` from `useMessagesStore` for current channelId: `const replyingTo = useMessagesStore((s) => s.replyingTo.get(channelId))`
   - If replying, render `<ReplyPreview message={replyingTo} onCancel={() => setReplyingTo(channelId, null)} />` above the textarea
   - On send: include `messageReference: { messageId: replyingTo.id }` in API call body, then clear reply state
   - Add `<FileUploadButton>` before the textarea (or inside `.message-composer`)
   - Add file state management: `const [pendingFiles, setPendingFiles] = useState<File[]>([])`. On files selected, add to pending. On send, use `FormData` with `api.messages.sendWithAttachments()` if files exist.
   - Render `<AttachmentPreview>` above textarea when files are pending (check if component exists, may need to create simple preview)

**Key existing code:**
- `useMessagesStore` already has `replyingTo: Map<string, Message | null>`, `setReplyingTo(channelId, message)` actions
- `api.messages.send(channelId, { content, nonce })` exists — need to check if it supports `messageReference`
- `ReplyPreview` component exists at `components/messages/ReplyPreview.tsx`
- `FileUploadButton` exists at `components/messages/FileUploadButton.tsx`

**Commit:** `fix: wire MessageList props, integrate reply/upload in composer`

---

## Task 3: Fix Emoji Picker Search + Positioning

**Files:**
- Modify: `apps/web/src/components/ui/EmojiPicker.tsx` — fix search, add position props

**What to build:**

1. **Search fix**: Create a static `EMOJI_NAMES` map at the top of the file mapping each emoji to a searchable name string (e.g., `{ '\u{1F600}': 'grinning face', '\u{1F44D}': 'thumbs up' }`). Cover all ~177 emojis in the existing categories. Replace the broken `allEmojis.filter(() => true)` with `allEmojis.filter(e => EMOJI_NAMES[e]?.includes(search.toLowerCase()))`.

2. **Positioning**: Add optional `x?: number`, `y?: number` props to `EmojiPickerProps`. When provided, use `position: fixed; left: x; top: y` with viewport bounds clamping (similar pattern to existing `ContextMenu` component). When not provided, use default positioning.

3. **Category tab bar** (optional): Add clickable category names in the header that scroll to that section. Low priority — skip if complex.

**Commit:** `fix: emoji picker search and positioning`

---

## Task 4: Reaction Gateway Events in SocketProvider

**Files:**
- Modify: `apps/web/src/providers/SocketProvider.tsx` — add MESSAGE_REACTION_ADD/REMOVE handlers

**What to build:**

The backend emits `MESSAGE_REACTION_ADD` and `MESSAGE_REACTION_REMOVE` gateway events but the SocketProvider doesn't handle them. Currently reactions only update locally via optimistic UI in `MessageItem.handleReactionToggle`, meaning other users' reactions never appear in real-time.

1. Add handler for `MESSAGE_REACTION_ADD`:
   ```ts
   socket.on('MESSAGE_REACTION_ADD', (data: { channelId: string; messageId: string; emoji: string; userId: string }) => {
     useMessagesStore.getState().addReaction(data.channelId, data.messageId, data.emoji, data.userId);
   });
   ```

2. Add handler for `MESSAGE_REACTION_REMOVE`:
   ```ts
   socket.on('MESSAGE_REACTION_REMOVE', (data: { channelId: string; messageId: string; emoji: string; userId: string }) => {
     useMessagesStore.getState().removeReaction(data.channelId, data.messageId, data.emoji, data.userId);
   });
   ```

3. **Verify payload shape**: Check `apps/api/src/modules/messages/messages.router.ts` PUT/DELETE reaction endpoints to confirm the exact event data shape matches.

**Key existing code:**
- `useMessagesStore` already has `addReaction(channelId, messageId, emoji, userId)` and `removeReaction(...)` actions (lines 167-218 of `stores/messages.store.ts`)

**Commit:** `fix: handle reaction gateway events for real-time sync`

---

## Task 5: Pinned Messages Panel Integration

**Files:**
- Modify: `apps/web/src/components/layout/TopBar.tsx` — add pin button for guild channels
- Modify: `apps/web/src/pages/ChannelPage.tsx` — render PinnedMessagesPanel conditionally

**What to build:**

The `PinnedMessagesPanel` component and `usePinnedMessages` hook exist and will be styled after Task 1. Wire them into the actual UI.

1. **TopBar.tsx**: Add a "Pinned Messages" button (pin icon) in `topbar-actions` for guild channels (when `!isDm`). Clicking calls `useUiStore.getState().togglePinnedPanel()`. The `pinnedPanelOpen` and `togglePinnedPanel` already exist in `ui.store.ts`.

2. **ChannelPage.tsx**: Read `pinnedPanelOpen` from `useUiStore`. When true, render `<PinnedMessagesPanel channelId={channelId} />` as an overlay panel on the right side of the message area. Use absolute positioning or a flex layout adjustment.

**Commit:** `feat: wire pinned messages panel into channel view`

---

## Task 6: Search UI — API Client, Store, and Panel

**Files:**
- Create: `apps/web/src/stores/search.store.ts`
- Create: `apps/web/src/components/search/SearchPanel.tsx`
- Modify: `apps/web/src/lib/api.ts` — add `search` namespace
- Modify: `apps/web/src/styles.css` — add search panel CSS
- Modify: `apps/web/src/components/layout/TopBar.tsx` — add search button
- Modify: `apps/web/src/stores/ui.store.ts` — add `searchPanelOpen` state
- Modify: `apps/web/src/pages/ChannelPage.tsx` — render SearchPanel

**What to build:**

Backend already has `GET /search/messages` with query, guildId, channelId, authorId, before/after, limit, offset.

1. **api.ts**: Add `search.messages(params)` method calling `GET /api/v1/search/messages`.

2. **search.store.ts**: Zustand store: `query: string`, `results: Message[]`, `isSearching: boolean`, `totalCount: number`. Actions: `search(params)`, `clearSearch()`, `loadMore()`.

3. **ui.store.ts**: Add `searchPanelOpen: boolean` and `toggleSearchPanel()`.

4. **TopBar.tsx**: Add magnifying glass button in `topbar-actions` (for both DM and guild channels). Clicking toggles search panel.

5. **SearchPanel.tsx**: Right-side overlay panel (similar to pinned panel pattern):
   - Search input with debounced query (300ms)
   - Results list: each result shows author avatar, display name, timestamp, channel name, and content snippet with highlighted matching text
   - Click result → navigate to that channel via `useNavigate`
   - "Load more" button at bottom for pagination
   - Empty state when no results

6. **styles.css**: Add search panel CSS (`.search-panel`, `.search-panel-header`, `.search-input`, `.search-results`, `.search-result-item`, `.search-result-author`, `.search-result-content`, `.search-result-meta`, `.search-highlight`, `.search-empty`).

**Commit:** `feat: add search UI with panel, store, and API client`

---

## Task 7: Thread UI — API Client, Store, and Panel

**Files:**
- Create: `apps/web/src/components/threads/ThreadPanel.tsx`
- Create: `apps/web/src/hooks/useThreads.ts`
- Modify: `apps/web/src/lib/api.ts` — add `threads` namespace
- Modify: `apps/web/src/styles.css` — add thread panel CSS
- Modify: `apps/web/src/components/messages/MessageItem.tsx` — add "Create Thread" to context menu
- Modify: `apps/web/src/stores/ui.store.ts` — add `threadPanelOpen`, `activeThreadId`
- Modify: `apps/web/src/providers/SocketProvider.tsx` — add THREAD_CREATE/UPDATE/DELETE handlers
- Modify: `apps/web/src/pages/ChannelPage.tsx` — render ThreadPanel

**What to build:**

Backend has full thread CRUD: POST/GET/PATCH/DELETE threads, GET/PUT/DELETE thread members. Threads ARE channels on the backend, so messages inside threads use the standard `/channels/:threadId/messages` endpoint.

1. **api.ts**: Add `threads` namespace:
   - `threads.create(channelId, { name, type?, autoArchiveDuration?, message? })` → POST `/channels/:channelId/threads`
   - `threads.list(channelId)` → GET `/channels/:channelId/threads`
   - `threads.get(threadId)` → GET `/threads/:threadId`
   - `threads.join(threadId)` → PUT `/threads/:threadId/members/@me`
   - `threads.leave(threadId)` → DELETE `/threads/:threadId/members/@me`

2. **useThreads.ts**: TanStack Query hook wrapping `api.threads.list(channelId)`.

3. **ui.store.ts**: Add `threadPanelOpen: boolean`, `activeThreadId: string | null`, `openThread(threadId)`, `closeThread()`.

4. **ThreadPanel.tsx**: Right-side panel showing thread content. Reuses `<MessageList>` and `<MessageComposer>` by passing `activeThreadId` as the `channelId` prop (threads ARE channels). Shows thread header with name, member count, close button. Also shows a thread list view when no specific thread is open.

5. **MessageItem.tsx**: Add "Create Thread" to `contextItems` array (only for guild channels). Opens a small inline prompt for thread name, calls `api.threads.create(channelId, { name, message: { id: message.id } })`, then opens the thread panel.

6. **SocketProvider.tsx**: Handle `THREAD_CREATE`, `THREAD_UPDATE`, `THREAD_DELETE` gateway events. Invalidate threads query cache on these events.

7. **styles.css**: Thread panel CSS (`.thread-panel`, `.thread-panel-header`, `.thread-panel-title`, `.thread-panel-close`, `.thread-panel-body`, `.thread-list`, `.thread-list-item`, `.thread-list-item-name`, `.thread-list-item-meta`).

**Commit:** `feat: add thread UI with panel, API client, and gateway events`

---

## Task 8: User Settings Page

**Files:**
- Create: `apps/web/src/pages/SettingsPage.tsx`
- Modify: `apps/web/src/App.tsx` — add `/settings` route
- Modify: `apps/web/src/styles.css` — add settings CSS
- Modify: `apps/web/src/components/sidebar/UserBar.tsx` — add gear icon linking to settings

**What to build:**

1. **Route**: Add `/settings` in App.tsx inside `RequireAuth`.

2. **SettingsPage.tsx**: Full-viewport overlay (like Discord settings — covers the entire app). Left sidebar with nav sections, right content area. Close button (X / Escape) navigates back. Sections:
   - **My Account**: Shows username, email, display name. "Edit Profile" button opens existing `EditProfileModal`. Display current avatar/banner.
   - **Appearance**: Theme selection placeholder (dark-only for now), font scale slider (reads/writes `api.users.updateSettings({ fontScale })`), message display density toggle.
   - **Notifications**: DND schedule UI (uses existing `api.users.getDndSchedule()` / `api.users.updateDndSchedule()`). Toggle DND, set start/end time, timezone, days.
   - **Log Out**: Calls `useAuthStore.getState().logout()`.

3. **UserBar.tsx**: Add a gear/cog icon button that navigates to `/settings` via `useNavigate`.

4. **styles.css**: Settings layout CSS (`.settings-page`, `.settings-sidebar`, `.settings-nav`, `.settings-nav-item`, `.settings-nav-item-active`, `.settings-content`, `.settings-section`, `.settings-section-title`, `.settings-close`, `.settings-field`, `.settings-field-label`, `.settings-field-value`).

**Commit:** `feat: add user settings page with account, appearance, notifications`

---

## Task 9: Final Integration + PROGRESS.md Update

**Files:**
- Modify: `PROGRESS.md` — document Phase 8 completion
- Potentially fix bugs found during integration

**What to build:**
1. Review all Task 1-8 changes for integration issues
2. Ensure EmojiPicker, SearchPanel, ThreadPanel, PinnedMessagesPanel don't conflict when multiple panels are open (should auto-close others or stack)
3. Add responsive media query rules for new panels (hide on mobile)
4. Update PROGRESS.md with Phase 8 summary: what was built, what's next

**Commit:** `docs: update PROGRESS.md with Phase 8 completion`

---

## Task Dependency Order

```
Task 1 (CSS) → Task 2 (Prop Wiring + Composer) → Task 3 (Emoji Fix)
                                                 → Task 4 (Reaction Gateway) — independent
Task 1 → Task 5 (Pinned Panel Integration)
Task 1 → Task 6 (Search UI)
Task 1 + Task 2 → Task 7 (Thread UI)
Task 1 → Task 8 (Settings Page)
All → Task 9 (Integration + PROGRESS.md)
```

**Execution order:** 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9

---

## Critical Files Reference

| File | Role |
|------|------|
| `apps/web/src/styles.css` | All CSS (~2264 lines, append new rules) |
| `apps/web/src/components/messages/MessageList.tsx` | Prop wiring fix needed (line 107) |
| `apps/web/src/components/messages/MessageItem.tsx` | Message display, context menu, hover bar |
| `apps/web/src/components/messages/MessageComposer.tsx` | Composer — needs reply/upload integration |
| `apps/web/src/pages/ChannelPage.tsx` | Main integration point for panels/pickers |
| `apps/web/src/lib/api.ts` | API client — add search + threads namespaces |
| `apps/web/src/stores/ui.store.ts` | UI state — add searchPanelOpen, threadPanelOpen |
| `apps/web/src/stores/messages.store.ts` | Already has replyingTo, addReaction, removeReaction |
| `apps/web/src/providers/SocketProvider.tsx` | Gateway events — add reaction + thread handlers |
| `apps/web/src/App.tsx` | Router — add /settings route |
| `apps/web/src/components/layout/TopBar.tsx` | Add search, pin buttons |
| `apps/web/src/components/sidebar/UserBar.tsx` | Add settings gear icon |

## Existing Utilities to Reuse

- `useMessagesStore.setReplyingTo(channelId, message)` — reply state management
- `useMessagesStore.addReaction/removeReaction` — reaction state updates
- `useUiStore.togglePinnedPanel()` + `pinnedPanelOpen` — already in store
- `usePinnedMessages(channelId)` hook — TanStack Query wrapper already exists
- `PinnedMessagesPanel` component — fully built, just needs CSS + wiring
- `ReplyPreview`, `FileUploadButton`, `AttachmentDisplay`, `MessageActionBar`, `ReactionBar` — all built, just need CSS
- `EmojiPicker` — built, needs search fix + positioning
- `ContextMenu` component — pattern to follow for positioned popups
- `ProfilePopover` — pattern to follow for positioned overlays
- `Modal` component — reuse for thread creation prompt

## Verification

After all tasks, verify end-to-end:
1. Hover a message → action bar appears with styled buttons
2. Click Reply → ReplyPreview appears above composer, sending includes reference
3. Click React emoji → EmojiPicker appears positioned, selecting adds reaction visible in real-time
4. Another user reacts → reaction appears via gateway in real-time
5. Click pin button in TopBar → PinnedMessagesPanel slides in
6. Click search icon → SearchPanel opens, type query → results appear
7. Right-click message → "Create Thread" → thread panel opens with messages
8. Click gear in UserBar → Settings page with account/appearance/notifications
9. Press Escape on settings → returns to previous view
