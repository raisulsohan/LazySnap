# LazySnap Developer Guide

How the extension is put together, how each extractor decides what to keep, and how to work on it. Read the [user guide](user-guide.md) first if you have not used the extension yet.

## Contents

1. [Overview](#overview)
2. [File map](#file-map)
3. [How code reaches the page](#how-code-reaches-the-page)
4. [Commentary extractor](#commentary-extractor)
5. [Article extractor](#article-extractor)
6. [The popup](#the-popup)
7. [Storage](#storage)
8. [Permissions](#permissions)
9. [Working on it](#working-on-it)

## Overview

LazySnap is a Manifest V3 extension written in plain JavaScript. There is no build step, no background service worker and no content script declared in the manifest. Everything happens while the popup is open: the popup injects an extractor function into the active tab, waits for its return value, formats it and shows it.

The only third-party code is `Readability.js`, Mozilla's article extractor (Apache-2.0), bundled unmodified.

## File map

```
manifest.json        MV3 manifest: activeTab, scripting, storage; popup action
popup.html           the popup UI
popup.js             the two injected extractors plus all popup logic
Readability.js       Mozilla Readability, injected before the article extractor
icons/               16, 32, 48 and 128 px icons
CHROMEWEBSTORE.md    store listing text, permission justifications, packaging steps
PRIVACY.md           privacy policy linked from the store listing
CHANGELOG.md         version history
docs/                this guide and the user guide
```

## How code reaches the page

Both extractors live in `popup.js` as ordinary top-level functions and are injected with `chrome.scripting.executeScript({ target, func, args })`. Chrome serialises the function's source and runs it in the page's isolated world, so:

- An extractor must be **self-contained**. It cannot reference any variable, constant or helper defined elsewhere in `popup.js`; everything it needs is defined inside it or passed through `args`.
- Its return value must be **JSON-serialisable** (plain objects, arrays, strings, numbers). DOM nodes cannot come back.
- It can be `async`; Chrome waits for the promise.

For article mode the popup first runs `executeScript({ files: ['Readability.js'] })`, then injects `extractArticleFromPage`. Both run in the same isolated world, so the extractor sees the `Readability` global. Injecting the library again on every click is harmless: the file only defines a function.

The popup only injects into tabs whose URL starts with `http://`, `https://` or `file://` (`getActiveHttpTab()`); anything else gets the *browser internal pages are off-limits* error without a call to `executeScript`.

## Commentary extractor

`extractCommentary(opts)` with `opts = { customSelector, reverse }` (`reverse` defaults to `true`).

**Definitions.**

- *Prose* (`isProse`): after removing invisible characters and collapsing whitespace, a string of at least 18 characters and 4 words that contains a space, is not purely numeric and is at least 50% letters.
- *Minute* (`MIN_RE`): `\d{1,3}(\+\d{1,2})?` followed by any apostrophe variant (`'`, `’`, `‘`, `` ` ``, `´`, `′`). Normalised to a plain `'` in the output.
- *Event types* (`EVENT_TYPES`): a fixed list (Goal, Own goal, Penalty goal, Yellow card, Red card, Substitution, VAR, Kick off, Half time, Full time, Big Chance, Summary, Lineups and others), matched case-insensitively against whole text nodes.
- *Block tags* (`BLOCK`): the set of tags that make an element a container rather than a leaf.

**Pipeline.**

1. `collectProse(root)`: every `div`, `p` or `span` under `root` whose text is 18 to 6000 characters, that contains no block-level descendant, and whose `innerText` is prose.
2. `findCard(el)`: climb from a prose element until the parent holds more than one prose child (`cardsInside(parent) > 1`). That element is the entry's *card*.
3. `harvestInto(map, order)`: for every prose element, take its card and the card's parent. The parent that owns the most cards is the commentary list. Keep a card if it belongs to that list, or if it carries a minute or an event type anywhere else on the page. Entries are keyed by their first 160 characters, so cards seen again on a later scroll are ignored. Each entry is `{ minute, type, body }`.
4. Loading: if no prose is found at all, scroll down 500 px up to 5 times (300 ms apart) and probe again. Still nothing means `{ error: 'NO_CONTAINER' }`.
5. Scroll-and-harvest: jump to the top, harvest, then repeatedly scroll by 85% of the window height (280 ms apart, at most 400 steps), harvesting after each step. Stop when the page bottom is reached and three consecutive steps found nothing new, or after eight consecutive empty steps anywhere.
6. Reverse the list if `reverse` is set, take the title from the first `h1` or `document.title`, and return `{ title, count, entries }`.

With `customSelector` set, step 3 searches only inside `document.querySelector(customSelector)` (falling back to the whole document if it does not match).

The popup turns each entry into a line with `lineFor()`: the minute, or the type, or `•`; then `[Type]` when the entry has both a minute and a type other than *Summary*; then the body.

## Article extractor

`extractArticleFromPage(opts)` with `opts = { customSelector }`. Everything is wrapped in one `try`; a thrown error comes back as `{ error: message }`. The success shape is `{ title, byline, site, text }` plus the flags `isSelection`, `isPdfPage` (with `pageNum`), `isModal` and `isCustom`.

The stages run in this order and the first one that produces text returns.

### 1. Google Docs (`location.hostname` contains `docs.google.com`)

The title comes from the `docs-title-input` field or `document.title`. Then:

- **A. Model chunks**: scan `<script>` elements for `DOCS_modelChunk = [...]` and take every item with `ty: "is"` (insert string), concatenating its `s`. If that fails, a regex sweep for `{"ty":"is", "s": "..."}` objects. Soft breaks (`\v`, `\f`) become newlines.
- **B. Plain-text export**: derive the document id from the path and `fetch` `/document/d/<id>/export?format=txt` with `credentials: 'include'`. Same origin as the page, uses the user's own session.
- **C. Published or preview docs**: paragraphs and headings under `#contents` or `.doc-content`.
- **D. Rendered nodes**: `.kix-paragraphrenderer`, `.kix-lineview-text-block`, `svg text`, `.kix-canvas-tile-content`.
- **E. Selection**: `window.getSelection()` if longer than 10 characters.

Otherwise the Canvas-mode error is returned. Results carry `byline: 'Google Docs'`.

### 2. Selection

On any other site, a selection longer than 30 characters wins outright. BBCode tags are stripped, whitespace normalised, and the result is flagged `isSelection`.

### 3. Focused page of a paginated viewer

Skipped when `customSelector` is set. `getFocusedPaginatedPage()` looks for `.pdfViewer .page`, `.page[data-page-number]`, `[data-page-number]`, `.pdf-page`, `.page-container` and `.document-page`, and picks the one with the largest visible area in the viewport (at least 150 px²). Text is read from its `.textLayer` if present: spans are grouped into lines by `getBoundingClientRect().top` (a new line when the top moves by 6 px or more). Fewer than 20 characters falls through to the next stage. The result is flagged `isPdfPage` with `byline: 'Page N'`.

### 4. Article body

**Candidate container.** With `customSelector`, `document.querySelector(customSelector)`. Otherwise every element matching the ordered `CANDIDATE_SELECTORS` list (open dialogs and modals first, then `article`, common content classes, `main`, `section`, `div`) is scored with `scoreContainer()` and the highest wins:

- The element must be visible (`isElementVisible`: larger than 20 × 20, not `display:none`, `visibility:hidden` or `opacity:0`).
- Count its leaf-ish blocks (`p`, `blockquote`, `div`, `span`, `li`, `pre`, `section`, `article` with at most 2 children, at least 30 characters, not inside `UNWANTED_SELECTORS`). No blocks or fewer than 80 prose characters scores −1; more headings than blocks with fewer than 3 blocks (a card grid) scores −1.
- Score = total prose characters + 80 per block. A container that looks like a dialog (`isModalOrOverlay`: `<dialog>`, `role="dialog"`, `aria-modal`, or a visible ancestor with a modal-ish class name) and has at least 200 prose characters gets +100000 so it beats the page behind it.

**Title.** The first heading-like element inside the container (`h1, h2, .entry-title, .modal-title, .title, [class*='title']`), else the page `h1`, else `document.title`.

**Cleaning.** `readContainer(node)` clones the node, runs `purgeUnwanted()` on the clone, then `textFromNode()`:

- `purgeUnwanted` removes `script`, `style`, `noscript`, `template`, `iframe`, `object`, `embed`, `svg`, `canvas` unconditionally, and every `UNWANTED_SELECTORS` match (figures, captions, asides, nav, header, footer, buttons, breadcrumbs, badges, pagination, meta lines, share buttons, ratings, comments, author boxes, related posts, widgets, pull-quotes, callouts, credits, promos, ads) when it holds fewer than 600 characters or is one of the structural tags.
- `textFromNode` walks `p, h1–h6, li, blockquote, pre, div`, keeps only leaf blocks (a `div` counts only if it contains no other block), skips anything inside figure/aside/nav/header/footer/button, strips BBCode, drops lines under 15 characters and lines `isNoiseLine` rejects (separators, close buttons, single-link promo paragraphs, credit/caption/promo/meta prefixes in English and Bengali, short badge-like fragments). Headings that are just a link, or equal to the title, are skipped. Blockquotes and short blocks whose text is contained in a longer paragraph are dropped as pull-quotes; exact consecutive duplicates are dropped. Blocks are joined with blank lines. If nothing survives, the container's raw text is used.

**Readability.** Unless the chosen container is a modal, the whole document is cloned, purged and handed to `new Readability(clone, { keepClasses: false }).parse()`. The parsed HTML is purged and passed through `textFromNode` too. Readability's text replaces the container's when it has at least `SUBSTANTIAL` (600) characters or is simply longer, and its title, byline and site name are adopted. A detected modal skips Readability. A container named by the custom selector skips it too, unless that container yields under 50 characters; `isCustom` is true when the custom container's text was kept.

**Fallbacks.** Under 50 characters, `document.body` is read the same way. Still under 50 characters returns `{ error: 'NO_ARTICLE' }`.

## The popup

State lives in three variables: `extractedText`, `extractedFilename` and `previewMeta`. `setResult()` sets all three, renders the preview, enables Copy/Download, writes the status and saves the session. `setError()` writes the status and, when a capture exists, appends *(Previous capture is still below.)* without touching it.

- **Commentary click**: reads the selector and the *newest first* checkbox, injects `extractCommentary`, builds the text as title, `=` underline, blank line, one `lineFor()` line per entry, and names the file `<sanitised title> - commentary.txt`.
- **Article click**: injects `Readability.js`, then `extractArticleFromPage`. With *include header* on, prepends title, underline and `site — byline`. The source label in the status and the preview meta comes from the result flags: *from selection*, *Page N*, *from open popup* or *from custom selector*. The word count reported in the status is computed from the final text so it always matches the preview meta. The file is `<sanitised title>.txt`.
- **Copy**: `navigator.clipboard.writeText`.
- **Download**: a `Blob` of type `text/plain;charset=utf-8` through a temporary `<a download>`.
- **Preview**: `renderPreview()` shows up to `PREVIEW_CHARS` (20,000) characters and appends a truncation note when the text is longer. **Hide / Show** flips `prefs.previewOpen`.
- `sanitizeFilename()` replaces `\ / : * ? " < > |` with spaces, collapses whitespace and cuts at 80 characters.

Preferences are loaded on open (`loadPrefs`), saved on every change (`savePrefs`; the selector input is debounced 400 ms because sync storage rate-limits writes), and the session capture is restored last (`restoreSession`).

## Storage

| Area | Key | Value |
|---|---|---|
| `chrome.storage.sync` | `includeHeader` | boolean, default `false` |
| `chrome.storage.sync` | `newestFirst` | boolean, default `true` |
| `chrome.storage.sync` | `selector` | string, default `""` |
| `chrome.storage.sync` | `previewOpen` | boolean, default `true` |
| `chrome.storage.session` | `lastExtraction` | `{ text, filename, meta, summary }` |

A session write that fails (quota exceeded on a very large capture) removes the key rather than leaving a partial record.

## Permissions

| Permission | Why |
|---|---|
| `activeTab` | Grants access to the current tab only after the user clicks the icon. No host permissions, no warning at install |
| `scripting` | `executeScript` for the extractor functions and `Readability.js` |
| `storage` | Preferences (sync) and the last capture (session) |

No network requests are made by the extension itself. The one `fetch` is the Google Docs plain-text export, issued from inside the Docs page to its own origin with the user's session.

## Working on it

### Load unpacked and iterate

Load the folder at `chrome://extensions` with Developer mode on. After editing `popup.js` or `popup.html`, click the reload icon on the extension card, then reopen the popup.

### Debugging

- **Popup**: right-click the LazySnap icon and choose **Inspect popup** for the popup's own console and DOM.
- **Injected code**: `console.warn` calls inside an extractor appear in the page's DevTools console. Switch the console context from `top` to **LazySnap** to see the isolated world, where `Readability` is defined after an article extraction.
- **Storage**: from the popup console, `chrome.storage.sync.get(null)` and `chrome.storage.session.get(null)`.

Useful test pages: a live or finished match on FotMob and on Sofascore with the Commentary tab open; a few news sites in English and Bengali; a PDF opened in a PDF.js demo viewer; a Google Doc you own; a page with a reader lightbox.

### Changing an extractor

Keep every helper inside the extractor function. A reference to something outside it works in the popup's own scope but throws `ReferenceError` in the page after serialisation. Return plain data only.

### Updating Readability

Take `Readability.js` from a release of [mozilla/readability](https://github.com/mozilla/readability), keep the Apache-2.0 header at the top, and check that it still defines a global `Readability` constructor with a `parse()` method returning `content`, `title`, `byline` and `siteName`.

### Releasing

1. Bump `version` in `manifest.json` and add a row to the version history in `CHROMEWEBSTORE.md`.
2. Add the changes to `CHANGELOG.md`.
3. Zip `manifest.json`, `popup.html`, `popup.js`, `Readability.js` and `icons/`, as described in `CHROMEWEBSTORE.md`.
