# Quick Open File Design

Date: 2026-05-06

## Summary

Upgrade Warp's existing files palette into an IDE-style Quick Open File flow, triggered by `Cmd+Shift+O`.

The interaction is:

- Press `Cmd+Shift+O`
- A modal opens
- The user types a filename or path fragment
- Results update in real time
- Up/Down changes the selected result
- The lower half of the modal shows a scrollable, read-only preview of the selected file
- Pressing `Enter` opens the selected file as a normal editable file tab
- If the file is already open in the current workspace, Warp switches to that existing tab instead of opening a duplicate

This feature is intentionally scoped as the file-specific `Cmd+Shift+O` experience. It should reuse Warp's existing files-palette entry point and file-opening infrastructure rather than introducing a second top-level shortcut pathway.

## Goals

- Match the core ergonomics of IDE "Open File" interactions found in tools like Android Studio and Xcode
- Reuse Warp's existing file search and file-opening infrastructure where possible
- Keep preview behavior separate from real tab lifecycle
- Open files as normal editable tabs after confirmation
- Reuse already-open tabs instead of creating duplicates

## Non-Goals

- Multi-repo or workspace-wide cross-repository search beyond the active local repository
- Search history, recent query history, or advanced open modes
- Inline editing inside the preview area
- Path/line-number jump syntax
- Side-by-side preview layouts
- Remote-session parity beyond existing local filesystem behavior

## User Experience

### Entry Point

- Reuse the existing files-palette entry point bound to `Cmd+Shift+O`
- Triggering the action opens the file-specific quick open surface

### Layout

The modal is vertically split into two sections:

- Top section:
  - Search input
  - Search results list
- Bottom section:
  - Read-only file preview for the currently selected result
  - Vertical scrolling is supported

This is a dedicated file modal, not a mixed command/result surface.

### Keyboard Behavior

- Typing updates search results in real time
- `Up` / `Down` moves the selected result
- Selection changes update the preview
- `Enter` confirms the selection and opens the file
- `Esc` closes the modal without changing the open tabs
- Preview scroll should support:
  - Mouse wheel
  - Trackpad scrolling
  - Keyboard paging if the embedded preview component already supports it cleanly

### Open Behavior

When the user presses `Enter`:

- If the file is already open somewhere in the current workspace, switch focus to that existing tab
- Otherwise, open the file as a new normal editable file tab
- Close the quick open modal after the open action is accepted

### Preview Behavior

- The preview is read-only
- The preview is not represented as a real tab
- The preview updates only for the currently selected item
- Preview loads should be selection-driven, not eagerly generated for all results

## Architecture

### Recommended Shape

Implement this as a dedicated quick open feature composed from existing primitives, not as a deep expansion of the generic command palette.

Recommended pieces:

1. Reuse the existing workspace files-palette binding and entry point for `Cmd+Shift+O`
2. Extend the file-mode palette UI with an embedded preview surface
3. Reuse the existing `FileSearchModel`
4. Reuse existing workspace/code file-opening flows for final tab opening
5. Use a modal-local read-only preview surface that does not participate in real tab lifecycle

### Why Reuse the Existing Files Palette Entry Point

Warp already has a file-specific `Cmd+Shift+O` entry point and file search data source. Reusing that path keeps the keyboard shortcut, telemetry path, and palette lifecycle aligned with the rest of the workspace modal system.

The implementation should still treat this as a file-specific interaction, not as a generic command search behavior. The preview UI and forced-new-tab open behavior are specific to file mode.

## Detailed Design

### 1. Workspace Action and Modal Launch

Reuse the existing workspace files-palette binding and action path for `Cmd+Shift+O`.

The workspace still owns opening and closing the modal, consistent with other top-level palette entry points.

The modal should be focus-first:

- Open centered
- Focus the search input immediately
- Dismiss cleanly on `Esc`

### 2. Search Data Source

Use the existing `FileSearchModel` as the source of truth for file discovery and fuzzy matching.

This model already provides:

- Current repo resolution
- Repository content enumeration
- Fuzzy path matching
- Cache invalidation based on repo metadata updates

