# Keep local revision history for device and browser-storage diagrams

Type: story
Created: 2026-11-20 15:27 UTC
Source: operator, task 65b92342-3ad6-4004-b957-a96e0e1fd6ee
Base branch: dev

## Intent
Add local revision history for diagrams saved to the device or to browser storage: keep a snapshot of the previous version on every save, show them in File > Revision history with the existing dialog, never overwrite the file on restore, and let users strip the history before sharing.

## Context
- The File > Revision history action exists today: `revisionHistory...` in src/main/webapp/js/diagramly/Menus.js:861, which checks `editorUi.isRevisionHistorySupported()` and calls `editorUi.getRevisions(...)`.
- The hooks default to unsupported: `DrawioFile.prototype.isRevisionHistorySupported` and `DrawioFile.prototype.getRevisions` in src/main/webapp/js/diagramly/DrawioFile.js:2719 and :2727 (return false / null).
- Only `DriveFile` overrides them today (src/main/webapp/js/diagramly/DriveFile.js:601, :609).
- The dialog is `RevisionDialog` in src/main/webapp/js/diagramly/Dialogs.js:8501.
- Device files use `LocalFile` (src/main/webapp/js/diagramly/LocalFile.js); browser storage uses `StorageFile` (src/main/webapp/js/diagramly/StorageFile.js). Neither overrides the revision hooks today.
- Strings come from src/main/webapp/resources/dia.txt (`revisionHistory=Revision History` at line 928).

## Acceptance
- A1: Each save of a device-file or browser-storage diagram stores a snapshot of the version being replaced.
- A2: File > Revision history lists those snapshots in the existing RevisionDialog for device-file and browser-storage diagrams.
- A3: Restoring a snapshot never overwrites the stored file; the restored content opens as a new, unsaved state the user must save explicitly.
- A4: The user can remove all local revision history for a diagram, and a shared or exported copy can be produced without it.
- A5: Google Drive and the desktop app behave as before.

## Out of scope
- Desktop (Electron) app and cloud storage integrations other than Google Drive, which already has its own history.
- Changing the existing RevisionDialog layout beyond what the restore and strip actions need.
- Translations other than English strings in dia.txt.

## Open questions
- Q1: Where are snapshots kept for device files, which cannot hold hidden data beside the file? Options: A) browser IndexedDB keyed by the file's identity, B) a sidecar file the user saves next to the diagram, C) not kept across sessions. Default: A (probable).
- Q2: How many snapshots are kept per diagram? Options: A) last 20, B) last 50, C) unlimited. Default: A (tentative).
- Q3: Where does "strip history before sharing" live? Options: A) a "Remove revision history" item in the File menu, B) a checkbox in the share/export dialog, C) both. Default: C (tentative).

## Sources
- src/main/webapp/js/diagramly/Menus.js:861 (revisionHistory action): read.
- src/main/webapp/js/diagramly/DrawioFile.js:2719-2750 (default revision hooks and history flow): read.
- src/main/webapp/js/diagramly/Dialogs.js:8501 (RevisionDialog): read.
- src/main/webapp/js/diagramly/DriveFile.js:601-615 (Drive override, reference only): read.
- src/main/webapp/resources/dia.txt:928 (revisionHistory string): read.
