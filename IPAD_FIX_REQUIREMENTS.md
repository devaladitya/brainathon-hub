# Memory Coach (brainathonhub.com) — iPad Compatibility Fix Requirements

---

## Instructions for the engineer / AI assistant

You are working on the **Memory Coach website codebase** hosted at `brainathonhub.com`. This document describes a bug that must be fixed so the companion iOS app can pass App Store review.

**Follow this order strictly — do not skip steps:**

1. **Read this entire document first.** Do not start editing code before you understand the full context.
2. **Reproduce the bug before fixing anything.** Load the site on an iPad (or iPad Simulator on a Mac). Confirm you see the black screen. If you cannot reproduce, stop and report back — the bug may have already been partially fixed.
3. **Investigate before editing.** Open Safari Web Inspector on the iPad, check the Console for JavaScript errors, check the Network tab for failed requests, inspect the DOM to see if content is rendered-but-hidden vs never-rendered. Report findings before making changes.
4. **Make minimal, targeted fixes.** Do not refactor. Do not "clean up" unrelated code. Every change must directly address the iPad rendering failure.
5. **Verify each fix on iPad before moving on.** Do not batch multiple unverified changes.
6. **Do not modify the iOS Capacitor project** (that lives in a separate repo). Your work is 100% website-side.
7. **When done, fill out the "Definition of done" checklist at the bottom of this doc** and report which fixes were applied.

**If you (the AI assistant) do not have access to an iPad or iPad Simulator,** tell the user immediately. You cannot complete this task without reproducing the bug — do not guess fixes.

---

## Context

The iOS app (wrapped via Capacitor, loads `https://brainathonhub.com/` inside a WKWebView) was **rejected by Apple App Review** under:

- **Guideline 2.1.0 — App Completeness** (app does not work on iPad)
- **Guideline 2.3.10 — Accurate Metadata** (app name mismatch — being handled separately by renaming the App Store listing to "Memory Coach")

This document covers the **website-side work** needed to fix the iPad rendering issue so the app can be resubmitted.

---

## The Problem (what Apple's reviewer saw)

Reviewer device: **iPad** (iPadOS, landscape, dated Tue Mar 24, 10:18 AM)
Reviewer screenshot shows: **A completely black screen with only the Memory Coach brain logo rendered in the center.** No header, no content, no "Sign In" / "Get Started" buttons — nothing.

