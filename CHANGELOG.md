# Changelog

All notable changes to LazySnap are listed here. Version numbers follow the `version` field in `manifest.json`.

## Rename — 2026-09-17

The extension was renamed from **TickerSnap** to **LazySnap**. The version number did not change. The rename updated the manifest name, the docs, the store listing and the repository links, and added the *Made by Raisul Sohan* credit link to the popup.

## [1.2] — 2026-09-04

Chrome Web Store release preparation.

### Added

- Preview panel under the buttons showing the captured text with word and character counts, with a **Hide / Show** link whose state is remembered.
- The last capture (text, file name, counts) is kept in session storage, so closing the popup no longer loses it; it is restored on reopen and cleared when the browser restarts.
- Checkbox states and the custom selector are saved to `chrome.storage.sync`; the Advanced panel opens by itself when a selector is saved.
- Article mode, Google Docs: reads the embedded document data, the plain-text export, published documents, rendered text nodes, or the selection, in that order.
- Article mode, paginated viewers: captures the most visible page of PDF.js-based viewers, Google Drive's PDF preview and similar web readers, with a *Page N* label.
- Article mode: a selection longer than 30 characters is captured on its own.
- Article mode: open popups and dialogs are read in preference to the page behind them.
- Article mode: content-container scoring with Readability preferred on ordinary pages, removal of site furniture (captions, share buttons, related stories, comments, breadcrumbs, badges, ads), noise-line filtering in English and Bengali, and pull-quote de-duplication.
- *Include Title & Source header* option for article text (off by default).
- Icons in four sizes, privacy policy and Chrome Web Store listing text.

### Changed

- A failed extraction keeps the previous capture instead of wiping it.
- File names are sanitised and capped at 80 characters.

## [1.1] — 2026-07-10

### Added

- **Extract article text**: clean body text of any news article page, powered by the bundled Mozilla Readability.js (Apache-2.0).

## [1.0] — 2026-06-14

First release, as **TickerSnap**.

### Added

- **Extract commentary**: one-click capture of the full match commentary from FotMob and Sofascore match pages, using prose-cluster detection instead of site-specific selectors, with minute stamps (`45'`, `90+3'`), event labels and automatic scrolling of lazy-loaded feeds.
- **Copy** and **Download .txt**.
- *Newest first* checkbox to flip the output into chronological order.
- Advanced custom container selector.
