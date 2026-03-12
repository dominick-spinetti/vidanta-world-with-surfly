# CXone Chat + Surfly Cobrowse Integration
### Implementation Notes for Web Development Team

---

## Overview

This document outlines the three key changes required to support simultaneous use of the **CXone Chat widget** and **Surfly Cobrowse** on VidantaWorld.com. Previously, the two SDKs could not be initialized at the same time. The changes below resolve that conflict and add supporting behavior for the bot-to-agent handoff experience.

---

## Change 1 — CXone Initialization Must Be Nested Inside Surfly Initialization

### Problem
When both Surfly and CXone were initialized independently at the page level, the two SDKs conflicted with each other and could not run simultaneously.

### Solution
The entire CXone loader and all `cxone()` configuration calls must be placed **inside** the `Surfly.init()` callback, and only executed when `!Surfly.isInsideSession` is true. This ensures CXone only loads in the leader (customer) context, not inside the Surfly session iframe.

### Implementation

```html
<script>
(function (s, u, r, f, l, y) {
    s[f] = s[f] || { init: function () { s[f].q = arguments; } };
    l = u.createElement(r);
    y = u.getElementsByTagName(r)[0];
    l.async = 1;
    l.src = "https://surfly.com/surfly.js";
    y.parentNode.insertBefore(l, y);
})(window, document, "script", "Surfly");

Surfly.init({ widget_key: "YOUR_WIDGET_KEY" }, function (initResult) {
    if (initResult.success) {
        if (!Surfly.isInsideSession) {

            // ✅ CXone loader goes HERE — inside Surfly's callback
            (function (n, u) {
                window.CXoneDfo = n;
                window[n] = window[n] || function () { (window[n].q = window[n].q || []).push(arguments); };
                window[n].u = u;
                var e = document.createElement("script");
                e.type = "module";
                e.src = u + "?" + Math.round(Date.now() / 1e3 / 3600);
                e.onload = function () {
                    // All cxone() calls go here, inside onload
                    cxone('init', 'YOUR_TENANT_ID');
                    cxone('guide', 'init');
                    // ... additional configuration
                };
                document.head.appendChild(e);
            })('cxone', 'https://web-modules-de-na1.niceincontact.com/loader/1/loader.js');

        }
    }
});
</script>
```

### Key Points
- Surfly **must** finish initializing before CXone loads.
- All `cxone()` calls must be inside the loader script's `onload` callback — the module needs to finish executing before commands can be issued.
- The `!Surfly.isInsideSession` guard prevents CXone from loading inside the Surfly session iframe, which would cause duplicate widget instances.

---

## Change 2 — CXone Chat Z-Index Fix for Surfly Overlay

### Problem
When a Surfly cobrowse session is active, the Surfly overlay renders on top of the CXone chat widget, making the chat window inaccessible to the customer.

### Solution
Add the following CSS to the page to force the CXone chat frame above the Surfly overlay layer.

### Implementation

```css
/* Raises the CXone chat widget above the Surfly cobrowse overlay */
.be-template {
    z-index: 2147483549 !important;
}
```

This can be added to the page's main stylesheet or in a `<style>` block in `<head>`. No JavaScript is required.

---

## Change 3 — Hide/Show Chat Text Input Based on Bot vs. Agent Assignment

### Problem
When a bot is handling the chat conversation, displaying the free-text reply box is misleading — the bot drives the interaction through quick replies and structured flows. The input area should only be visible when a live human agent is assigned.

### Solution
Use the `onAnyPushUpdate` push event listener to detect `CaseInboxAssigneeChanged` events and toggle the reply box visibility via `setCustomCss` depending on whether the new assignee is a bot or a human agent.

### Required API Calls

| Call | Purpose |
|------|---------|
| `cxone("chat", "setAllowedExternalMessageTypes", [...])` | **Required.** Enables the push update stream. Without this, `onAnyPushUpdate` will not fire. |
| `cxone("chat", "onAnyPushUpdate", callback)` | Subscribes to real-time case events, including assignee changes. |
| `cxone("chat", "setCustomCss", cssString)` | Applies or overrides widget CSS at runtime to hide or restore the reply box. |

### Implementation

```javascript
// Place inside the cxone loader's onload callback

cxone("chat", "setAllowedExternalMessageTypes", [
    "MESSAGE_SENT",
    "MESSAGE_RECEIVED",
]);

(function () {
    var BOT_USER_ID = 294069; // Replace with your bot's CXone user ID
    var isBotMode = false;

    function applyChatCss(botMode) {
        cxone("chat", "setCustomCss", botMode
            ? `
                [data-selector="REPLY_BOX"],
                [data-selector="TEXTAREA"],
                [data-selector="INPUT"],
                [data-selector="SEND_BUTTON"] {
                    display: none !important;
                }
              `
            : `
                [data-selector="REPLY_BOX"],
                [data-selector="TEXTAREA"],
                [data-selector="INPUT"],
                [data-selector="SEND_BUTTON"] {
                    display: revert !important;
                }
              `
        );
    }

    cxone("chat", "onAnyPushUpdate", function (payload) {
        if (payload?.eventType !== "CaseInboxAssigneeChanged") return;

        var assigneeUser = payload?.data?.case?.inboxAssigneeUser;
        var assigneeId   = payload?.data?.case?.inboxAssignee;

        var botMode =
            assigneeUser?.isBotUser === true ||
            assigneeUser?.id === BOT_USER_ID ||
            assigneeId === BOT_USER_ID;

        if (botMode === isBotMode) return; // No change, skip re-render
        isBotMode = botMode;
        applyChatCss(isBotMode);
    });
})();
```

### Key Points
- `BOT_USER_ID` must be updated to match the actual CXone user ID of the bot agent in your tenant.
- The check uses three conditions to identify a bot assignee — `isBotUser` flag, user `id`, or the `inboxAssignee` numeric ID — to be resilient against variations in the payload structure.
- The `isBotMode` flag prevents redundant `setCustomCss` calls if the same assignee type is set consecutively.
- `setAllowedExternalMessageTypes` must be called **before** `onAnyPushUpdate` or the push stream will not be active.

---

## Summary of Changes

| # | Change | Type | 
|---|--------|------|
| 1 | Nest CXone init inside `Surfly.init()` callback | JavaScript | 
| 2 | `.be-template` z-index override | CSS | 
| 3 | Bot/agent reply box toggle via `onAnyPushUpdate` | JavaScript | 