Meanwhile, the same site renders correctly on:
- Android phones (confirmed by Apple's additional screenshots)
- (Presumably desktop and iPhone)

**Conclusion: The site fails to render properly on iPad / iPadOS Safari / WKWebView.** This is an iPad-specific bug.

---

## Most likely root causes (investigate in this order)

### 1. Viewport / media query issue (highest probability)

The site may have CSS that works on phone widths and desktop widths but **breaks at iPad width** (typically 768px – 1366px). Possibly:

- A media query hides main content at tablet breakpoints
- Layout uses `display: none` on container elements at certain widths
- Background color defaults to black at tablet size while text/UI color is also dark (invisible text)

**Check:** Open the site in desktop browser, resize to 820×1180 (iPad Air) and 1024×1366 (iPad Pro). Does the content disappear?

### 2. JavaScript execution failure in WKWebView

WKWebView (iOS) has stricter behavior than Chrome/desktop Safari:

- **Service workers** are **not supported** inside WKWebView — if the site relies on a service worker to render content, it will never render on iOS
- **Third-party cookies** blocked by default
- **CORS** stricter — any failed fetch to a different origin may silently break the app
- **localStorage** may be cleared unexpectedly in a WKWebView

**Check:** Connect the iPad (or iPad Simulator on a Mac) to Safari's **Web Inspector** (Mac Safari → Develop → [iPad] → brainathonhub.com). Look at the Console for JS errors.

### 3. Framework bootstrap / hydration failure on iPad

If the site is built with React / Vue / Angular / Next.js:
- A hydration error on iPad can leave the page blank
- Unsupported JS syntax (e.g., very new ES2024 features without transpiling) breaks only older Safari versions
- `Intl.*` / `BigInt` / `?.` etc. may fail on older iPadOS

**Check:** Does the site support iOS Safari 14+? What's your minimum supported Safari version in the build config (e.g., `browserslist`)?

### 4. 100vh / safe-area bug

iOS Safari treats `100vh` differently (includes or excludes the URL bar). If the root container uses `height: 100vh` and content is absolutely positioned, parts can render off-screen.

### 5. Dark background + invisible content

The black screen in the screenshot is suspicious. Possibly:
- `body` has `background: #000` applied via a media query
- Main content container has `opacity: 0` waiting for JS animation that never fires
- CSS variable fallback fails on iPad, leaving text color same as background

---

## Required fixes

### A. Add/verify iPad viewport meta tag

In the site's root HTML `<head>`:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
```

### B. Verify responsive CSS at iPad breakpoints

Test the site at these exact widths:
- **820 × 1180** (iPad Air, portrait)
- **1180 × 820** (iPad Air, landscape)
- **1024 × 1366** (iPad Pro 12.9", portrait)
- **1366 × 1024** (iPad Pro 12.9", landscape)

Every page (home, sign-in, register, chat, memory coach pages) must render fully at all four sizes.

### C. Check all media queries

Search the CSS for:
```
@media (min-width: 768px)
@media (max-width: 1024px)
@media (min-width: 1024px)
```
…and confirm none of them hide the main content, set `display: none` on layout containers, or apply a dark background without a corresponding dark text color.

### D. Remove service-worker dependency for initial render

If the site uses a service worker:
- The first render **must not depend on SW caching** (WKWebView doesn't run SWs)
- All critical JS/CSS must be loadable via normal HTTP

### E. Fix any `100vh` usages

Replace:
```css
height: 100vh;
```
With:
```css
height: 100vh;
height: 100dvh; /* dynamic viewport height — respects iOS URL bar */
min-height: -webkit-fill-available;
```

### F. Add a visible error fallback

If any async load fails (API, auth, analytics), the UI **must** show an error message — not a blank screen. Apple failing to see content = instant rejection.

### G. Test in actual iPad Simulator before delivery

**Do not ship without this test.** On a Mac:
1. Open Xcode → Xcode menu → Open Developer Tool → Simulator
2. Device → iPad (Air or Pro)
3. Open Safari in the simulator → navigate to `https://brainathonhub.com/`
4. Confirm the full site renders — header, buttons, sign-in, everything
5. Also test the full app via the iOS Capacitor build (with WKWebView wrapping the site)

---

## Out of scope (we are handling separately)

- App Store listing rename from "Brainathon Hub" → "Memory Coach"
- `capacitor.config.json` `appName` update
- iOS `Info.plist` display name update
- Uploading new App Store screenshots
- Rebuilding and resubmitting the iOS app via Codemagic

---

## Definition of done

- [ ] Website renders fully on iPad Simulator (iPad Air + iPad Pro, portrait + landscape)
- [ ] Website renders fully in physical iPad Safari (if available)
- [ ] Website renders fully inside the Capacitor iOS app running on iPad Simulator
- [ ] No JavaScript errors in Safari Web Inspector console when loaded in iPad
- [ ] No black/blank screen at any viewport width between 320px and 1366px
- [ ] Error fallback displays a visible message if any critical resource fails to load

Once all boxes are ticked, ping the iOS team to rebuild and resubmit to App Review.

---

## Reference — Apple's rejection evidence

Apple attached 5 files to the rejection (iPad screenshot is the critical one):

| File | Device | Status |
|---|---|---|
| `Screenshot-0324-101803.png` | iPad (Tue Mar 24) | **Black screen with only logo — THIS IS THE BUG** |
| `b4242dfb…jpg` | Android phone | Home page — renders fine |
| `9385384…jpg` | Android phone | Sign-in page — renders fine |
| `48810318…jpg` | Android phone | Chat page (L1 Finishing Pages) — renders fine |
| `738be45…jpg` | Android phone | Chat page (L1 Finishing Pages) — renders fine |

The bug is **iPad-specific**. Phones are fine.
