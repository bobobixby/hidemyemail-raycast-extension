# PRD: Hide My Email – Cookie Bypass Fork

**Date:** December 29, 2025  
**Status:** Draft  
**Upstream:** `svenhofman/hidemyemail-raycast-extension` (to be forked)  
**Objective:** Restore functionality to the abandoned Raycast extension by replacing broken SRP authentication with a **Session Cookie Injection** mechanism.

---

## 1. Problem Statement

The current extension relies on `fast-srp-hap` to emulate Apple's Secure Remote Password (SRP) authentication protocol. As of late 2024, Apple updated their cryptographic handshake parameters, rendering the login flow non-functional (`-20101 Invalid credentials` or network errors). The original repository is unmaintained.

**Symptoms:**
- Users cannot log in to generate or list Hide My Email addresses.
- SRP handshake fails silently or throws credential errors.

---

## 2. Solution Overview

Switch from **Active Login** (User/Pass → SRP → Session) to **Passive Injection** (User provides valid Browser Cookie → Session).

This approach:
- Eliminates the need for cryptographic handshake logic
- Leverages long-lived sessions (~2 months when "Trust this browser" is selected)
- Allows the user to update credentials without touching code

---

## 3. Functional Requirements

### 3.1 Configuration (Preferences)
| Field | Type | Title | Description | Required |
|-------|------|-------|-------------|----------|
| `sessionCookie` | `password` | iCloud Session Cookie | Full `Cookie` header from icloud.com | `true` |
| `sortByCreationDate` | `checkbox` | Sort list by creation date | Sort list on most recently created address | `false` |

The `password` type ensures:
- Text is masked in UI
- Value is encrypted on disk
- Quick updates via `Cmd + ,` (Command Preferences)

### 3.2 Authentication Logic
- **Bypass Login Screen:** If `sessionCookie` preference is populated, skip `Login.tsx`, `LoginForm.tsx`, and `TwoFactorAuthForm.tsx` entirely.
- **Initialize Service:** Instantiate `iCloudService` immediately using the provided cookie.
- **Validation:** Optionally perform a lightweight `/validate` call to verify the session and fetch webservice URLs.

### 3.3 Network Layer
- **Header Injection:** Every HTTP request via `axios` must include:
  - `Cookie`: Derived from `preferences.sessionCookie`
  - `Origin`: `https://www.icloud.com`  
  - `Referer`: `https://www.icloud.com/`
- **Cookie Jar Override:** Manual header takes precedence over `tough-cookie` jar or LocalStorage.

### 3.4 Error Handling
- **Expiration Detection:** On `401 Unauthorized` or `450 Auth Required`:
  - Display Toast: *"Session Expired: Please update your Cookie in Preferences."*
  - Abort further requests gracefully

### 3.5 Core Features (Unchanged)
These must function identically to the original version:
- ✅ List existing "Hide My Email" addresses
- ✅ Generate/Reserve new addresses
- ✅ Deactivate/Reactivate addresses
- ✅ Update Label/Note metadata

---

## 4. Technical Implementation

### 4.1 `package.json` Updates
Add preference entry:
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

### 4.2 `src/api/connect.ts` Refactoring

```typescript
import { getPreferenceValues } from "@raycast/api";

export class iCloudSession {
  async init() {
    const prefs = getPreferenceValues<{ sessionCookie?: string }>();

    this.instance = wrapper(
      axios.create({
        jar: this.jar,
        timeout: 5000,
      }),
    );

    // COOKIE INJECTION
    if (prefs.sessionCookie) {
      this.instance.defaults.headers.common["Cookie"] = prefs.sessionCookie;
      this.instance.defaults.headers.common["Origin"] = "https://www.icloud.com";
      this.instance.defaults.headers.common["Referer"] = "https://www.icloud.com/";
    }
  }
}

export class iCloudService {
  // SRP logic removed
  async authenticate() {
    // Perform validate call to get webservices URLs
    const response = await this.session.request("post", `${iCloudService.SETUP_ENDPOINT}/validate`);
    this.data = response.data;
    this.webservices = this.data?.webservices;
  }
}
```

### 4.3 `src/components/Login.tsx` Bypass

```typescript
import { getPreferenceValues } from "@raycast/api";

export function Login({ onLogin }: { onLogin: (service: iCloudService) => void }) {
  useEffect(() => {
    if (!effectRan.current) {
      effectRan.current = true;
      
      const prefs = getPreferenceValues<{ sessionCookie?: string }>();
      
      // FAST TRACK – Cookie bypass
      if (prefs.sessionCookie) {
        const service = new iCloudService("cookie-session-user");
        service.init().then(() => {
          service.authenticate()
            .then(() => onLogin(service))
            .catch(() => showToast(Toast.Style.Failure, "Session Failed", "Check your cookie."));
        });
        return;
      }

      // ... fallback to old logic if needed
    }
  }, []);

  return <List isLoading={true} />;
}
```

### 4.4 Cleanup (Recommended)
Remove or comment out these unused modules:
- `src/components/forms/LoginForm.tsx`
- `src/components/forms/TwoFactorAuthForm.tsx`
- SRP methods: `srpAuthenticate()`, `validate2FACode()` in `connect.ts`
- (Optional) Remove `fast-srp-hap` from `package.json` dependencies

---

## 5. User Workflow

### 5.1 Initial Setup (Cookie Extraction)
1. Open **Chrome Incognito** or **Safari Private Window**
2. Navigate to `icloud.com` and log in
3. **Critical:** When prompted for 2FA, check ✅ **"Trust this browser"** (extends session to ~2 months)
4. Open Developer Tools → **Network Tab**
5. Filter for `validate` or `account` (any request to `setup.icloud.com`)
6. Click request → **Request Headers** → Copy the `Cookie:` value
7. In Raycast: Right-click extension → *Configure Extension* → Paste into "iCloud Session Cookie"

### 5.2 Cookie Refresh (When Session Expires)
When Toast shows *"Session Expired"*:
1. Repeat **Cookie Extraction** (Step 5.1)
2. Press `Cmd + ,` while extension is open → Command Preferences
3. Paste new cookie string
4. Press `Esc` and run command again

---

## 6. Implementation Checklist

- [ ] **Fork** `svenhofman/hidemyemail-raycast-extension` to personal GitHub
- [ ] **Update `package.json`**: Add `sessionCookie` preference
- [ ] **Refactor `src/api/connect.ts`**: Inject cookie into axios headers
- [ ] **Refactor `src/components/Login.tsx`**: Bypass login if cookie present
- [ ] **Cleanup**: Remove unused SRP logic and form components
- [ ] **Test**: List emails, create new address, handle 401 gracefully
- [ ] **Document**: Update README with new setup instructions
- [ ] **Import**: Install local extension into Raycast

---

## 7. Success Metrics

| Metric | Criteria |
|--------|----------|
| **Auth Bypass** | User can list emails without seeing login form |
| **Create Address** | User can successfully generate a new email |
| **Error Handling** | 401/450 errors display clear "Update Cookie" message |
| **Session Duration** | Cookie lasts ~2 months with "Trust this browser" |

---

## 8. Appendix: File Changes Summary

| File | Action | Description |
|------|--------|-------------|
| `package.json` | Modify | Add `sessionCookie` preference |
| `src/api/connect.ts` | Modify | Cookie injection, remove SRP |
| `src/components/Login.tsx` | Modify | Bypass logic |
| `src/components/forms/LoginForm.tsx` | Delete | No longer needed |
| `src/components/forms/TwoFactorAuthForm.tsx` | Delete | No longer needed |
| `README.md` | Modify | New setup instructions |
