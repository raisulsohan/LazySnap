# LazySnap User Guide

LazySnap is a one-click text extractor with two modes: the full **match commentary** from a FotMob or Sofascore match page, and the clean **article text** of a news story, blog post, web document or PDF page. Everything runs inside your browser. This guide covers every feature of version 1.3.

## Contents

1. [Install](#install)
2. [The popup at a glance](#the-popup-at-a-glance)
3. [Extract match commentary](#extract-match-commentary)
4. [Extract article text](#extract-article-text)
5. [Preview, Copy and Download](#preview-copy-and-download)
6. [What is remembered](#what-is-remembered)
7. [Advanced: custom container selector](#advanced-custom-container-selector)
8. [Troubleshooting](#troubleshooting)
9. [Usage rights](#usage-rights)

## Install

LazySnap is loaded as an unpacked extension. There is no build step.

1. Download `LazySnap-v1.3.zip` from the [latest release](https://github.com/raisulsohan/LazySnap/releases/latest) and unzip it somewhere permanent, or clone the repository. The browser loads the files from that folder, so do not move or delete it afterwards.
2. Open `chrome://extensions` (Edge: `edge://extensions`, Brave: `brave://extensions`).
3. Turn on **Developer mode** (top right).
4. Click **Load unpacked** and pick the `LazySnap` folder, the one that contains `manifest.json`.
5. Optional: pin the LazySnap icon to the toolbar from the extensions menu.

LazySnap uses the `activeTab` model, so it cannot read any page until you click its icon on that tab, and it shows no "read all websites" warning. It works on `http://`, `https://` and `file://` pages. For `file://` pages, open the extension's **Details** page and turn on **Allow access to file URLs**.

## The popup at a glance

| Control | What it does |
|---|---|
| **Extract commentary** (blue) | Reads the commentary feed of the match page in the current tab |
| **Extract article text** (green) | Reads the body text of the page in the current tab |
| **Copy** | Copies the last capture to the clipboard |
| **Download .txt** | Saves the last capture as a plain-text file |
| Status line | Progress, result summary or error |
| Preview panel | The captured text, with word and character counts, and a **Hide / Show** link |
| **Article only: include Title & Source header** | Adds a title line, an underline and a `site — byline` line above article text |
| **Commentary only: page lists newest entries first — flip to chronological** | Reverses the commentary so the output starts at kick-off |
| **Advanced: custom container selector** | A CSS selector that limits extraction to one part of the page |

Both extract buttons are disabled while an extraction is running.

## Extract match commentary

1. Open the match page on **FotMob** or **Sofascore** and switch to its **Commentary** tab.
2. Click the LazySnap icon, then **Extract commentary**. The status line reads *Extracting… (auto-scrolling the commentary)*. The page scrolls itself from top to bottom to make the site load every entry. Leave the tab in the foreground and do not scroll it yourself until the status changes.
3. When it finishes, the status shows how many entries were captured and the preview appears.
4. **Copy** or **Download .txt**. The file is named `<match title> - commentary.txt`.

**Order.** The checkbox *page lists newest entries first* is ticked by default because both sites show the latest event at the top. With it ticked, LazySnap reverses the list so the output runs from kick-off to full time. Untick it if the page already lists the oldest entry first.

**Output format.** One entry per line, after a title block:

```
Arsenal 2-1 Chelsea
===================

1'  Kick off. Arsenal get the game under way.
23'  [Goal] Saka finishes from close range after a low cross from the left.
•  Half-time. Arsenal lead through Saka's early strike.
90+3'  Full-time whistle. Arsenal hold on.
```

- The line starts with the minute stamp (`45'`, `90+3'`). Entries without a minute get `•`, or the event name if the entry is a recognised event.
- If an entry has both a minute and a recognised event type (Goal, Own goal, Yellow card, Red card, Substitution, VAR, Penalty and so on) the type appears in square brackets. *Summary* is not tagged.

**How detection works.** LazySnap uses no site-specific selectors. It finds every block of real prose on the page, groups the blocks into cards, takes the densest cluster of cards as the commentary list, and keeps every card in that cluster plus any card elsewhere that carries a minute stamp or an event type. Duplicate entries picked up on successive scrolls are dropped.

Because FotMob and Sofascore structure their pages differently, one of them may leak a little noise (preview text, stat sentences) into the output, or auto-detect may miss. Use the [custom container selector](#advanced-custom-container-selector) in that case.

## Extract article text

Click **Extract article text** on any page. LazySnap picks what to read in this order:

1. **Google Docs**: the whole document (see [Google Docs](#google-docs)).
2. **Your selection**: if you have selected more than 30 characters on the page, only that selection is captured. Select text first when you want a single passage, or when auto-detect keeps picking the wrong block.
3. **A page in a paginated viewer**: in PDF.js-based viewers, Google Drive's PDF preview and similar web readers, the page that is most visible in the window is captured. Scroll to the page you want first. The header line shows *Page N*.
4. **The article body**: otherwise LazySnap reads the main content of the page, using Mozilla's Readability engine (the engine behind Firefox Reader View) with its own clean-up on top. Navigation, headers, footers, captions, share buttons, related-story rails, comments, cookie banners and similar furniture are removed, pull-quotes that repeat a paragraph are dropped, and short promotional lines such as *Read more* or *Also read* are skipped, in English and in Bengali.

If the article is shown in an open popup or dialog (a reader overlay, a lightbox), that popup is read instead of the page behind it, and the status shows *from open popup*.

**Header.** Tick *include Title & Source header* to put the title, a line of `=` and a `site — byline` line above the text. It is off by default, so the capture is the body text only. The word count in the status includes the header when it is on.

### Google Docs

Open the document in its normal editing view. LazySnap tries, in order: the document data embedded in the page, the document's plain-text export (fetched from `docs.google.com` with your own login, so it works for documents you can open), the content of a published document, the text nodes of the rendered page, and finally any text you have selected.

If none of those work the status reads *Could not read this Google Doc automatically (Canvas mode active). Please select the text (Ctrl+A) and try again.* Click inside the document, press `Ctrl+A`, then extract again.

### PDFs

Web viewers built on PDF.js, Google Drive's PDF preview and readers that render one page container per page are supported; the most visible page is captured. Chrome's own built-in PDF viewer is a browser-internal page that no extension can script, so LazySnap reports *This tab can't be read* there. Open the file in Google Drive or in a web-based viewer instead.

### Paywalls and logins

Text that a site hides behind a paywall or a login is not on the page, so it is not extracted, and LazySnap does not try to get around that.

## Preview, Copy and Download

- The preview shows the first 20,000 characters of the capture. **Copy** and **Download .txt** always give the full text.
- The line above the preview shows the entry count or the source (*from selection*, *Page 4*, *from open popup*, *from custom selector*), the word count and the character count.
- **Hide** collapses the preview; **Show** brings it back. The choice is remembered.
- **Copy** writes to the clipboard and confirms with *Copied to clipboard.*
- **Download .txt** saves to your Downloads folder. Article files are named after the title (`<title>.txt`); commentary files as `<title> - commentary.txt`. Characters that are not allowed in file names are replaced and the name is cut at 80 characters.

## What is remembered

| What | Where | How long |
|---|---|---|
| The two checkboxes, the custom selector and whether the preview is open | `chrome.storage.sync`, so they follow your Chrome profile if sync is on | Until you change them |
| The last capture: text, file name and counts | `chrome.storage.session`, in memory only | Until the browser is closed |

Reopening the popup restores the last capture with the status *Restored — …* and enables **Copy** and **Download**. A failed extraction keeps the previous capture and says so: *(Previous capture is still below.)*. A capture too large for session storage is not restored after the popup closes; copy or download it before closing.

When a custom selector is saved, the Advanced panel opens by itself so you do not forget it is set.

## Advanced: custom container selector

Type a CSS selector, for example `.entry-content` or `[data-testid='commentary']`, to limit extraction to that element. Leave it empty for auto-detect. To find a selector, right-click a line of the text you want, choose **Inspect** and read the element's id or class.

- **Commentary**: only prose inside the selected element is harvested.
- **Article**: the selected element is read on its own. The paginated-page detection is skipped and Readability is not consulted, unless the element yields no usable text (under 50 characters). The status line shows *from custom selector*. A text selection on the page still takes priority over the selector.

The selector is saved, so a site that needs one only needs it typed once. Clear the field to go back to auto-detect.

## Troubleshooting

**This tab can't be read (browser internal pages are off-limits).** The tab is a `chrome://` page, the Web Store, Chrome's built-in PDF viewer or another extension's page. Open the content as a normal web page.

**No commentary text detected. Open the Commentary tab and retry.** The match page is not on its Commentary tab, the feed has not loaded yet, or a saved custom selector does not match this page. Switch to the Commentary tab, wait for it to load, and clear the selector.

**No commentary entries found. Reload the page and retry.** The page had prose but no commentary cluster. Reload and try again, or set a selector.

**The commentary has stray lines or misses some entries.** Set a custom selector on the commentary list container.

**Couldn't find an article body here.** The text is behind a paywall or login, or the page is mostly images or scripts. Select the text you want and extract again.

**The wrong section was extracted.** Select the passage you want first, or set a custom selector.

**Could not read this Google Doc automatically.** Click in the document, press `Ctrl+A`, then extract again.

**Copy failed.** The clipboard was not available to the popup. Use **Download .txt** instead.

**Extraction is slow.** A long commentary feed can take a minute: the page is scrolled in steps of about 85% of the window height, with a short pause after each step, until no new entries appear.

**Nothing was restored when I reopened the popup.** The browser was restarted since the capture, or the capture was too large to keep.

## Usage rights

FotMob and Sofascore both prohibit scraping in their Terms of Use, and commentary, like news articles, is copyrighted editorial text. Treat all output as **personal research and reference**. Do not republish it verbatim or use it commercially.

Article extraction is powered by [Mozilla Readability](https://github.com/mozilla/readability) (Apache License 2.0); the license header is preserved in `Readability.js`.
