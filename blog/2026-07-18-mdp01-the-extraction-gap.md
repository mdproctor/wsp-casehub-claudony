---
layout: post
title: "The Extraction Gap — When Shadow DOM Piercing Isn't Enough"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [claudony]
tags: [playwright, shadow-dom, lit, e2e]
---

# Claudony — The Extraction Gap

**Date:** 2026-07-18
**Type:** phase-update

---

## What I was trying to achieve: make the E2E tests match the new DOM

Issue #170 moved claudony's channel-panel from a vanilla HTMLElement to a LitElement backed by `@casehubio/blocks-ui-channel-activity` components. Every E2E test that queried channel-panel DOM was broken. The issue description said "only selector updates" — and that turned out to be wrong in an interesting way.

## What we believed going in: Playwright handles Shadow DOM transparently

The existing garden entry GE-20260614 established a clear rule: Playwright's locator engine auto-pierces open shadow DOM. We expected the fix to be mechanical — rename `.ch-case-header` to `.case-header` (the `ch-` prefix was dropped when shadow DOM made namespacing redundant), update `#ch-select` to `.ch-select`, done.

The selector renames were straightforward. CaseContextPanelE2ETest needed fifteen individual updates — class renames, a structural change where Lit's conditional rendering replaced CSS class toggling for lineage visibility, and a `getElementById` call that doesn't pierce shadow DOM. All mechanical, all predictable.

## The trap: locator selection pierces, but locator methods don't

The interesting part was `feedText()`. This helper called `textContent()` on the `<channel-feed>` element to get all message text for assertions. After the refactor, it returned empty string. Every time. No error.

The DOM had three levels of shadow roots: `claudony-channel-panel` → `channel-feed` → `channel-message`. Playwright found `.message-item` elements inside `channel-feed`'s shadow root without trouble — its locator engine pierced both boundaries. But `textContent()` calls the standard DOM API, which stops at the first shadow boundary. The text content lives inside `channel-message`'s shadow root. `channel-feed.textContent` sees `channel-message` as an empty element.

The fix was `evaluateAll()` on the leaf shadow hosts:

```java
Object result = panelShadow().locator("channel-message").evaluateAll(
    "els => els.map(el => el.shadowRoot ? el.shadowRoot.textContent "
    + ": el.textContent).join(' ')");
```

Playwright's piercing locator finds the `channel-message` elements across shadow boundaries. Then `evaluate()` reads each element's own `shadowRoot.textContent` — which contains that component's rendered badges, content, and metadata. No recursive traversal needed because the leaf components don't nest further shadow hosts.

## One production bug, hidden by the old tests

The channel-panel's `connectedCallback()` never set the initial `collapsed` class on the host element, even though `_collapsed` defaulted to `true`. The `:host(.collapsed)` CSS rule — which collapses the panel to zero width — never fired on first render. The old vanilla HTMLElement tests didn't catch this because they set the class differently. One line fixed it: `this.classList.add('collapsed')` in `connectedCallback()`.

## Where it is now

Twenty-one of twenty-five E2E tests pass. The four remaining failures are unrelated to shadow DOM — Qhorus now rejects EVENT messages with content (a validation change), `messageStore.put()` doesn't make messages visible to the timeline REST endpoint, and `pages-component-terminal` doesn't emit resize events in headless Chromium. Those are filed separately as #174.

The asymmetry between Playwright's selection engine and its extraction methods is the kind of thing that burns hours because the first half works perfectly. You find the right elements, your wait conditions pass, and then every assertion fails with empty string. No error, no warning — just silence at the shadow boundary.
