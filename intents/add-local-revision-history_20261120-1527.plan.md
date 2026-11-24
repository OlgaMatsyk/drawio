# Plan: Keep local revision history for device and browser-storage diagrams

## Approach
Snapshots of replaced versions for device files (LocalFile) and browser-storage files (StorageFile) are kept in a new IndexedDB store, keyed by a per-file identity, and never inside the diagram file itself. The save path of both classes writes the prior version to that store before the write, and both classes override the two existing hooks (`isRevisionHistorySupported`, `getRevisions`) so File > Revision history and the existing RevisionDialog work unchanged for them. Restore loads the chosen snapshot as a new, unsaved document, so the stored file is never written by restore. Removing history deletes the store entries for one file, and because snapshots never leave IndexedDB, a shared or exported copy contains no history by construction. Google Drive (DriveFile.js) and the desktop app (guarded by the existing `EditorUi.isElectronApp` check) keep their current behaviour. Enabling the hooks for these two classes also activates `DrawioFile.prototype.getRecoveryVersion` for them, which reads the same revision list; this is accepted here and the S2 check covers it. Retention is capped at 20 snapshots per file (Q2 default). The store is the first step, because both the save hook and the removal action depend on its API.

## Subtasks

### S1: Snapshot store module
- **Does:** Add an IndexedDB-backed store with put (append snapshot, prune oldest beyond 20), list (ordered oldest to newest, with timestamp and XML), and removeAll for one file key; register the new script in the two script lists so it loads in the app and the service worker.
- **Covers:** A1 (storage part), A4 (removal primitive).
- **Paths:** src/main/webapp/js/diagramly/LocalRevisionStore.js (new); src/main/webapp/service-worker.js (script list only); src/main/webapp/js/diagramly/Devel.js (script list only).
- **Done when:** Loading the app in a browser shows no script errors; in the devtools console, putting 22 snapshots for one key leaves 20 from list(), and removeAll(key) leaves list(key) empty while another key's entries are untouched.

### S2: Save snapshots and restore for LocalFile and StorageFile
- **Does:** In LocalFile.saveFile and StorageFile.saveFile, store the version being replaced before the write, using the file's key from S1; override `isRevisionHistorySupported` (true only when not in the desktop app) and `getRevisions` (returns items shaped like DriveFile's: title, modifiedDate, getXml, plus a restoreFn that opens the snapshot as a new unsaved document rather than writing it back).
- **Covers:** A1, A2, A3, A5 (only these two classes are touched).
- **Paths:** src/main/webapp/js/diagramly/LocalFile.js; src/main/webapp/js/diagramly/StorageFile.js.
- **Done when:** In the browser, saving a device file three times then opening File > Revision history lists two prior snapshots in RevisionDialog; restoring one opens it as modified and unsaved, and the file on disk or in storage is unchanged until the user saves; the same check passes for a browser-storage file; a Google Drive file still lists Drive revisions as before; the status bar shows the revision link for these files and not on the desktop app.

### S3: Remove revision history action
- **Does:** Add a "Remove revision history" item to the File menu (Q3 option A, the menu part) that asks for confirmation, calls removeAll for the current file via S1, and shows a status message; add its English strings to dia.txt.
- **Covers:** A4.
- **Paths:** src/main/webapp/js/diagramly/Menus.js; src/main/webapp/resources/dia.txt.
- **Done when:** For a device or browser-storage file with snapshots, the menu item appears in File, confirming removes all snapshots (Revision history then shows none), and the diagram content is untouched; the item does not appear for Drive files.

## Order
S1 before S2 and S3 (both call the S1 API). S2 and S3 may run in any order.

## Not in this plan
- Desktop (Electron) app and cloud integrations other than Google Drive (task Out of scope).
- Any change to the RevisionDialog layout (task Out of scope); the strip action lives in the File menu, not the dialog.
- Translations other than English in dia.txt.
- A share/export checkbox (Q3 option B). Not needed: snapshots never enter the exported file, so a checkbox would do nothing. Left for a later step if a person wants a visible option.
- Making recovery-from-revision behaviour a separate setting. It follows from the hooks being enabled; a person may decide later whether to turn it off for local files.

## Assumptions
- Q1: Option A, browser IndexedDB keyed by the file's identity (the probable default). The code already keeps browser-storage data in a database with a localStorage fallback (StorageFile.js:232-267, :400), so IndexedDB fits. Device files use the same store, keyed by a per-file identity taken from the file handle or name chosen in S1.
- Q2: Option A, keep the last 20 snapshots per file (the tentative default).
- Q3: Option C narrowed to option A, since option B has no effect (see Not in this plan). The tentative default is therefore taken with one item removed.
- The desktop check is `EditorUi.isElectronApp` (Editor.js uses it the same way). S2 confirms it is the right guard for LocalFile.
- The snapshot of the replaced version is the content the app last loaded or saved for that file; S2 confirms that this is available at save time without an extra read of the file.
