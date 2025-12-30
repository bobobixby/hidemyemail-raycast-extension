Here is a Product Requirements Document (PRD) outlining the necessary changes to restore functionality to the **Hide My Email** Raycast extension.

---

# PRD: Hide My Email (Local Refactor)

**Date:** December 29, 2025
**Status:** Draft
**Objective:** Restore functionality to the extension by replacing the broken SRP authentication method with a Session Cookie injection method.

## 1. Problem Statement
The current extension relies on the `fast-srp-hap` library to emulate Apple’s Secure Remote Password (SRP) authentication protocol. As of late 2024, Apple updated their crypto parameters (salt/initialization), rendering the current login flow non-functional (`-20101 Invalid credentials` or network errors). The original repository is unmaintained.

## 2. Solution Overview
Instead of reverse-engineering Apple's new SRP protocol, we will implement a **"Bring Your Own Cookie" (BYOC)** model. The user will manually authenticate via a web browser, retrieve the session cookie, and store it in Raycast Preferences. The extension will use this cookie to authorize API requests, bypassing the native login form entirely.

## 3. Functional Requirements

### 3.1 Authentication
*   **Remove:** The extension shall no longer require the user to input Apple ID password or 2FA codes within Raycast.
*   **Add:** The extension must accept a raw HTTP `Cookie` string via the Raycast Preferences window.
*   **Logic:** Upon initialization, the extension must check for the presence of this cookie.
    *   If present: Bypass `Login.tsx` and `TwoFactorAuthForm.tsx` and immediately load the `iCloudService`.
    *   If absent: Display a prompt instructing the user to configure the extension in Settings.

### 3.2 Session Management
*   **Persistence:** The session validity relies entirely on the lifespan of the pasted cookie (approx. 2 months if "Trust this browser" was selected).
*   **Expiry Handling:** If an API request returns a `401` or `450` (Auth Required) error, the extension should display a Toast error: *"Session expired. Please update your cookie in Preferences."*

### 3.3 Core Features (Unchanged)
The following features must function identically to the original version, utilizing the new cookie for authorization:
*   List existing "Hide My Email" addresses.
*   Generate/Reserve new addresses.
*   Deactivate/Reactivate addresses.
*   Update Label/Note metadata.

## 4. Technical Specifications

### 4.1 `package.json` Updates
*   **New Preference Field:** Add a configuration field to securely store the cookie.
    *   **Name:** `sessionCookie`
    *   **Type:** `password` (Ensures text is masked and encrypted on disk).
    *   **Title:** "iCloud Session Cookie"
    *   **Description:** "Paste full Cookie header from icloud.com to bypass login."
    *   **Required:** `true`

### 4.2 `src/api/connect.ts` Refactor
*   **Disable SRP:** Comment out or remove logic related to `srpAuthenticate`, `validate2FACode`, and `SRP` imports.
*   **Cookie Injection:** Modify the `iCloudSession` class initialization.
    *   The `axios` instance must inject `preferences.sessionCookie` into the `Cookie` header for every request.
    *   *Note:* The `Referer` and `Origin` headers must remain set to `https://www.icloud.com` to pass CORS checks.

### 4.3 `src/components/Login.tsx` Refactor
*   **Bypass Logic:**
    *   Remove the state `AuthState.UNAUTHENTICATED`.
    *   On `useEffect`, check `getPreferenceValues<Preferences>().sessionCookie`.
    *   If valid, instantiate `iCloudService`, call a lightweight `init()` (no auth), and trigger `onLogin(service)`.

## 5. User Workflow (How to set up)

This new workflow will be documented in the `README.md` for the local user.

1.  **Browser Step:** Open an **Incognito/Private** window in your browser.
2.  **Login:** Go to `icloud.com` and log in.
3.  **Trust:** When prompted for 2FA, **check "Trust this browser"** (Critical for 2-month duration).
4.  **Extract:** Open Developer Tools $\to$ Network Tab $\to$ Refresh Page. Click the first request (`webservices` or `validate`). Copy the value of the **`Cookie:`** request header.
5.  **Configure:** In Raycast, right-click the extension $\to$ *Configure Extension* $\to$ Paste into "iCloud Session Cookie".

## 6. Implementation Plan

### Step 1: Clean Dependencies
*   (Optional) Remove `fast-srp-hap` from `package.json` to reduce bloat, as it is no longer used.

### Step 2: Update Manifest (`package.json`)
Add the preference entry:
```json
"preferences": [
  {
    "name": "sessionCookie",
    "title": "iCloud Session Cookie",
    "description": "Paste the full 'Cookie' header string from a logged-in icloud.com session.",
    "type": "password",
    "required": true,
    "placeholder": "X-APPLE-WEBAUTH-USER=..."
  },
  {
    "name": "sortByCreationDate",
    "title": "Sort list by creation date",
    "label": "Sort list on most recently created address",
    "description": "If enabled, the email list is sorted on most recently created.",
    "type": "checkbox",
    "required": false,
    "default": false
  }
]
```

### Step 3: Modify `src/api/connect.ts`
Simplify the `init` and `authenticate` flow.

```typescript
// Pseudo-code for modified logic
import { getPreferenceValues } from "@raycast/api";

export class iCloudSession {
  // ...
  async init() {
    const prefs = getPreferenceValues();
    this.instance = axios.create({
      headers: {
        "Cookie": prefs.sessionCookie,
        "Origin": "https://www.icloud.com",
        "Referer": "https://www.icloud.com/"
      }
    });
  }
}

export class iCloudService {
  // Remove SRP logic
  async authenticate() {
     // No-op. We assume the cookie is valid.
     // Optionally fetch account info to verify validity.
     this.webservices = { 
        // Hardcode the Premium Mail Settings URL if dynamic fetching fails, 
        // though usually, we can fetch this via a standard 'init' call if the cookie is good.
        "premiummailsettings": { "url": "https://p68-fmfmobile.icloud.com" } // Example, usually dynamic
     };
     
     // Better approach: Perform one "validate" call to get the webservices URLs
     const response = await this.session.request("post", `${iCloudService.SETUP_ENDPOINT}/validate`);
     this.data = response.data;
     this.webservices = this.data?.webservices;
  }
}
```

### Step 4: Modify `src/components/Login.tsx`
Strip the UI.

```typescript
export function Login({ onLogin }: { onLogin: (service: iCloudService) => void }) {
  useEffect(() => {
    async function init() {
      const service = new iCloudService("user"); // Username doesn't matter with cookie
      await service.init();
      try {
        await service.authenticate(); 
        onLogin(service);
      } catch (e) {
        showToast(Toast.Style.Failure, "Session Failed", "Check your cookie.");
      }
    }
    init();
  }, []);

  return <List isLoading={true} />;
}
```

## 7. Success Metrics
*   User can successfully "List Emails" without seeing a login form.
*   User can create a new email address.
*   Extension handles 401 errors gracefully.