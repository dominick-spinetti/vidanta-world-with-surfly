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

### Important: `setCustomCss` Is a Full Override

Every call to `cxone("chat", "setCustomCss", ...)` **completely replaces** all previously applied widget CSS. There is no merging or appending — the last call wins. This means if branding CSS is applied on load and then a separate call is made later to hide the reply box, the branding will be lost.

**To avoid this**, all widget CSS must live inside a single function that is called any time the styles need to change. The `botMode` parameter controls only the reply box portion, while all other styles (branding, etc.) are always included in every call.

### Required API Calls

| Call | Purpose |
|------|---------|
| `cxone("chat", "setAllowedExternalMessageTypes", [...])` | **Required.** Enables the push update stream. Without this, `onAnyPushUpdate` will not fire. |
| `cxone("chat", "onAnyPushUpdate", callback)` | Subscribes to real-time case events, including assignee changes. |
| `cxone("chat", "setCustomCss", cssString)` | Applies the complete widget CSS. Must always include all styles — branding and reply box — in a single call. |

### Finding the Bot's `BOT_USER_ID`

Bot user accounts are system-provisioned and **do not expose their `agentId` anywhere in the CXone UserHub UI** under a bot's own profile. However, the `agentId` shown in the **ACD → Users** list (when you search "bot") is the correct value to use — it matches the `agentId` field in the live `CaseInboxAssigneeChanged` payload.

> **Note on the two ID fields in the payload**
>
> The `CaseInboxAssigneeChanged` event contains two different numeric identifiers for the assignee:
> - `payload.data.inboxAssignee.agentId` — the **UserHub-visible** ID shown in ACD → Users. Use this one.
> - `payload.data.inboxAssignee.id` — an internal system ID that does **not** appear anywhere in the UserHub UI. Do not use this.

**To discover the `agentId` for a bot on any channel**, add `?cxdebug` to the page URL and use the built-in logging (see [Bot ID Discovery Logging](#bot-id-discovery-logging) below). Send a message in the chat and the bot's `agentId` will be logged to the console when the `CaseInboxAssigneeChanged` event fires.

### Implementation

```javascript
// Place inside the cxone loader's onload callback

cxone("chat", "setAllowedExternalMessageTypes", [
    "MESSAGE_SENT",
    "MESSAGE_RECEIVED",
]);

// Use the agentId value from ACD → Users in CXone UserHub (search "bot").
// This is the UserHub-visible ID. Do NOT use payload.data.inboxAssignee.id —
// that is an internal system ID not shown anywhere in the UI.
var BOT_USER_ID = 0; // ← Replace with your bot's agentId from UserHub
var isBotMode = false;

// ⚠️ This function is the single source of truth for ALL widget CSS.
// Always call this function instead of calling setCustomCss directly anywhere else.
// Adding a new style? Add it here — do not create a separate setCustomCss call.
function applyAllChatCss(botMode) {
    var replyBoxCss = botMode
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
          `;

    cxone("chat", "setCustomCss", `
        /* ── Branding ── */
        [data-selector="HEADER"] {
            background-image: url("YOUR_LOGO_URL") !important;
            background-repeat: no-repeat !important;
            background-size: 70%;
            height: 85px !important;
        }

        /* ── Reply box (bot vs. agent) ── */
        ${replyBoxCss}
    `);
}

// Apply full CSS on load with reply box visible (bot mode off)
setTimeout(() => applyAllChatCss(false), 500);

// Re-apply full CSS on every assignee change with updated bot state
cxone("chat", "onAnyPushUpdate", function (payload) {
    if (payload?.eventType !== "CaseInboxAssigneeChanged") return;

    var assigneeUser = payload?.data?.case?.inboxAssigneeUser;
    // agentId is the UserHub-visible ID (ACD → Users).
    // inboxAssignee.id is an internal system ID not shown in the UI.
    var agentId = payload?.data?.inboxAssignee?.agentId;

    var botMode =
        assigneeUser?.isBotUser === true || // primary: reliable flag in payload
        agentId === BOT_USER_ID;            // fallback: UserHub-visible agentId

    if (botMode === isBotMode) return; // No change, skip re-render
    isBotMode = botMode;
    applyAllChatCss(isBotMode);
});
```

### Key Points
- `BOT_USER_ID` must be set to the bot's `agentId` as shown in **ACD → Users** in CXone UserHub. See [Bot ID Discovery Logging](#bot-id-discovery-logging) to retrieve it from the live payload if needed.
- **All future CSS changes must go inside `applyAllChatCss`.** Never add a standalone `setCustomCss` call elsewhere — it will wipe out everything applied by this function.
- The `isBotUser: true` flag is the primary bot detection signal. The `agentId` match is a resilient fallback in case the flag is absent.
- The `isBotMode` flag prevents redundant `setCustomCss` calls if the same assignee type is set consecutively.
- `setAllowedExternalMessageTypes` must be called **before** `onAnyPushUpdate` or the push stream will not be active.

---

## Bot ID Discovery Logging

The page includes built-in debug logging to help identify bot `agentId` values for additional channels. It is **off by default** and activated by appending `?cxdebug` to the page URL — no code changes required.

### How to Use

1. Load the page with `?cxdebug` appended to the URL
2. Open DevTools → Console
3. Open the chat widget and **send a message** (tap a quick reply button or type)
4. When the bot is assigned, a `[CXone] CaseInboxAssigneeChanged` group appears in the console
5. Read the `agentId (UserHub-visible)` line — that is the value for `BOT_USER_ID`

### What Gets Logged

```
[CXone] CaseInboxAssigneeChanged
  agentId (UserHub-visible):   69381993   ← use this as BOT_USER_ID
  inboxAssigneeUser.isBotUser: true
  inboxAssigneeUser.firstName: Vidanta
  loginUsername:               vw.bot@...
  nickname:                    VidantaWorld
  ── Full payload ──
  { ... }
```

> The logging block is gated behind the `?cxdebug` flag and does not execute in normal page loads. It can be safely left in the production script.

### Known Bot `agentId` Values (Vidanta Tenant)

| Bot Name | UserHub agentId | Channel |
|----------|----------------|---------|
| Vidanta World | 69381993 | VidantaWorld Chat – DEV |
| Elegant Chat | 69479955 | — |
| MBE Bot | 69480059 | — |
| Ocean Breeze | 69479938 | — |
| Vidanta.com Bot | 69479956 | — |

> Channel assignments for all bots except Vidanta World are pending confirmation via `?cxdebug` testing on each respective demo page.

---

## Summary of Changes

| # | Change | Type | Required |
|---|--------|------|----------|
| 1 | Nest CXone init inside `Surfly.init()` callback | JavaScript | ✅ Yes |
| 2 | `.be-template` z-index override | CSS | ✅ Yes |
| 3 | Bot/agent reply box toggle via `onAnyPushUpdate` | JavaScript | ✅ Yes |