The quick open feature should reuse that logic rather than introducing a new indexing or scanning path.

### 3. Result Ranking

Use current file-search ranking behavior as the baseline, including any prioritization already informed by:

- Opened files
- Git-changed files
- Fuzzy match quality

No new ranking system is required for v1.

### 4. QuickOpenFileView Responsibilities

The file-mode palette UI should own only the UI state needed for this interaction:

- Current query string
- Current result list
- Selected result index
- Preview loading state
- Preview content or preview error/placeholder state

It should not own file-tab lifecycle or broader workspace state.

### 5. Preview Area

The preview area should render the selected file in read-only mode.

Requirements:

- Scrollable
- Not editable
- Updates on selection change
- Handles empty, unreadable, binary, and oversized-file cases with explicit placeholders

The preview should prefer reusing existing code/file rendering capabilities in a read-only configuration where practical, rather than inventing a second text-rendering stack.

However, preview must remain modal-local. It should not create a real pane or real tab just to show the content.

### 6. Final Open Path

Pressing `Enter` should emit a final open-file event into the existing workspace/code opening path.

The existing open path should remain responsible for:

- Determining whether the file is already open
- Switching to an existing tab if present
- Creating a new editable file tab if not

The quick open feature should not duplicate tab-resolution logic if the workspace/code layer already provides it.

## State Flow

1. User presses `Cmd+Shift+O`
2. Workspace opens `QuickOpenFileView`
3. Search input is focused
4. User types a query
5. `QuickOpenFileView` requests filtered results from `FileSearchModel`
6. The result list updates
7. The selected item changes through keyboard or pointer input
8. The preview area loads the selected file content
9. User presses `Enter`
10. Workspace/code open logic either:
   - switches to an already-open tab, or
   - opens a new editable file tab
11. Quick open modal closes

## Error and Empty States

The modal should show explicit states rather than silently failing:

- No local repository available:
  - show an empty-state message explaining that quick open requires a local repo context
- No results:
  - show a standard no-results state
- File unreadable:
  - show a preview error state
- Binary file:
  - show a binary preview placeholder
- Oversized file:
  - show a large-file preview placeholder or truncated preview, depending on what existing rendering primitives support safely

For v1, conservative behavior is preferred over aggressively trying to preview every file type.

## Scope Boundaries

### In Scope

- Dedicated quick open modal
- `Cmd+Shift+O`
- Real-time file search in the active local repository
- Up/Down result navigation
- Scrollable read-only preview in the lower half of the modal
- `Enter` opens selected file
- Open existing tab if already open
- Open new editable tab if not already open

### Out of Scope

- Repo picker inside quick open
- Preview as a real temporary tab
- Query history
- Recent searches
- Multiple open modes
- Split-pane opening from quick open
- Path:line navigation syntax

## Testing Strategy

### Unit / Model-Level Coverage

- Query updates produce filtered result lists
- Selection movement updates the selected item
- Empty results are handled correctly

### View / Interaction Coverage

- Modal focuses input on open
- Up/Down updates selection
- Selection changes trigger preview refresh
- Preview is read-only
- Preview scroll works
- `Esc` dismisses without side effects

### Open Behavior Coverage

- `Enter` on an unopened file opens a new editable file tab
- `Enter` on an already-open file switches to the existing tab
- Modal closes after successful open

### Regression Coverage

- Existing command palette behavior remains unchanged
- Existing file-tree open behavior remains unchanged
- Existing file search ranking remains stable unless intentionally updated

## Implementation Notes

- Prefer reusing the existing files-palette entry point over introducing a second top-level shortcut path
- Reuse existing search and file-opening infrastructure wherever possible
- Keep preview lifecycle local to the modal
- Avoid introducing speculative abstractions for other future palettes

## Open Questions Resolved

- Preview location: lower half of the modal
- Preview scrolling: supported
- Final open behavior: if already open, switch to that tab; otherwise create a new editable file tab
- Final opened tab type: normal editable file tab, not a special preview tab
